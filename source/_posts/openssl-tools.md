---
title: Openssl tools
date: 2016-11-24 17:34:05
tags: 
    - openssl 
    - tools
---

# 1. 生成rsa密钥文件

```
openssl genrsa -out private/cakey.pem
```

# 2. 查看密钥文件

```
openssl rsa -noout -text -in private/cakey.pem

```

# 3. 生成CSR

```
openssl req -new -key server_key.pem -out server_csr.pem
```

    依赖：密钥文件server_key.pem

# 4. 查看CSR信息

```
openssl req -noout -text -in server_csr.pem 

```

# 5. 生成证书

```
openssl req -new -x509 -key private/cakey.pem -out cacert.pem
```

依赖：密钥文件cakey.pem

# 6. 对CSR签名，生成证书

```
openssl ca -in server_csr.pem -out server_crt.pem
```

# 6. 查看证书

```
openssl x509 -noout -text -in server_crt.pem
```

