---
title: HTTPS自签名证书
abbrlink: f1f60ed2
tags:
  - Nginx
  - Https
categories:
  - Linux
index_img: /img/pages/https.jpeg
date: 2023-08-13 14:37:48
---
### 自签名证书
> 自签名证书在用于小范围测试等目的的时候，用户也可以自己生成数字证书，但没有任何可信赖的人签名，这种自签名证书通常不会被广泛信任，使用时可能会遇到电脑软件的安全警告

### 生成自签名证书
#### 生成
这里我是将自定义域名添加到了证书，你可以将IP等信息也添加进去
自定义域名配置host[/etc/hosts]
```shell
openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt -days 365 -nodes -subj "/CN=example.com" -addext "subjectAltName=DNS:example.com"
```
#### 信任
将公钥.crt证书复制到/usr/local/share/ca-certificates/目录下，有些系统在/etc/ssl/certs/目录下
```shell
sudo update-ca-certificates
```

### 总结
> 如果你生成的是域名证书，则需要将自定义域名添加到host
> 大部分linux操作系统都有update-ca-certificate，没有的话需要自行安装
> 浏览器访问需要将证书添加到信任列表

到此自签名证书就生成完成了