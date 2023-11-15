---
layout: post
title: 'Wordpress基础搭建'
tags: [blog]
---

# 前记
作为一个几乎纯后端开发选手，对前端知识了解比较匮乏，看见一些个人站主的站点感觉非常有意思，于是好奇扒一扒其背后的技术实现，除少部分自己定制的以外，还有很大一部分站长采用的wordpress，进一步熟悉了解，知道了wordpress丰富的模块组件和插件生态，于是准备照猫画虎，来折腾一波。

# 准备
网上简单搜了下，关于wordpress建站的文章可以说是多如牛毛，原生技术组件一步步搭建的特别多。但是我又不想一步一步安装LAMP，麻烦不说，后续如果涉及到迁移还得再来一遍（脚本化搞也不是不可以）。于是想着用容器化的方式来实现。

# 概念
Docker & Docker-compose：

> - Docker：Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的镜像中，然后发布到任何流行的 Linux或Windows 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口
> - Docker-compose：Docker-Compose 是 Docker 的一种编排服务，是一个用于在 Docker 上定义并运行复杂应用的工具，可以让用户在集群中部署分布式应用。


服务 & 项目：

> - 服务 (service)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
> - 项目 (project)：由一组关联的应用容器组成的一个完整业务单元，在 docker-compose.yml 文件中定义。
Compose 的默认管理对象是项目，通过子命令对项目中的一组容器进行便捷地生命周期管理。

# WordPress搭建
## 前提
Linux环境，我这里使用的是`CentOS Linux release 7.9.2009 (Core)`

## 安装Docker
```
uname -r #查看你当前的内核版本
yum update #更新yum
yum -y install docker #安装 docker
systemctl start docker.service #启动 docker 服务
docker version #查看 docker版本
```

## 安装Docker-compose
```
yum install docker-compose #安装 docker-compose
docker-compose version #查看版本
```

## 安装WordPress
找个合适的目录，创建个目录
```
mkdir -p /home/docker/wordpress
cd /home/docker/wordpress
touch docker-compose.yml
vi docker-compose.yml
```
按需填写配置信息
```
version: '3.1'

services:

  nginx:
    image: nginx:stable-alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/conf/nginxconfig.io:/etc/nginx/nginxconfig.io:ro
      - ./nginx/conf/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/html:/usr/share/nginx/html:rw
      - ./nginx/log:/var/log/nginx:rw
      - ./nginx/ssl:/etc/ssl:ro
    networks:
      - default

  wordpress:
    image: wordpress
    restart: always
    # ports:
    #   - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: xxxx
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - ./flickerhub/html:/var/www/html
    networks:
      - default

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: xxxx
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: xxxx
    volumes:
      - ./flickerhub/db:/var/lib/mysql
    networks:
      - default

networks:
  default:
```
配置信息说明：
> 基于官方 docker-compose.yml 文件修改而来，主要修改了三个地方，一是为两个容器设置了网络，使其处于同一个子网，并与其他网络隔离开来，二是使用挂载主机目录的方式对容器数据进行持久化储存，方便以后管理和迁移，三是注释了 WordPress 容器的端口绑定，默认使用容器的 80 端口，后续在子网内可以直接使用容器名字访问 WordPress。

## Nginx基础配置
```
# 建立 nginx 配置文件夹
cd /home/docker/wordpress
mkdir -p nginx/conf

# 新建一个临时 nginx 容器
docker run --name nginx_temp -p 80:80 -d nginx:stable-alpine

# 将 nginx 默认配置拷贝到主机上的配置文件夹
docker container cp nginx_temp:/etc/nginx/conf.d ./nginx/conf/
docker container cp nginx_temp:/etc/nginx/nginx.conf ./nginx/conf/

# 停止删除临时 nginx 容器
docker container stop nginx_temp
docker container rm nginx_temp
(或者直接 docker container rm -f nginx_temp)
```
配置说明：

前文中的 docker-compose.yml 文件设置了三个容器，分别为 nginx，WordPress 和其使用的 Mysql。nginx 的作用是作为前端的反向代理服务器，用于响应用户的网络访问请求，并将对应的网络请求进行代理转发。当然我们本可以直接将 WordPress 容器的 80 端口暴露给外网，但是这样一来则无法在同一个 IP 地址下部署多个网站，也难以对网站进行 HTTPS 配置等操作，因此我们使用 nginx 来作为 Web 服务器，处理 Web 响应，进行代理和转发。

同时后续我们进行网站HTTPS化配置也会更便捷。


> 获取了 nginx 的默认配置以后，我们就可以启动容器了。
> - nginx 的基本配置位于 /home/docker/wordpress/nginx/nginx.conf
> - 网站配置文件位于 /home/docker/wordpress/nginx/conf.d 目录

## 启动项目
```
docker-compose -f docker-compose.yml up -d #后台运行
docker-compose -f docker-compose.yml down #停止并删除服务
```

## Nginx反向代理Wordpress并开启Https
**第一步：先生成证书**

由于我直接在阿里云上购买了免费的SSL证书，因此直接下载配置就好了，其他也有自己使用`Let's Encrypt`服务进行证书生成的，详细步骤可以看[这个教程](https://ethanblog.com/tips/install-wordpress-and-enable-ssl-with-docker.html)或者[这个教程](https://mofei.de/2022/5/)

**第二步：上传证书到指定目录**

把对应的`.pem`和`.key`文件放到之前的映射目录里，把对应的`.pem`和`.key`文件放到之前的映射目录里，把对应的`.pem`和`.key`文件放到之前的映射目录里。重要的事情说三遍。

**第三步：nginx配置**

```
cd /home/docker/wordpress/nginx/conf/conf.d
touch flickerhub.com.conf
vi flickerhub.com.conf

# 配置文件信息填写

server {
    listen 80;
    listen [::]:80;

    location / {
        return 301 https://flickerhub.com$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

   server_name .flickerhub.com;
    location / {
    proxy_pass http://wordpress;

        proxy_http_version    1.1;
        proxy_cache_bypass    $http_upgrade;

        proxy_set_header Upgrade            $http_upgrade;
        proxy_set_header Connection         "upgrade";
        proxy_set_header Host                $host;
        proxy_set_header X-Real-IP            $remote_addr;
        proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto    $scheme;
        proxy_set_header X-Forwarded-Host    $host;
        proxy_set_header X-Forwarded-Port    $server_port;
  }

    ssl_certificate /etc/ssl/flickerhubcom.pem;
    ssl_certificate_key /etc/ssl/flickerhubcom.key;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
    ssl_session_tickets off;

    # curl https://ssl-config.mozilla.org/ffdhe2048.txt > /path/to/dhparam
    ssl_dhparam /etc/ssl/dhparam.pem;

    # intermediate configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (63072000 seconds)
    add_header Strict-Transport-Security "max-age=63072000" always;
}
```
然后重启nginx容器

## 访问测试
浏览器输入域名测试是否OK

## 修改后台默认配置
访问后台页面地址：`domain/wp-admin`

修改语言：
```
1、先修改user-profile下的语言配置
2、再修改settings-general下的语言配置
```

修改时区，也是在settings-general下。

## 主题设置
还有很多其他设置，这里不赘述，对着后台页面进行摸索即可，这里着重说一下主题的设置。

后台主题市场的主题有很多可以选择，但是对国内的不是很友好，可以找一些国内的免费主题安装上，如果有些特殊的需求，也可以找国内的商用主题购买安装，主题选择方面后面有机会再开一篇文章推荐。这里说一个主题上传过程中的坑。

```
PS：我这里采用的Ri-Pro5 theme，感兴趣的朋友可以试试，请支持正版！
https://webphp.lanzoum.com/itft2148obsh
密码:`be56`
```

上传主题时会提示你：`The uploaded file exceeds the upload_max_filesize directive in php.ini.`

我们进到容器内部去找这个`php.ini`文件又找不到，这就很尴尬了。经过摸索，比较详细的修改配置步骤如下：

1、查看当前WordPress容器ID，命令为：docker ps 获取其容器CONTAINER ID为cdae0b8ff9d7
```
root@chillifish:~# docker ps
CONTAINER ID   IMAGE               COMMAND                  CREATED        STATUS                  PORTS                                NAMES
cdae0b8ff9d7   wordpress           "docker-entrypoint.s…"   37 hours ago   Up 18 hours             0.0.0.0:8082->80/tcp                WordPress
```

2、使用命令进入 wordpress 容器！
```
docker exec -it cdae0b8ff9d7 /bin/bash       //备注：cdae0b8ff9d7是wordpress容器ID
```

3、wordpress 容器中的这个路径/usr/local/etc/php/是存放 php.ini 的地方，但是默认是没有 php.ini 这个文件的，所以我们要通过复制一份php.ini-production文件，来生成 php.ini 文件。
```
cp /usr/local/etc/php/php.ini-production /usr/local/etc/php/php.ini
```

4、我们定位到 /usr/local/etc/php 文件夹下，如下命令：
```
cd /usr/local/etc/php
```

5、正常情况下，我们就可以使用Vim编辑器编辑php.ini文件了，但是不幸的是，官方的Wordpress容器中并没有预装vim编辑器。更新及安装vim，使用如下代码：
```
apt-get update
apt-get install vim
```

6、安装完成vim，现在就可以对php.ini进行编辑了
```
vi php.ini
# 一般情况下我们修改这几个变量，当然根据自己需求修改。
upload_max_filesize = 4096M    #文件大小限制
post_max_size = 4096M    #post大小限制
memory_limit = 500M        #内存占用限制
```

7、退出容器，进入主机，然后重启WordPress容器
```
root@chillifish:~# docker exec -it cdae0b8ff9d7 /bin/bash 
root@cdae0b8ff9d7:/var/www/html# cd /usr/local/etc/php
root@cdae0b8ff9d7:/usr/local/etc/php# vi php.ini
root@cdae0b8ff9d7:/usr/local/etc/php# exit # 退出容器
exit
root@chillifish:~# docker restart cdae0b8ff9d7 cdae0b8ff9d7 #回车重启WordPress容器
```

# 附录
包括但不限于wordpress的好看主题
```
hugo主题：https://github.com/joway/hugo-theme-yinyang
ripro主题：https://ritheme.com/
主题store：https://www.nicetheme.cn/
```

# 参考文章
```
 [1] https://zhuanlan.zhihu.com/p/93832797
[2] https://www.chillifish.cn/2944.html
```