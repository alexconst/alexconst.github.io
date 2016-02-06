---
layout: post
title:  'Docker basics in X minutes'
author: 'Alexandre Constantino'
date:   2016-02-02
categories: devops
tags:       cheat-sheet virtualization docker
---


# Docker cheat sheet


## Installing Docker

```bash
# https://docs.docker.com/engine/installation/linux/debian/

# install Docker
apt-get purge lxc-docker*
apt-get purge docker.io*
apt-get update
apt-get install apt-transport-https ca-certificates
apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
rm -f /etc/apt/sources.list.d/docker.list
echo 'deb https://apt.dockerproject.org/repo debian-wheezy main' > /etc/apt/sources.list.d/docker.list
apt-get update
apt-cache policy docker-engine

# upgrade Docker
apt-get upgrade docker-engine
```


## Using Docker

Permission to dock
```bash
# either su to root
sudo bash

# or add user to docker group
usermod -aG docker $username
```


Check version
```bash
docker --version
# Docker version 1.10.0, build 590d5108
```


Litmus test with busybox
```bash
# download busybox image
docker pull busybox
# list available image templates
docker images
# run busybox image with an interactive terminal and remove container on exit
docker run -it --rm busybox

# on the busybox container:
ls -la
exit
```


