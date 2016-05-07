---
title: docker搭建nginx配置多域名以及多端口号
date: 2016-05-07 21:46:51
categories:
- nginx
- docker
tags:
- nginx
- docker
- 反向代理
meta:
- nginx
- 反向代理
---

最近为了搭建网站，采用nginx做反向代理，将自己的简单方法在此做一些相应的笔记。
## 1.需求：
 我的需求，需要将多个域名解析到同一台服务器上，并采用为服务的方式，将不同的域名和不同的端口号对应映射。
## 2.技术：
 由于目前的docker相对成熟，对于Linux中的镜像配置已变得很简单，所以我将采用docker进行搭建相应的配置，采用nginx服务器做反向代理（对于nginx在这里不多说，用的人都知道nginx的有点）。
 ## 3.实现：
 ### 第一步：
 安装docker，如果你的服务器上还没有相应的docker,可以参考CentOS安装步骤或者Ubuntu 系列安装 Docker步骤进行安装。
 ### 第二步：
 获取nginx镜像，执行命令：
``` bash
 $ docker pull nginx 
```
 等下载完成执行命令：
``` bash
 $ docker images
```
 即可看见你的服务器上已下载好对应的nginx镜像。
 ### 第三步：
 做配置的准备工作，先在自己的系统不目下建立一个文件夹，例如路径：/nginx/conf.d。
 ### 第四步：
 创建并启动nginx容器，执行命令：
``` bash
 sudo docker run –name=nginx -p 80:80 -v /nginx/conf.d:/etc/nginx/conf.d -d nginx
```
 此时你已经创建了一下名字为nginx的容器，该容器中/etc/nginx/conf.d目录下的文件将拷贝你本系统/nginx/conf.d目录下的文件，后面会讲到为什么要这样做。
 ### 第五步：
 到此你的nginx容器已经创建成功，我们在此修改对应的配置文件即可，例如我现在需要将www.aaa.abc.com的域名路径指向当前系统的运行端口号8080项目上，只需要进入你创建的目录/nginx/conf.d中增加一个文件，文件名字要求必须为.conf格式，例如可以改名为：www.aaa.abc.com.conf,里面的内容如下：
 ``` bash
 server {
    listen       80;
    server_name www.aaa.abc.com;
    location / {
       proxy_pass http://docker-Ip:8080;
      }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```
其中docker-Ip是你容器docker对应的ip,可以执行命令ifconfig看到对应的ip,如下：
{% img  /images/blog/docker-nginx/ifconfig-1.jpg 600 200  %}
可以看到ip为：192.168.42.1。将上面的docker-Ip改为这个即可。如果你需要多个域名的配置，只需要在/nginx/conf.d目录下加相应的配置文件即可，一般只需要修改server_name和proxy_pass即可。然后重启nginx容器，即：
``` bash
docker restart nginx 
```
打开你的域名看看是否已成功！


