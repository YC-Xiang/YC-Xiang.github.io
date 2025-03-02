# Chpater 2 应用层

## 2.1 应用层协议原理

### 2.1.1 网络应用程序体系结构

client-server 结构

P2P 结构

### 2.1.2 进程通信

主机由**IP 地址**标识.

**端口号**用于标识主机上的特定进程或服务.

## 2.2 Web 和 HTTP

### 2.2.1 HTTP 概况

超文本传输协议(HyperText Transfer Protocol), HTTP.

URL格式:

```txt
Protocol://user:psw@host:port/path
```

Protocol: 协议名.
user:psw: 登录名和密码.
host: 主机名.
port: 端口号.
path: 路径.

RTT: 往返时间, 指从客户机到服务器再返回客户机的总时间.

### 2.2.2 非持续连接和持续连接

非持续连接：每个请求/相应都是经过一个单独的 TCP 连接发送. 每次都要三次握手建立连接.

持续连接：所有的请求和相应都是经过相同的 TCP 连接发送.

### 2.2.3 HTTP 报文格式

HTTP 报文分为请求报文和响应报文.

**请求报文**

典型的 HTTP 请求报文:

```txt
GET /somedir/page.html HTTP/1.1
Host: www.someschool.edu
Connection: close
User-agent: Mozilla/5.0
Accept-language: fr
```

用普通的ASCII文本书写, 每行以回车和换行符结束.

第一行: 请求行(request line), 包括方法, 请求的URL, HTTP版本.

后续的行: 首部行(header line).

方法字段: GET, POST, HEAD, PUT, DELETE.

Host: 请求对象所在的主机.  
Connection: close 表示使用非持续连接.  
User-agent: Mozilla/5.0 表示用户使用的代理(浏览器).  
Accept-language: fr 表示接受法语版本.

</br>

通用的请求报文格式:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250301105204.png)

实体行(entity body) 在使用GET方法时为空, 在使用POST方法时, 实体行中包含用户输入的数据.

也可以不使用POST方法, 而使用GET方法, 并在请求的URL中包含数据, 比如 www.somesite.com/animalsearch? monkey&bannas

HEAD: 和GET方法类似, 但不返回请求对象.
PUT: 上传对象到服务器指定路径.
DELETE: 删除服务器对象.

**响应报文**

典型的 HTTP 响应报文:

```txt
HTTP/1.1 200 OK
Connection: close
Date: Tue, 18 Aug 2015 15:44:04 GMT
Server: Apache/2.2.3 (CentOS)
Last-Modified: Tue, 18 Aug 2015 15:44:04 GMT
Content-Length: 6821
Content-Type: text/html

(data data data)
```

Connection: close 表示使用非持续连接.  
Date: 服务器发送响应报文的日期和时间.  
Server: 服务器信息.  
Last-Modified: 对象创建或最后修改的日期和时间.  
Content-Length: 被发送对象的字节数.  
Content-Type: 被发送对象的类型.  

通用的响应报文格式:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250301104227.png)

状态码:

200 OK: 请求成功.  
301 Moved Permanently: 请求的对象已经被移到新的位置, 由 Location 首部给出新的位置.  
400 Bad Request: 通用的差错代码, 表示该请求不能被服务器理解.  
404 Not Found: 请求的资源未找到.  
505 HTTP Version Not Supported: 服务器不支持请求的HTTP版本.  

### 2.2.4 用户与服务器交互: cookie

服务器维护客户端状态.

### 2.2.5 Web缓存

Web缓存器(Web cache)也叫代理服务器(proxy server).

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250302174454.png)
