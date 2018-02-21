---
title: Linux（Centos 7.x）下安装nginx
date: 2017.01.23 23:03
tags: linux
categories: '后端'
---

#下载源码包安装nginx

>* 1  cd /usr/local/src
* 2  wget http://nginx.org/download/nginx-1.10.2.tar.gz
* 3  tar -xzvf nginx-1.10.2.tar.gz 
* 4  cd nginx-1.10.2
* 5  ./configure  --with-file-aio  --with-ipv6 --with-http_ssl_module  --with-http_stub_status_module --with-http_sub_module --with-http_realip_module --with-http_dav_module --with-http_gzip_static_module --with-mail   --with-mail_ssl_module   --with-debug  
* 6  make && make install

默认nginx命令在` /usr/local/nginx/sbin/nginx  `没添加到环境变量
可添加软连接` ln -s  /usr/local/nginx/sbin/nginx   /usr/local/bin/nginx `

#为nginx添加开机自启动服务

> * 1 nginx -c /usr/local/nginx/conf/nginx.conf   （启动nginx）
* 2 touch /lib/systemd/system/nginx.service (创建系统自启服务文件)
* 3 cd /lib/systemd/system
* 4 chmod 754 nginx.service (更改为754权限)
* 5 vi nginx.service(编辑服务文件)

***
（将以下内容写入文件内）
[Unit]
Description=nginx.service
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
***
> * 6 systemctl enable nginx.service
* 7 pkill -9 nginx （关闭nginx进程）
* 8 systemctl start nginx.service (启动nginx服务)

**直接访问ip，如果出现以下页面说明成功了!**
![](http://upload-images.jianshu.io/upload_images/4101261-387f8aaf318d7464.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)