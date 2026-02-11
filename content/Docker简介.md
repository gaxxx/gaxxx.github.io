+++
title = "Introduction to Docker"
date = 2014-08-20
[taxonomies]
tags = ["cloud", "pass"]
+++

[Docker](http://docker.io) is a PAAS platform service software written in Go. It controls LXC services to provide hundreds or thousands of LXC containers on a single host, far more efficient than SAAS.

<!-- more -->

## Installation
On Archlinux, install directly with pacman:

```
root# pacman -S docker
root# systemctl start docker
```

On Ubuntu, it's renamed to docker.io due to naming conflicts:

```
root# apt-get install docker.io
root# start docker.io
```

## Common Commands

* Pull images - you can download various Linux distro images, from busybox to ubuntu:

```
//Download Ubuntu 14.04 minimal version
root# docker pull ubuntu:14.04
```

* List images - show pulled images:

```
root# docker images
    REPOSITORY          TAG                   IMAGE ID            CREATED             VIRTUAL SIZE
    busybox             latest                a9eb17255234        10 weeks ago        2.433 MB
    google/debian       wheezy                89f520140765        11 weeks ago        118.1 MB
    scratch             latest                511136ea3c5a        14 months ago       0 B
```

* Create containers - run programs from images like nginx, apache, or interactive shells like bash:

```
//Run interactive bash
root# docker run -i -t busybox /bin/sh

//Run a service: -d (detach) -P (export ports) -p (map container external port to internal port)
root# docker run -t ubuntu:14.04 -d -P -p 322:22 <service>
```

* Show containers:

```
//Active containers
root# docker ps
//All containers, including stopped ones
root# docker ps -a
```

* Commit an image - if you've installed software or updated the system in a container, commit it as an image to avoid repetitive work:

```
//Commit a container
root# docker commit fdafdafda  streamer:1.0
```

## Use Cases

* Version iteration testing - put each version in a fixed image, then create containers for testing
* Rapid deployment - create an image, then run it on various Linux versions that support Docker
* PAAS services - a single host can run thousands of small containers, e.g. nginx static file servers

## Dockerfile

Dockerfiles simplify Docker commands. For example, with a test directory containing streamer and node.conf that need to be deployed to Docker with port 9527, create a Dockerfile:

```
#docker pull ubuntu:14.04
FROM ubuntu:14.04

# Set correct environment variables.
ENV HOME /root
#Add files
ADD streamer /root/streamer
ADD node.conf /root/node.conf
#Export port
EXPOSE 9527
#Working directory
WORKDIR /root/
#Run program - without this line you can run all ubuntu:14.04 programs, but with it only streamer runs
ENTRYPOINT ["./streamer"]
```

Build the image from the Dockerfile:

```
root# docker build -t streamer  .
```

Run the image:

```
//Can only run streamer
root# docker run -t streamer -P  -p 9527:9527
```
