---
layout: post
title: 'Wordpress基础搭建'
tags: [blog]
---

# 前记
作为一个Almost纯后端开发选手，对前端知识了解比较匮乏，看见一些个人站主的站点感觉非常有意思，于是好奇扒一扒其背后的技术实现，除少部分自己定制的以外，还有很大一部分站长采用的wordpress，进一步熟悉了解，知道了wordpress丰富的模块组件和插件生态，于是准备照猫画虎，来折腾一波。

# 准备
网上简单搜了下，关于wordpress建站的文章可以说是多如牛毛，原生技术组件一步步搭建的特别多。但是我又不想一步一步安装LAMP，麻烦不说，后续如果涉及到迁移还得再来一遍（脚本化搞也不是不可以）。于是想着用容器化的方式来实现。

# 概念
Docker & Docker-compose
1、Docker：Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的镜像中，然后发布到任何流行的 Linux或Windows 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口
2、Docker-compose：Docker-Compose 是 Docker 的一种编排服务，是一个用于在 Docker 上定义并运行复杂应用的工具，可以让用户在集群中部署分布式应用。

服务 & 项目：
1、服务 (service)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
2、项目 (project)：由一组关联的应用容器组成的一个完整业务单元，在 docker-compose.yml 文件中定义。

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
cd /home
mkdir wordpress
cd wordpress
vi docker-compose.yml
```
按需填写配置信息
```
version: '3.3'
services:
   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress
   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "80:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
       WORDPRESS_DB_NAME: wordpress
volumes:
    db_data: {}
```

## 启动项目
```
docker-compose -f docker-compose.wordpress.yml up -d #后台运行
docker-compose -f docker-compose.wordpress.yml down #停止并删除服务
```

## 访问测试
浏览器访问地址：`IP`（因为上面端口映射的是80端口，因此这里直接访问IP即可）

## 修改后台默认配置
访问后台页面地址：`IP/wp-admin`

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