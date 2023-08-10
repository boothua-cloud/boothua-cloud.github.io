---
title: Nginx代理Harbor
layout: post
tags:
  - Harobor
  - Docker
categories:
  [ Docker,Harobor]

index_img: /img/pages/R-C.png
---

### 场景描述：

> 有A,B,C三台服务器 A在互联网,B可以访问A,C在政务网,可以反问B,但是不能访问A
> 将Harbor装在A上,B作为代理服务器安装Nginx,C作为目标服务器,拉取镜像发布服务
<!-- more -->
### B服务器Nginx配置

```nginx
upstream harbor {
  server      100.64.0.2:18088;
}
server {
  listen       8096;
  server_name  localhost;

  location / {
    proxy_pass  http://harbor;
    client_max_body_size 0;
    proxy_connect_timeout 90;
    proxy_read_timeout 90;
    proxy_buffer_size 4k;
    proxy_buffers 6 32k;
    proxy_busy_buffers_size 64k;
    proxy_temp_file_write_size 64k;
  }
}
```

### C服务器配置

/etc/docker/daemon.json

```conf
{
  "insecure-registries": [
    "172.16.100.10:8096"
  ],
}
```

### C服务器Docker代理配置

C服务器需要配置docker代理,才能正常访问harbor,不然无法登录
/etc/systemd/system/docker.service.d/http-proxy.conf

```service
[Service]
# NO_PROXY is optional and can be removed if not needed
# Change proxy_url to your proxy IP or FQDN and proxy_port to your proxy port
# For Proxy server which require username and password authentication, just add the proper username and password to the URL. (see example below)

# Example without authentication
Environment="HTTP_PROXY=http://172.16.100.10:8096" "NO_PROXY=localhost,127.0.0.0/8"

# Example with authentication
Environment="HTTP_PROXY=http://username:password@172.16.100.10:8096" "NO_PROXY=localhost,127.0.0.0/8"
```

到此，C就可以通过B拉取A镜像仓库上的镜像来发布

### 拉取示例

```shell
docker pull 172.16.100.10:8096/library/nginx:latest
```