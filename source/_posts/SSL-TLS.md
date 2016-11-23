---
title: SSL/TLS
date: 2016-11-23 14:12:57
tags: 技术
---

最近在折腾Microservice的gRPC鉴权问题，顺带梳理了下ssl/xls的原理。
以前没有特别深究过里面的问题，最近大概都算理清楚了，这里Mark也下。

# 1. ssl/tls
ssl/tls是一套基于非对称加密的安全协议，实际上ssl和tls是两个版本，大多数情况下这两个名词一起被提到，可以理解为tls是ssl的升级版本。

# 2. https
其实https是泛指http over secure，是在http协议之上增加的安全协议。当然，也有人会把它理解为http over ssl或者http over tls。
一般我们提到https都会默认提到ssl/tls，是因为ssl/tls在当前是最成熟的应用到https上。

# 3. ssl/tls握手
实际上https在安全机制上是做了两个事情，我们常常也会忽略。
- 加密传输
- 证书校验

握手的过程大概如下
- Client端发送ClientHello给Server端。包含协议版本、一个随机数(Random1)、支持的加密算法、压缩算法等。
- Server端发送ServerHello给Client端。包含协议版本的确认、一个随机数(Random2)、确认的加密算法、压缩算法等。
- Server端发送Certification(证书)给Client端。
- Server端发送ServerHelloDone给Client端。
- Client端发送ClientKeyExchange给Server端。包含随机数PreMasterSecret，并且PreMasterSecret要用证书的公钥进行加密。
- Client和Server端用Random1、Random2、PreMasterSecret算出来MasterSecret，这个MasterSecret就是双方约定的加密传输的对称秘钥，并且使用之前协商的加密算法，进行后续的数据交互。

# 4. ssl/tls加密传输
加密传输是https的最重要特性，保证了所有网络上的数据不会被明文探测。整个握手过程实际上是明文的，生成MasterSecret的前面两个随机数也是明文。实际最终保证安全性的关键在于，PreMasterSecret这个随机数的安全性。而这个随机数的安全性是建立在非对称加密的安全性上面。

# 5. ssl/tls证书校验
## 5.1 基本原理
证书签名的基本原理很简单，一般是利用非对称加密算法。非对秘钥一般成对出现（最常用的RSA），一个公钥，一个私钥。
公钥加密的信息，只有私钥能解开。同理，私钥加密的信息，只有公钥能解开。
例如：我用私钥加密一段内容，同时告诉对方我的公钥，对方如果可以用我的公钥解开我的内容，就可以证明这段信息是我的。这里面，被加密的内容就可以作为签名信息。

## 5.2 流程
ssl/tls的握手过程，往往容易忽略证书校验的过程，这个发生在握手的第3步，Server端发送证书给客户单之后，客户端是怎么验证证书的合法性呢？
握手的过程很简单，Server端的一组非对称秘钥，并没有参与到任何证书校验的环节，只是用来做了最后一个随机数PreMasterSecret的安全保障。

我一开始也有一些误解，实际上我们Server的一对秘钥，并不是用来做证书校验，证书校验是通过另外一组非对称秘钥来保证。
这里就要提到CA(Certification Authority)，CA是一个权威机构，主要负责证书的签名、发放、校验等职责。

![certification_flow](/images/tls_certification.png)

这个图比较清楚，证书校验的过程在第3步。
CA本身也有一套私钥、证书(证书里面一般包含了公钥信息)。
Server端的公钥证书是到CA加上签名，这个签名的过程，可能就是用CA的私钥加密的一段签名信息。
Client端拿到Server端的证书之后，同时Client端也有CA的证书，可以用CA根证书里面的CA公钥，对Server的证书进行签名校验。

所以，最终证书的安全性建立在Client端的CA根证书的安全性。（关于怎么保证Client端CA证书的安全性，这块后面再去了解下。）

## 5.3 为什么要给钱
正常的网站，为了做网站的证书签名，都需要找到权威的CA机构进行证书签名。
而CA对这块都是要付费的，那问题是，我们为什么非得找这些CA付费，自己随便搞一个假的CA进行对自己的证书进行签名不就行了吗。
大家可以参考万恶的[http://12306.cn](http://www.12306.cn)，就是那个订火车票的网站。例如用Chrome访问都会提示证书不安全，因为他们并没有找权威CA进行证书签名。
![https_error](/images/https_error.png)
所以，这就是一个行业规则问题，大多数的浏览器都会遵循认可这些官方的CA机构。所以，为了让浏览器能显示你是认证的网站 ，必须找权威的CA机构，给钱、签名。

# 6. Openssl自建CA、Server数字证书
恩，为什么要自建CA呢。
上面说了，ssl/tls实际上是做了两个事情，加密传输和证书验证。这两个事情，其实不太相关。
证书验证，大部分场景是在网站浏览器端，所以网站必须给权威的CA机构付费。因为，正常的浏览器都要遵循CA的游戏规则。
但是，如非浏览器应用，我们只是想利用xls/tls加密传输。我们就完全有理由自己搞私有的CA，自己给自己发证书了。

Openssl工具提供了一整套完整的工具，大概流程如下
## 6.1 配置CA
如果安装了Openssl，默认的路径在/etc/pki/。
配置文件路径在tls/openssl.cnf，基本的配置可以不用改。
可以简单关注一下policy_match相关的配置，这里配置了对不同证书信息是否进行校验。

- 进入CA目录
```
cd CA/
```
- 创建两个文件
```
touch index.txt serial
echo 01 > serial
```

- 创建根密钥
```
openssl genrsa -out private/ca_private_key.pem 2048
```

- 创建根证书
需要用到密钥 
```
openssl req -new -x509 -key private/ca_private_key.pem -out ca_cert.pem
```

## 6.2 生成Server证书

- 生成Server密钥
```
openssl genrsa -out server_private_key.pem 2048
```

- 生成Server证书
需要用到秘钥
```
openssl req -new -key server_private_key.pem -out server_csr.pem
...
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:GD
Locality Name (eg, city) []:SZ
Organization Name (eg, company) [Internet Widgits Pty Ltd]:COMPANY
Organizational Unit Name (eg, section) []:IT_SECTION
Common Name (e.g. server FQDN or YOUR name) []:your.domain.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
...
```

- CA对Server证书进行签名
把上一步的server_csr.pem发给CA机器，在CA机器上执行
```
openssl ca -in server_csr.pem -out server_cert.pem
```

## 6.3 完成
Server端拿到秘钥server_private_key.pem、证书server_cert.pem
Client端拿到CA提供的根证书ca_cert.pem
双方即可进行ssl/tls握手通信

# 7. ssl/tls双向校验
因为在做gRPC鉴权的时候，我们需要的Server端对Client进行身份鉴权，于是发现ssl/tls是支持双向鉴权。
简单来说就是，除了Client端对Server端进行证书鉴权，Server端也会要求对Client进行证书鉴权。

握手过程如下，加粗的步骤是新增的。
- Client端发送ClientHello给Server端。包含协议版本、一个随机数(Random1)、支持的加密算法、压缩算法等。
- Server端发送ServerHello给Client端。包含协议版本的确认、一个随机数(Random2)、确认的加密算法、压缩算法等。
- ****Server端发送Certification(证书)给Client端。****
- Server端发送Certification Request给Client端，要求Client端提供证书。
- Server端发送ServerHelloDone给Client端。
- ****Client端发送Certification给Server端。****
- Client端发送ClientKeyExchange给Server端。包含随机数PreMasterSecret，并且PreMasterSecret要用证书的公钥进行加密。
- Client和Server端用Random1、Random2、PreMasterSecret算出来MasterSecret，这个MasterSecret就是双方约定的加密传输的对称秘钥，并且使用之前协商的加密算法，进行后续的数据交互。

服务器拿到Client端的证书，进行类似Client端的证书校验过程，这样就实现了双向校验。
