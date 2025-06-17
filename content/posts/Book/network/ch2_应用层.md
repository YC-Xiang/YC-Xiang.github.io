# Chpater 2 应用层

## 2.1 应用层协议原理

### 2.1.1 网络应用程序体系结构

client-server 结构

P2P 结构

### 2.1.2 进程通信

主机由**IP 地址**标识。

**端口号**用于标识主机上的特定进程或服务。

## 2.2 Web 和 HTTP

### 2.2.1 HTTP 概况

超文本传输协议 (HyperText Transfer Protocol), HTTP.

URL 格式：

```txt
Protocol://user:psw@host:port/path
```

Protocol: 协议名.
user:psw: 登录名和密码.
host: 主机名.
port: 端口号.
path: 路径。

RTT: 往返时间，指从客户机到服务器再返回客户机的总时间。

### 2.2.2 非持续连接和持续连接

非持续连接：每个请求/相应都是经过一个单独的 TCP 连接发送。每次都要三次握手建立连接。

持续连接：所有的请求和相应都是经过相同的 TCP 连接发送。

### 2.2.3 HTTP 报文格式

HTTP 报文分为请求报文和响应报文。

**请求报文**

典型的 HTTP 请求报文：

```txt
GET /somedir/page.html HTTP/1.1
Host: www.someschool.edu
Connection: close
User-agent: Mozilla/5.0
Accept-language: fr
```

用普通的 ASCII 文本书写，每行以回车和换行符结束。

第一行：请求行 (request line), 包括方法，请求的 URL, HTTP 版本。

后续的行：首部行 (header line).

方法字段：GET, POST, HEAD, PUT, DELETE.

Host: 请求对象所在的主机。
Connection: close 表示使用非持续连接。
User-agent: Mozilla/5.0 表示用户使用的代理 (浏览器).  
Accept-language: fr 表示接受法语版本。

</br>

通用的请求报文格式：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250301105204.png)

实体行 (entity body) 在使用 GET 方法时为空，在使用 POST 方法时，实体行中包含用户输入的数据。

也可以不使用 POST 方法，而使用 GET 方法，并在请求的 URL 中包含数据，比如 www.somesite.com/animalsearch? monkey&bannas

HEAD: 和 GET 方法类似，但不返回请求对象.
PUT: 上传对象到服务器指定路径.
DELETE: 删除服务器对象。

**响应报文**

典型的 HTTP 响应报文：

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

Connection: close 表示使用非持续连接。
Date: 服务器发送响应报文的日期和时间。
Server: 服务器信息。
Last-Modified: 对象创建或最后修改的日期和时间。
Content-Length: 被发送对象的字节数。
Content-Type: 被发送对象的类型。

通用的响应报文格式：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250301104227.png)

状态码：

200 OK: 请求成功。
301 Moved Permanently: 请求的对象已经被移到新的位置，由 Location 首部给出新的位置。
400 Bad Request: 通用的差错代码，表示该请求不能被服务器理解。
404 Not Found: 请求的资源未找到。
505 HTTP Version Not Supported: 服务器不支持请求的 HTTP 版本。

### 2.2.4 用户与服务器交互：cookie

服务器维护客户端状态。

### 2.2.5 Web 缓存

Web 缓存器 (Web cache) 也叫代理服务器 (proxy server).

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250302174454.png)

## 2.3 因特网中的电子邮件

### 2.3.1 SMTP

## 2.4 DNS: 因特网的目录服务

### 2.4.1 DNS 提供的服务

DNS(Domain Name System): 域名系统，进行主机名到 IP 地址的转换服务。DNS 协议运行在 UDP 之上，使用 53 号端口。

除了进行主机名到 IP 地址的转换，还提供以下服务：

- 主机别名
  - 有着复杂主机名的主机能拥有一个或多个别名。
- 邮件服务器别名
  - 电子邮件应用程序可以调用 DNS，对提供的主机名别名进行解析，以获得该主机的规范主机名及其 IP 地址。
- 负载均衡
  - 一个 IP 地址集合与同一个主机名相关联。

### 2.4.2 DNS 工作原理

### 2.4.3 DNS 记录和报文

DNS 服务器存储了资源记录，资源记录是一个包含了下列字段的 4 元组：

(Name, Value, Type, TTL)

TLL 是该记录的生存时间，它决定了资源记录应当从缓存中删除的时间。Name 和 Value 的值取决于 Type：

- 如果 Type = A，则 Name 是主机名，Value 是该主机名对应的 IP 地址。
- 如果 Type = NS，则 Name 是个域 (如 foo. com),而 Value 是个知道如何获得该域
中主机 IP 地址的权威 DNS 服务器的主机名。
- 如果 Type = CNAME，则 Value 是别名为 Name 的主机对应的规范主机名。
- 如果 Type = MX，则 Value 是个别名为 Name 的邮件服务器的规范主机名

### 2.4.4 DNS 查询报文

DNS 查询报文和响应报文都是使用 UDP 的。

## 2.5 P2P 文件分发

BitToiTent 是一种用于文件分发的流行 P2P 协议。用 BitTorrent 的术语来讲，参与一个特定文件分发的所有对等方的集合被称为一个洪流（torrent）。

在一个洪流中的对等方彼此下载等长度的文件块（chunk），典型的块长度为 256KB。

每个洪流具有一个基础设施节点，称为追踪器（tracker）。

当一个对等方加入某洪流时，它向追踪器注册自己，并周期性地通知追踪器它仍在该洪流中。以这种方式，追踪器跟踪参与在洪流中的对等方。

## 2.6 视频流和内容分发网

## 2.7 套接字编程
