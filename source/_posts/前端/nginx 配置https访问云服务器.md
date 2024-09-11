---
title: nginx 配置https访问云服务器
date: 2023-07-12
author: falser101
tag:
  - nginx
category:
  - frontend
---

## 第一步
首先自己购买一个服务器
## 第二步
购买一个域名挂载到服务器，各个域名服务商有详细的文档这里就不再写了
## 第三步
申请ssl证书并且将证书和需要部署的静态页面上传到服务器如下图
![](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2021/03/28/kuangstudy6d0a31df-a8f5-467b-b518-24ac70e870f2.png)
[腾讯SSL证书域名验证指引](https://cloud.tencent.com/document/product/400/4142#ManualVerification)

# 编写default.conf
下面是我自己的证书和域名
```conf
server {
    #SSL 访问端口号为 443
    listen 443 ssl;
    #填写绑定证书的域名
    server_name falser.top;
    #证书文件名称
    ssl_certificate 1_falser.top_bundle.crt;
    #私钥文件名称
    ssl_certificate_key 2_falser.top.key;
    ssl_session_timeout 5m;
    #请按照以下协议配置
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    location / {
    #网站主页路径。此路径仅供参考，具体请您按照实际目录操作。
        root html;
        index  index.html index.htm;
    }
}

server {
	listen 80;
	#填写绑定证书的域名
	server_name falser.top;
	#把http的域名请求转成https
	return 301 https://$host$request_uri;
}

```
## 编写Dockerfile
```
# 设置基础镜像
FROM nginx
# 定义作者
MAINTAINER falser <1023535569@qq.com>
# 将dist文件中的内容复制到 /etc/nginx/html/ 这个目录下面
COPY dist/  /etc/nginx/html/
#用本地的 default.conf 配置来替换nginx镜像里的默认配置
COPY default.conf /etc/nginx/conf.d/default.conf
#将证书拷贝到docker镜像中的/etc/nginx/目录下
COPY 1_falser.top_bundle.crt /etc/nginx/
COPY 2_falser.top.key /etc/nginx/
```
## 创建镜像并启动
```shell
docker build -t falser .
```

```shell
docker run -d -p 443:443 -p 80:80 --name fblog falser
```
启动成功
![启动成功](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2021/03/28/kuangstudy610b21ad-2256-4b1c-b387-a47eb35df8e5.png)