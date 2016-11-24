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
- Server端发送Certificate(证书)给Client端。
- Server端发送ServerHelloDone给Client端。
- Client端发送ClientKeyExchange给Server端。包含一个随机数PreMasterSecret，并且PreMasterSecret要用证书里面的公钥进行加密。
- Server端用自己的私钥把PreMasterSecret解开。Client和Server都用Random1、Random2、PreMasterSecret算出来MasterSecret，这个MasterSecret就是双方约定的加密传输的对称秘钥，并且使用之前协商的加密算法，进行后续的数据交互。

    注意：
    1. Server端所持有的私钥和公钥（在Certificate里面），并没有参与Certificate校验的过程，只是用来做最后一个随机数PreMasterSecret的加密保障。
    2. 这个过程并没有描述Certificate在Client端是怎么验证的，也就是Client根据什么确认Server的签名信息。

# 4. ssl/tls加密传输
加密传输是https的最重要特性，保证了所有网络上的数据不会被明文探测。整个握手过程实际上是明文的，生成MasterSecret的前面两个随机数也是明文。实际最终保证安全性的关键在于，PreMasterSecret这个随机数的安全性。而这个随机数的安全性是建立在非对称加密的安全性上面。

# 5. ssl/tls证书校验
## 5.1 基本原理
证书签名的基本原理很简单，一般是利用非对称加密算法。非对秘钥一般成对出现（最常用的RSA），一个公钥，一个私钥。
公钥加密的信息，只有私钥能解开。同理，私钥加密的信息，只有公钥能解开。
例如：我用私钥加密一段内容，同时告诉对方我的公钥，对方如果可以用我的公钥解开我的内容，就可以证明这段信息是我的。这里面，被加密的内容就可以作为签名信息。

## 5.2 关于CA/证书校验
ssl/tls的握手过程，往往容易忽略证书校验的过程，这个发生在握手的第3步，Server端发送证书给客户单之后，客户端是怎么验证证书的合法性呢？
握手的过程很简单，Server端的一组非对称秘钥，并没有参与到任何证书校验的环节，只是用来做了最后一个随机数PreMasterSecret的安全保障。

实际上我们Server的一对秘钥，并不是用来做证书校验，证书校验是通过另外一组非对称秘钥来保证。
这里就要提到***[CA(Certificate Authority)](https://en.wikipedia.org/wiki/Certificate_authority)***，CA是一个权威机构，主要负责证书的签名、发放、校验等职责。
主要逻辑参考这个图

![certification_flow](/images/tls_certification.png)

1. CA本身有自己的一套根密钥(cakey.pem)、根证书(cacert.pem一般包含了证书信息、对应的公钥信息等)
2. Server生成自己的密钥(server_key.pem)和证书签名请求CSR(server_csr.pem)
3. Server拿server_csr.pem到CA进行签名，得到server_crt.pem，即最终的证书。CA签名的过程，可以简单理解为用cakey.pem对server_csr.pem增加一段加密的签名信息。这段签名信息，只有CA的公钥（公钥信息在cacert.pem可以得到）可以解开。
```
F(cakey.pem, server_csr.pem) = server_crt.pem
```
4. Client端（一般是浏览器）会内置了CA的根证书(cacert.pem)。在ssl/tls握手的第3步，Client拿到Server的Certificate（这个其实就是server_crt.pem），就可以根据CA的根证书对Certificate进行验证。

所以，最终证书的安全性建立在Client端的CA根证书的安全性。

    Note: 怎么保证Client端CA证书的安全性？

## 5.3 为什么要给钱
正常的网站，为了做网站的证书签名，都需要找到权威的CA机构进行证书签名，而CA对这块都是要付费的。
那问题是，我们为什么非得找这些CA付费，自己随便搞一个假的CA进行对自己的证书进行签名不就行了吗。
其实也有人这样干，大家可以参考万恶的[http://12306.cn](http://www.12306.cn)。对，就是那个全世界最大的电商网站。
例如用Chrome访问都会提示证书不安全，因为他们并没有找权威CA进行证书签名。
![https_error](/images/https_error.png)
CA的权威性，是建立在行业标准之上的存在，所以对浏览器厂商也会遵循这些行业标准。也就是，这些浏览器都会内置权威的CA的根证书。为了让浏览器能显示你是认证的网站 ，必须找权威的CA机构，给钱、签名。

# 6. Openssl自建CA、Server数字证书
恩，为什么要自建CA呢。
上面说了，ssl/tls实际上是做了两个事情，加密传输和证书验证。
这两个事情，其实不太相关。
证书验证，大部分场景是在网站浏览器端，所以网站必须给权威的CA机构付费。
但是，如非浏览器应用，我们只是想利用xls/tls加密传输。
我们就完全有理由自己搞私有的CA，自己给自己发证书了。

Openssl工具提供了一整套完整的命令行工具，大概流程如下
主要做几件事情
1. 配置自己的CA
2. 生成Server的密钥，CSR
3. 用CSR到CA签名，生成CRT，也就是证书



    CSR: Certificate Signing Request，证书签名请求
    CRT: Certificate，证书


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
openssl genrsa -out private/cakey.pem 2048
```

- 创建根证书
需要用到密钥 
```
openssl req -new -x509 -key private/cakey.pem -out cacert.pem
```

## 6.2 配置Server

- 生成Server密钥
```
openssl genrsa -out server_key.pem 2048
```

- 生成Server CSR
需要用到密钥
```
openssl req -new -key server_key.pem -out server_csr.pem
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

## 6.3 证书签名
把上一步的server_csr.pem发给CA机器，在CA机器上执行
```
openssl ca -in server_csr.pem -out server_crt.pem
```

## 6.4 完成
Server端持有: 密钥server_key.pem、证书server_crt.pem
Client端持有: CA的根证书cacert.pem

双方即可进行ssl/tls握手通信

# 7. ssl/tls双向校验
因为在做gRPC鉴权的时候，我们需要的Server端对Client进行身份鉴权，于是发现ssl/tls是支持双向鉴权。
简单来说就是，除了Client端对Server端进行证书鉴权，Server端也会要求对Client进行证书鉴权。

握手过程如下，加粗的步骤是新增的。
- Client端发送ClientHello给Server端。包含协议版本、一个随机数(Random1)、支持的加密算法、压缩算法等。
- Server端发送ServerHello给Client端。包含协议版本的确认、一个随机数(Random2)、确认的加密算法、压缩算法等。
- Server端发送Certificate(证书)给Client端。
- ****Server端发送Certificate Request给Client端，要求Client端提供证书。****
- Server端发送ServerHelloDone给Client端。
- ****Client端发送Certificate给Server端。****
- Client端发送ClientKeyExchange给Server端。包含随机数PreMasterSecret，并且PreMasterSecret要用证书的公钥进行加密。
- Client和Server端用Random1、Random2、PreMasterSecret算出来MasterSecret，这个MasterSecret就是双方约定的加密传输的对称秘钥，并且使用之前协商的加密算法，进行后续的数据交互。

服务器拿到Client端的Certificate，进行类似Client端对Server端的Certificate校验过程，这样就实现了双向校验。
