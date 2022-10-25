---
title: 利用OpenSSL创建自签名证书
categories:
- 系统软件
date: 2018-07-10
tags: 
  - Node.js
  - OpenSSL
---

## 生成CA证书

1. 创建CA证书私钥

```bash
openssl genrsa -out ca-key.pem -des 1024
```

2. 通过CA私钥生成CSR

```bash
openssl req -new -key ca-key.pem -out ca-csr.pem
```

3. 通过私钥和CSR生成CA证书

```bash
openssl x509 -req -in ca-csr.pem -signkey ca-key.pem -out ca-cert.pem -days 1095
```

4. 检测CA证书是否正常

```bash
openssl x509 -in ca-cert.pem -noout -text
```



## 创建服务端或者客户端证书

1. 创建私钥

```bash
openssl genrsa -out server-key.pem 1024
```

2. 根据私钥生成CSR

```bash
openssl req -new -key server-key.pem  -out server-csr.pem
```



3. 创建extfile, 文件内容如下：

```ini
[v3_req]
    authorityKeyIdentifier=keyid,issuer
    basicConstraints=CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    subjectAltName = @alt_names
[alt_names]
	# 注意这个IP.1的设置，IP地址需要和你的服务器的监听地址一样
    DNS.1 = localhost
    DNS.2 = 127.0.0.1
    DNS.3 = m.asyncoder.com
```
> 注意DNS.3为你的域名信息，保持和步骤2里面的Common Name一致

3. 通过私钥和CSR生成自签名证书

```bash
openssl x509 -req -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -in server-csr.pem -out server-cert.pem -extensions v3_req -extfile extfile -days 1095
```

## 证书打包
一般服务器部署需要的SSL证书，我们直接用`server-cert.pem`和`server-key.pem`即可，但是在Node.js上为了开发方便，我们可以把证书进行合并。合并时，请记住输入的密码。

```bash
openssl pkcs12 -export -in server-cert.pem -inkey server-key.pem -certfile ca-cert.pem -out server.pfx
```

## 导入证书到操作系统

1. Windows上将`ca-cert.pem`改为`ca-cert.cer`, 点击导入到`受信任的根证书颁发机构`即可

2. Mac上直接安装`ca-cert.cer`即可

## Node.js测试

```javascript
const https = require('https')
const fs = require('fs')

const options = {
	pfx:fs.readFileSync('./server.pfx'),
	passphrase:'your password' // 生成server.pfx时需要的密码
};

https.createServer(options, (req, res) => {
	res.writeHead(200)
    res.end('hello world\n')
    
}).listen(443,'127.0.0.1')
```

用Chrome访问[https://localhost](https://localhost), 可以看到左上角绿色的锁，页面显示为`hello world`