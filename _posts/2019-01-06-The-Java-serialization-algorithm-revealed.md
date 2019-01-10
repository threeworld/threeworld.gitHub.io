---
layout: post
title:  "The Java serialization algorithm revealed"
date:   2019-01-06 20:15:33 +0700
categories: [Web安全]
---

## 0x00 序列化与反序列化

序列化是将一个结构或者对象保存为字节的存储数据的过程；反序列化是将这些字节数据重新构建一个活动对象的过程。

![Snipaste_2019-01-05_23-31-39]({{site.url}}/images/Snipaste_2019-01-05_23-31-39.png)

Java 序列化API给开发者提供一个标准的机制来处理序列化对象。但是如果想序列化一个对象，那么该对象必须实现 `java.io.Serializable` 接口。

## 0x01 为什么需要序列化

在开发中，一个典型的应用会由多个组件构成，也运行在不同的系统和网络中。在Java中，一切皆为对象，如果两个组件间需要通信，这个时候就需要一种机制来交换数据。一种方式是自定义协议来传输数据，这就意味着接受者必须知道两者之间定义的协议内容，对于第三方通信来说这样是非常困难的，而且是不方便的。因此，需要有一个通用的和有效的协议来在组件之间传输对象。序列化就是为此目的而定义的，Java组件使用此协议传输对象。

## 0x02 Java序列化机制

**一、实现反序列化接口**

如果想序列化一个对象，那么该对象必须实现 `java.io.Serializable` 接口。

```java
import java.io.Serializable;

class TestSerial implements Serializable {
    public byte version = 100;
    public byte count = 0;
}
```

在上面的代码中实现了`java.io.Serializable` 的接口，它是一个标记接口，没有任何方法，它只是告诉我们这个对象可以被序列化。

 

这个类已经可以被序列化，接下来就是实际序列化这个对象，Java提供了`writeObject()` 方法，该方法在`java.io.ObjectOutputStream` 类中，

**二、调用writeObject()**

```java
public static void main(String args[]) throws IOException{
    FileOutputStream fos = new FileOutputStream('temp.out');
    ObjectOutputStream oos = new ObjectOutputStream(fos);
    TestSerial ts = new TestSerial();
    oos.writeObejct(ts);
    oos.flush();
    oos.close();
}
```

上面代码将`TestSerial` 对象转化为字节数据存储在`temp.out`中， `oos.writeObject(ts)`实际上启用了反序列化算法，将对象写入到 `temp.out`中。



如果需要从文件中重新激活该对象，即反序列化对象，看下面代码：

```java 
public static void main(String args[]) throws IOException{
    FileIutputStream fis = new FileIutputStream('temp.out');
    ObjectInputStream oin = new ObjectInputStream(fis);
    TestSerial ts = (TestSerial)oin.readObejct();
    System.out.println("version="+ts.version);
}
```

通过调用`oin.readIbject()` 方法将字节数据激活对象。因为`readObject()`可以读取任何可序列化的对象，所以需要转换为正确的类型。然后输出`version=100`

## 0x03 反序列化对象的格式

那么序列化对象是怎么存储的呢？我们来看一下`TestSerial`的十六进制格式。

```hex
AC ED 00 05 73 72 00 0A 53 65 72 69 61 6C 54 65
73 74 A0 0C 34 00 FE B1 DD F9 02 00 02 42 00 05
63 6F 75 6E 74 42 00 07 76 65 72 73 69 6F 6E 78
70 00 64
```

`TestSerial` 的成员

```java
    public byte version = 100;
    public byte count = 0;
```

因为一个变量是一个字节，所以两者占了两个字节，但是你看到hex格式里面有51个字节，那么这些字节有什么含义呢，现在介绍一下Java的序列化算法，探讨一下这些字节是从哪里来的。

## 0x04 Java序列化算法过程

java反序列算法底层是怎么工作的呢？一般Java的算法都会做一下事情：

1. 它写出与实例关联的类的元数据。
2. 递归写出它继承的类的描述直到找到`Java.lang.object`为止
3. 一旦完成写入元数据信息，然后从与实例关联的实际数据开始。但它从最上面的超类开始。
4. 递归写出与实例相关的数据，从最小的超类到派生类

下面来看个例子:

```java
class parent implements Serializable {
	int parentVersion = 10;
}

class contain implements Serializable{
	int containVersion = 11;
}
public class SerialTest extends parent implements Serializable {
	int version = 66;
	contain con = new contain();

	public int getVersion() {
		return version;
	}
	public static void main(String args[]) throws IOException {
		FileOutputStream fos = new FileOutputStream("temp.out");
		ObjectOutputStream oos = new ObjectOutputStream(fos);
		SerialTest st = new SerialTest();
		oos.writeObject(st);
		oos.flush();
		oos.close();
	}
}
```

该例子还是比较清晰，`class SerialTest` 继承 `class parent`，`class SerialTest`中有 `class contain` 

运行后，查看序列化后的字节：

![Snipaste_2019-01-06_14-58-32]({{site.url}}/images/Snipaste_2019-01-06_14-58-32.png)

算法的具体大致流程：

![jtip050709-fig2-100156512-orig]({{site.url}}/images/jtip050709-fig2-100156512-orig.gif)

让我们详细查看对象的序列化格式，并查看每个字节代表什么，从协议的信息开始：

- AE ED：STREAM_MAGIC，表示这是序列化协议
- 00 05 :  STREAM_VERSION，表示序列化的版本
- 0x73 :  TC_OBEJCT，表示这是一个新的对象

算法的第一步是写入与类实例的描述，这个例子的对象的`SerialTest`，所以开始将`SerialTest`的描述写入文件。

- 0x72 : TC_CLASSDESC，表示这是一个新的类
- 00 0A : 类名的长度
- 53 65 72 69 61 6c 54 65 73 74 :  `SerialTest` ，类的名称
- 05 52 81 5A AC 66 02 F6 : `SerialVersionUID`，类的标识符
- 0x02 : 标记符，表示这个对象可以被序列化
- 00 02 :  类中的字段数目

然后再写入字段`int version = 66`

- 0x49 : 字段类型，49表示int
- 00 07 : 字段名的长度
- 76 65 72 73 69 6F 6E :  `version`，字段的名称

之后开始写入下一个字段，`contain con = new contain();` 这是一个对象，所以它将编写该字段的规范JVM签名。

- 0x74 :  `TC_STRING`，表示一个新的字符串
- 00 09 : 字符串的长度
- 4C 63 6F 6E 74 61 69 6E 3B :  `Lcontain;`, 规范的jvm签名
- 0x78 : `TC_ENDBLOCKDATA` 可选块，对象的结尾字段

下一步将写入父类`parent class`的对象描述，它是`SerialTest`的父类

- 0x72 : TC_CLASSDESC, 表示这是一个新的class
- 00 06 : 类名的长度
- 0E DB D2 BD 85 EE 63 7A: `SerialVersionUID`，类标识
- 0x02 : 标记符，表示这个对象可以被序列化
- 00 01 :  类中的字段数目

然后将写入字段`int parentVersion = 100;`

- 0x49 : 字段类型，49表示int
- 00 0D : 字段名的长度
- 70 61 72 65 6E 74 56 65 72 73 69 6F 6E : `parentVersion`，字段的名称
- 0x78 : `TC_ENDBLOCKDATA` 可选块，对象的结尾字段
- 0x70 : `TC_NULL`, 表示已经没有继承其他父类了，parent类实最顶层的类

到目前为止，序列化算法已经编写了与实例及其所有超类关联的类的描述。接下来，它将写入与实例关联的实际数据。它首先写入父类成员:

- 00 00 00 0A : 10,   `parentVersion`的值

然后写入`SerialTest`的成员值

- 00 00 00 42 : 66, version的值

接下来的几个字节很有趣。算法需要编写包含对象的信息。

**contain object**

```java
contain con = new contain();
```

通过上面可以知道，我们还没有将`contain`对象的描述写进去。

- 0x73 :  TC_OBEJCT，表示这是一个新的对象
- 0x72 : TC_CLASSDESC，表示这是一个新的类
- 00 07 : 类名的长度
- 63 6F 6E 74 61 69 6E : `contain`, 类的名称
- FC BB E6 0E FB CB 60 C7 : `SerialVersionUID`, 类的标识符

- 0x02 : 标记符，表示这个对象可以被序列化
- 00 01 :  类中的字段数目

接下来，写入该对象的字段`int containVersion = 11;`

- 0x49 : 字段类型，49表示int
- 00 0E : 字段名的长度
- 63 6F 6E 74 61 69 6E 56 65 72 73 69 6F 6E : `containVersion`, 字段的名称
- 0x78 :  `TC_ENDBLOCKDATA`

下一步，算法将会检测`contain`类是否还有父类，如果有，算法继续将其父类写入到文件中，如果没有就写入`TC_NULL`

- 0x70 : `TC_NULL`, 表示已经没有继承其他父类了，parent类实最顶层的类

最后写入`contain` 字段的值

- 00 00 00 0B : 11,  `containVersion`的值

## 0x05 总结

这文章中，已经了解如何序列化对象，并详细了解序列化算法的工作原理以及序列化的应用。

