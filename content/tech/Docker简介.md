+++
title = "Docker简介"
date = 2014-08-20
[taxonomies]
tags = ["cloud", "pass"]

+++

# Overview
[Docker](http://docker.io)是一个提供PAAS平台服务的软件，它由golang编写，通过控制lxc服务，在一台宿主机上提供成百上千个lxc container的服务器，效率远高于SAAS。

<!-- more -->

## 安装配置
在archlinux下，直接运行 pacman

```
root# pacman -S docker
root# systemctl start docker
```

在Ubuntu下，运行因为与其他应用重名，所以更名为docker.io

```
root# apt-get install docker.io
root# start docker.io
```

## 常用命令

* 下载镜像,可以下载其他linux发行版的镜像，小到busybox，大到ubuntu


```
//下载Ubuntu14.04 minimal版本
root# docker pull ubuntu:14.04  
```


* 列出镜像,列出pull下来的镜像


```
root# docker images
    REPOSITORY          TAG                   IMAGE ID            CREATED             VIRTUAL SIZE
    busybox             latest                a9eb17255234        10 weeks ago        2.433 MB
    google/debian       wheezy                89f520140765        11 weeks ago        118.1 MB
    scratch             latest                511136ea3c5a        14 months ago       0 B
```

* 创建Container, 其实就是执行镜像中的程序,比如nginx、apache等服务，也能执行bash等交互脚本


```
//执行交互式bash
root# docker run -i -t busybox /bin/sh

//执行服务 -d (detach) -P (导出端口) -p (将Contianer的外部端口映射到内部端口)
root# docker run -t ubuntu:14.04 -d -P -p 322:22 <service>
```

* 显示Containers


```
//活动的Container
root# docker ps 
//所有的Container,包括结束的
root# docker ps -a
```

* 提交image,如果在Container中安装了软件，或者更新系统，可以将Container提交成Image，避免重复的工作


```
//提交Container,需要
root# docker commit fdafdafda  streamer:1.0
```


## 使用场景

* 版本迭代测试，每个版本放一个不动的Image，然后通过建立Container进行测试
* 快速部署，创建一个image，然后运行在各种不同的linux版本上，只要支持docker
* PAAS服务，一台宿主机可以运行上千个小的Container,比如nginx静态文件服务

## Dockerfile

通过Dockerfile可以简化docker命令，比如有一个测试程序目录，里面有streamer , node.conf,需要将这两个文件部署到docker,并通过9527端口进行服务.可以在该目录新建一个Dockfile文件，内容如下

```
#docker pull ubuntu:14.04
FROM ubuntu:14.04

# Set correct environment variables.
ENV HOME /root
#增加文件
ADD streamer /root/streamer 
ADD node.conf /root/node.conf
#需要导出的端口
EXPOSE 9527
#工作目录
WORKDIR /root/
#运行程序，没有这一行的话，可以运行ubuntu:14.04的所有程序，但是有这一行，只能执行streamer了
ENTRYPOINT ["./streamer"]
```

根据Dockerfile编译Image 

```
root# docker build -t streamer  .
```

运行Image
	
```
//只能运行streamer
root# docker run -t streamer -P  -p 9527:9527
```
