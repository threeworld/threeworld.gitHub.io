---
layout: layout: default
category: Windows
tags: [Windows]
---

## 一.WFP介绍

> Windows过滤平台，是Vista系统后新增的一套API和服务，开发人员可以通过这套API集将Windows防火墙嵌入到开发软件中 。WFP为网络数据包过滤提供了架构支持，是微软在VISTA之后，替代之前的基于包过滤的防火墙设计，如Transport Driver Interface(TDI)过滤,Network Driver Interface Specification(NDIS)过滤，Winsock layered Service Providers(LSP)。 
>
> WFP框架被设计成分层结构，可以在不同层进行过滤，重定向，修改网络数据包

流程图：

![](https://www.andseclab.com/wp-content/uploads/2018/08/307940_z2zc7lt10p5qxhu.png)

**WFP框架：**

- 用户态基础过滤引擎，所需的API以及RPC接口都被封装在模块`fwpuclnt.dll`中 
- 内核态过滤引擎，无论是用户态过滤引擎，还是内核态过滤引擎，最终还是都是与内核态过滤引擎交互

**框架图：**

![](https://www.andseclab.com/wp-content/uploads/2018/08/15353829071-300x282.png)

内核态过滤引擎是整个WFP的主体，同时被分为很多层，不同的分层代表着网络协议栈特定的层，每个分层又可以存在子层和过滤器。当过滤引擎检查到网络数据包命中了过滤器的规则，就会执行过滤器中的指定动作。对应一般的应用，过滤器的动作只有两个：放行和拦截。但在真正的应用中，Filter Engine会存在很多层，很多过滤器，这些层和过滤器，可能会同时命中，这样就会存在多个过滤动作，对于这些过滤动作可能不相同，于是WFP引入了过滤仲裁器模块，这个模块将最终结果给Filter Engine，而Filter Engine将结果给垫片。 

## 二.基本对象模型

### 1.过滤引擎

WFP的核心组成部分，分为用户态基础过滤引擎和内核态过滤引擎，内核态过滤引擎是整个过滤引擎的主体，内部被划分为几个分层，不同分层代表着网络协议的特定层，在每个层都存在子层和过滤器。内核过滤引擎会检查网络数据包是否命中过滤器的规则，对于命中规则的过滤器，内核过滤引擎会执行指定的操作。

### 2.垫片

特殊的内核模块，被安插在系统网络协议栈的不同层中，主要作用是获取网络协议栈的数据。还有一个重要作用就是把内核过滤引擎的过滤结果反馈给网络协议栈，是网络协议栈与WFP的通信桥梁。 但是，垫片对于开发者来说是透明，无需关心垫片的结构，更多是在数据包的处理。

### 3.呼出接口（callout）

callout为一种数据结构，是对WFP功能的扩展，是由一系列回调函数组成，由过滤器指定。 对应callout结构体，在其中包含了一个GUID值，这个值是用来识别唯一的callout 。在callout中，还存在三个回调函数，对于这三个函数，只有一个是事前回调函数，其余两个是事后回调函数，一般只用事前回调函数，这些回调函数当被命中时会被调用。 

### 4.分层

过滤引擎由很多层组成。每一层代表了网络协议栈的一个层。分层是一个容器，里面包含了0或多个过滤器，或一个或多个子层。每个分层都有一个唯一标识的UID（内核中使用64位LUID,用户层使用128位GUID）。 

### 5.子层

子层由分层划分而来，是分层的一个更小单位，一个分层中包含了多个或零个子层，而子层之间的优先级由权重来区分，权重越大，优先级越高。 

### 6.过滤器

过滤器存在于WFP的分层中，过滤器实际上一套规则和动作的集合，规则指明了对那些网络数据包感兴趣，即指明需要过滤那些网络数据包。当过滤条件全部成立时，WFP就会执行过滤器指定的动作。开发者必须明确知道过滤器被添加到内核态过滤引擎的哪一个分层。

在同一分层内，不同的过滤器的权重不能相同，为了避免权重重复，可以为过滤器指定一个子层。

过滤器可以关联呼出接口，当过滤器的规则命中时，WFP就会执行过滤器关联的呼出接口内的回调函数。

## 三.呼出回调函数

**三个回调函数：**

- `classifyFn`
- `notifyFn`
- `flowDeleteFn`

### 1.classifyFn

当过滤器关联这个呼出接口，并且规则被命中，过滤引擎会回调这个函数。

### 2.notifyFn

当过滤器被添加到过滤引擎中或者从过滤引擎去除时，WFP会调用这个过滤器对应的呼出接口的`notifyFn`函数

### 3.flowDeleteFn

当一个网络数据流将要终止的时，WFP会调用该函数。但是是有条件的，只有在这个将要终止的数据流被关联了上下文的情况下，该函数才会被回调。

## 四.WFP操作

开发者在内核态使用WFP提供的API实现网络数据包过滤，一般步骤如下：

1. 定义一个或多个呼出接口，然后向过滤引擎注册呼出函数
2. 添加步骤1的呼出接口到内核过滤引擎。注册和添加时不同的操作。
3. 设计一个或多个子层，把子层添加到分层中。
4. 设计过滤器，把呼出函数，子层，分层和过滤器关联起来。向过滤引擎添加过滤器。

### 1.呼出接口的注册

```c
NTSTATUS NTAPI FwpCalloutRegister1(
    IN OUT void *deviceObject,
    IN const FWPS_CALLOUT1 *callout,
    OUT OPTIONAL UINT32 *calloutId
);
```

- deviceObject ： 呼出接口驱动所创建的设备对象指针。
- callout：呼出接口对象的指针
- calloutId：呼出接口的ID

### 2.呼出接口的添加

添加呼出接口时，需要过滤引擎的句柄，首先需要打开过滤引擎；

```c
NTSTATUS NTAPI FwpmEngineOpen0(
	In const wchar_t *serverName OPTIONAL,
	In UINT32 authnService,
	In SEC_WINNT_AUTH_IDENTITY_W *authIdentity OPTIONAL,
	IN const FWPM_SESSION0 *session OPTIONAL,
	OUT HANDLE *engineHandle
);
```

- servername：设置为NULL
- authnService：认证服务，设置成`RPC_C_AUTHN_WINNT`或者`RPC_C_AUTHN_DEFAULT`
- session：WFP的API是基于session的，表明相关的session信息，可以简单设置为NULL
- engineHandle：返回句柄，表示打开的过滤引擎句柄

通过这个句柄，可以将呼出接口添加到过滤引擎中

```c
DWORD WINAPI FwpmCalloutAdd0(
	__in HANDLE engineHandle,
    __in const FWPM_CALLOUT0* callout,
    __in_opt PSECURITY_DESCRIPTOR sd,
    __out_opt UINT32* id
)
```

- engineHandle：过滤引擎句柄
- callout：呼出接口指针
- sd：安全描述符，可以设置为NULL
- id：成功添加呼出接口后，系统会返回一个id.

### 3.子层的添加和移除

添加子层相对简单，开发者需要关心的是子层的权重以及子层的的GUID。添加子层：

```
DWORD WINAPI FwpmSubLayerAdd0(
	__in HANDLE engineHandle,
	__in const FWPM_SUBLAYER0* subLayer,
	__in_opt PSECURITY_DESCRIPTOR sd
);
```

- engineHandle：过滤引擎句柄
- subLayer：子层对象的指针
- sd：安全描述符，可以设置为NULL

可以使用`FwpmSubLayerDeleteByKey()`来移除子层

### 4.过滤器的添加和移除

添加过滤器相对复杂一些，主要原因是过滤器本身的定义复杂，需要为过滤器对象内的成员赋值，如权重，分层，子层，呼出接口，该函数如下：

```
DWORD WINAPI FwpmFilterAdd0(
	__in HANDLE engineHandle,
	__in const FWPM_FILTER0* filter,
	__in_opt PSECURITY_DESCRIPTOR sd,
	__out_opt UINT64* id
)
```

- engineHandle：过滤引擎的句柄
- filter：过滤器对象的指针
- sd：安全描述符，可以传递NULL
- id：当过滤器被成功添加到过滤引擎后，系统会返回一个id来标识它。

移除过滤器可以使用`FwpmFilterDeleteById0`或者`FwpmFilterDeleteByKey0`函数。