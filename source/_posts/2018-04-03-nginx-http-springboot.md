---
layout: post
title: "nginx做https转发，springboot应用内获取为http解决方案"
date: 2018-04-03 18:41:00 +0800
comments: true
tags: 
    - nginx
    - SpringBoot
keywords: nginx, SpringBoot
---
## 案列介绍
今天项目上线，使用nginx做负载均衡和反向代理，由于有应用内跳转的内容，发现跳转后的url均为http，导致应用无法正常访问。

体现在第三方授权的回调，以及应用内使用request.getRequestURL获取当前URL时，得到的结果均为http而不是期望的https。

<!--more-->

## nginx配置
nginx配置文件中在需要转发的部分加入以下内容。
```
proxy_set_header X-Forwarded-Proto  $scheme;
```

重启nginx。
```
sudo nginx -s reload
```
## SpringBoot配置
SpringBoot的application.properties中加入以下配置。
```
server.tomcat.protocol_header=x-forwarded-proto
server.tomcat.port-header=X-Forwarded-Port
server.use-forward-headers=true
```
SpringBoot参考文档地址。<https://docs.spring.io/spring-boot/docs/2.0.0.M3/reference/htmlsingle/#howto-use-tomcat-behind-a-proxy-server>


## 解决方法说明
在nginx端，需要将当前访问的真实scheme设置到request的头部。

在SpringBoot应用中，需要使用nginx设置的头部信息替换默认的信息。

同样的方式也可以做request.getRemoteIp等。

## 参考文档
<https://blog.csdn.net/yucharlie/article/details/77547754>
<http://www.cnblogs.com/yang-wu/p/5107899.html>