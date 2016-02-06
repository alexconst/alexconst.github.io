---
layout: post
title:  'Docker basics in X minutes'
author: 'Alexandre Constantino'
date:   2016-02-02
categories: devops
tags:       cheat-sheet virtualization docker
---


# Installing Docker

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

Check your permission to dock:
```bash
# either su to root
sudo bash

# or add user to docker group (and require a complete logout and login)
usermod -aG docker $username
```



# Using Docker


```bash
docker --version
# Docker version 1.10.0, build 590d5108
```


## Litmus test with busybox
```bash
# download busybox image
docker pull busybox
# list available image templates
docker images
# run busybox image with an interactive terminal and remove container on exit
docker run -it --rm busybox

## on the busybox container:
ls -la
exit
```

## Basics with ubuntu
```bash
# select distro and release
dimage="ubuntu"
dimagetag=":wily"

# search for an image
docker search $dimage
# list all tags for an image
# http://stackoverflow.com/questions/24481564/how-can-i-find-docker-image-with-specific-tag-in-docker-registry-in-docker-comma/32622147#32622147
# http://stackoverflow.com/questions/28320134/how-to-list-all-tags-for-a-docker-image-on-a-remote-registry
# NOTE: this is not working correctly because for debian the registry failed to show the "latest" tag
curl -s -S "https://registry.hub.docker.com/v2/repositories/library/$dimage/tags/" | jq '."results"[]["name"]' | sort
# or more verbose:
curl -s -S "https://registry.hub.docker.com/v2/repositories/library/$dimage/tags/" | jq '.' -C | less -r

# download an image (by default it will pull the one tagged with "latest")
docker pull "$dimage"
# check the distro release version for the latest tag (sometimes it's not really the latest!)
docker run -i -t --rm=true "${dimage}:latest" cat /etc/issue
# download an image for our chosen tag
docker pull "$dimage$dimagetag"
# check the distro release version
docker run -i -t --rm=true "${dimage}${dimagetag}" cat /etc/issue
# list images in our system
#   by default these are saved in /var/lib/docker
#   http://stackoverflow.com/questions/19234831/where-are-docker-images-stored-on-the-host-machine/25978888#25978888
docker images
# remove an image
docker rmi $some_image


# runs a command in a new container, it will also:
#   attach a terminal (to provide a console),
#   make it interactive to keep stdin open even if not attached,
#   name the container 'foo_meister'
#   spin the container from the ubuntu:wily image,
#   and the process executed will be /bin/bash
docker run -t -i --name="foo_meister" "${dimage}${dimagetag}" /bin/bash
## leave the container without terminating it (so any background processes executing will continue to do so)
# (you may have to type it really fast since ctrl+p is interpreted by the shell as the arrow key up)
ctrl+p+q
# list containers running (a user given or random name is given to each container)
docker ps
# get back on the container, where $container can be either the name or the id
# (you may have to press CR for the prompt to show up)
docker attach $container
## exit/suspend the container
exit

# list running and terminated containers
docker ps -a
# start a previously exited container
docker start $container
# get back to the container
docker attach $container
# leave the container
ctrl+p+q
# stop the container
docker stop $container
# remove a container (but keep any volumes)
# $container can be either the name or the id of a container
docker rm $container
```


## Sharing data between host and containers
```bash
# name the container
container="..."
# mount point on guest container
contmnt="/shared/"
# mount point on host machine
hostmnt="/home/someuser/documents/website/"
# run container with chosen data volume mapping
docker run -t -i --name=$container -v $hostmnt:$contmnt ubuntu /bin/bash
```

## Network port exposure/forwarding
```bash
# name the container
container="..."
# set guest container and host machine ports
contport="4000"
hostport="4000"
docker run -t -i --name=$container -p $hostport:$contport ubuntu /bin/bash
```

## Testing internet connectivity
```bash
# check network connectivity (the ip is from a google DNS machine)
# NOTE: this assumes a custom image which already has ping installed
docker run -i -t --rm=true "${dimage}${dimagetag}" ping -c4 8.8.8.8
docker run -i -t --rm=true "${dimage}${dimagetag}" ping -c4 google.com
```


## Creating an image from a container
```bash
# choose a name for our image
new_image_name="foo_bar"
# get the ID of an exited container
docker ps -a
container_id="..."
# commit the current version of the container
docker commit $container_id $new_image_name

# list images
docker images
# run a container from our image
container="container_from_new_image"
docker run -t -i --name=$container $new_image_name /bin/bash
```

## Locales

```bash
# switch the locale from POSIX to UTF-8
locale-gen en_US.UTF-8
dpkg-reconfigure locales
echo 'export LC_ALL="en_US.UTF-8"' >> $HOME/.bashrc
update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
```

<http://jaredmarkell.com/docker-and-locales/>


# Example: Jekyll container

Install Jekyll via the GitHub Pages gem.

NOTE: this is just a simple test and you actually don't want to use Jekyll on your system like this. Check my [Vagrant](vagrant.md) development environment [recipe](https://github.com/alexconst/vagrant_recipes/) instead.

Creating a container with GHP and Jekyll:
```bash
container="template"
docker run -t -i --name=$container ubuntu:wily /bin/bash

# on the container run:
apt-get update
apt-get install -y ruby-dev bundler zlib1g-dev ruby-execjs
# There is a bug which forces us to explicitly install activesupport, which would otherwise be done by the github-pages gem.
# https://github.com/github/pages-gem/issues/181
gem install activesupport
# choosing a specific version here because that was what I originally used,
# it may or may not work with latest github-pages release
gem install github-pages -v 43
# and because pygments.rb needs python pygments
apt-get install -y python2.7-dev python-pygments
# switch the locale from POSIX to UTF-8
locale-gen en_US.UTF-8
dpkg-reconfigure locales
echo 'export LC_ALL="en_US.UTF-8"' >> $HOME/.bashrc
update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

# before we test jekyll, refresh the locale
exit
docker start $container
docker attach $container
locale

# test jekyll
apt-get install -y wget
cd /tmp/
jekyll new blank_site
cd blank_site
jekyll build
jekyll serve &
wget -O - http://0.0.0.0:4000/ | less

# cleanup and exit
pkill jekyll
cd /tmp/
 rm -rf /tmp/*
apt-get -y autoremove
apt-get -y clean
exit
```




Creating an image from the container:
```bash
# check that the container is exited
docker ps -a
# create a new image
new_image="template_ghp_jekyll"
docker commit $container $new_image
# list images
docker images
```


Spinning a new container using the newly created image, with folder sharing and port forwarding:
```bash
contport="4000"
hostport="4000"
contmnt="/shared/"
hostmnt="/home/username/home/downloads/share_docker/"
docker run -t -i -v "${hostmnt}:${contmnt}" -p "${hostport}:${contport}"  $new_image  /bin/bash
```

To start jekyll and leave it running in the background:
```bash
cd /tmp/
jekyll new blank_site
cd blank_site
jekyll build
nohup jekyll serve &
ctrl+p+q
```



# Example: Jekyll Dockerfile

Dockerfile recipe for GitHub Pages Jekyll with a twist. It uses the `ENTRYPOINT` option to bind the container to an executable.

Create a dir for the Dockerfile:
```bash
mkdir -p docker_recipes/jekyll
cd !!:2
```

Edit the `Dockerfile` with the following:
```dockerfile
FROM ubuntu:wily

# install packages
RUN apt-get update && \
    apt-get install -y \
        ruby-dev \
        bundler \
        zlib1g-dev \
        ruby-execjs \
        --no-install-recommends && \
    apt-get install -y \
        python2.7-dev
        python-pygments
# install gems
RUN gem install activesupport
RUN gem install github-pages -v 43
# cleanup
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# fix locale
RUN locale-gen en_US.UTF-8
RUN dpkg-reconfigure locales
RUN echo 'export LC_ALL="en_US.UTF-8"' >> $HOME/.bashrc
RUN update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

# informs Docker that the container listens on the specified network ports at runtime
# and makes this port available to processes inside the container
EXPOSE 4000

# set workdir for any commands below
# with /shared/ being the guest volume dir when a container is spinned
WORKDIR /shared/blog/
# setting an entrypoint will set the container to run as an executable
ENTRYPOINT ["jekyll"]
# so everytime a container is created, it will cd to WORKDIR and run ENTRYPOINT
```


Or in case you already created an image and just want to test things:
```dockerfile
# replace the variable with the proper name
FROM $new_image

EXPOSE 4000
WORKDIR /shared/blog/
ENTRYPOINT ["jekyll"]
```





Build the image:
```bash
image_name_tag="alexconst/jekyll:v1"
docker build -t "${image_name_tag}" .
# check it
docker images
```


Spin a new container from our image:
```bash
contport="4000"
hostport="4000"
contmnt="/shared/"
hostmnt="/home/username/home/downloads/share_docker/"
# Before you spin an image, make sure you have a directory named `blog` in the host shared folder.
# NOTE: only run the `build` line if you have nothing there
# create a new site:
docker run -t -i --rm=true -v "${hostmnt}:${contmnt}" -p "${hostport}:${contport}"  "${image_name_tag}" new .
# build the site:
docker run -t -i --rm=true -v "${hostmnt}:${contmnt}" -p "${hostport}:${contport}"  "${image_name_tag}" build
# serve the site:
docker run -t -i --rm=true -v "${hostmnt}:${contmnt}" -p "${hostport}:${contport}"  "${image_name_tag}" serve
```



References:
<https://docs.docker.com/engine/userguide/containers/dockerimages/#building-an-image-from-a-dockerfile>
<https://docs.docker.com/engine/reference/builder/>
<http://stackoverflow.com/questions/28510982/dockerfile-understanding-volume-instruction>
<http://container42.com/2014/11/03/docker-indepth-volumes/>











# Troubleshooting

- docker not working
ERROR: Cannot connect to the Docker daemon. Is the docker daemon running on this host?
SOLUTION:
````bash
# either switch to root
su -
# or add the user to the docker group
usermod -aG docker $username    # and logout and login
/etc/init.d/docker restart
docker info
````

- docker not working
CONTEXT: /etc/init.d/docker start ; docker info
ERROR: Cannot connect to the Docker daemon. Is the docker daemon running on this host?
TROUBLESHOOTING: /var/log/docker.log
PROBLEM: Module nf_nat not found ... Error starting daemon: Error initializing network controller: error obtaining controller instance: Failed to create NAT chain: iptables failed: iptables -t nat -N DOCKER: iptables v1.4.14: can't initialize iptables table 'nat': Table does not exist (do you need to insmod?)
SOLUTION: enable CONFIG_NF_NAT_IPV4 in the kernel

- no network, unable to ping the gateway
PROBLEM: if your host firewall policy on the INPUT chain is set to drop then it's dropping packets
SOLUTION:
````bash
ifconfig | grep -A1 docker | sed 's|\.1 |\.0 |g'
network=`ifconfig | grep -A1 docker | grep "inet addr" | sed 's#\.1 #\.0 #g' | sed 's#.*addr:\([0-9\.]*\).*#\1#g'`
iptables -A INPUT  -s $network/24  -j ACCEPT
````

- no network, unable to ping google.com (or have network access)
PROBLEM: if your host firewall policy on the FORWARD chain is set to drop then it's dropping packets
SOLUTION:
````bash
docker_inet=`ifconfig | grep -A0 docker | awk '{print $1}'`
iptables -A FORWARD -i $docker_inet -j ACCEPT
iptables -A FORWARD -o $docker_inet -j ACCEPT
````

- failure to start a container when using port forwarding
ERROR: Error response from daemon: Cannot start container 5b47861d44fa5d48bcec0fc9ce965a53ea8d97a542e2f2780d3e772d17c07725: failed to create endpoint hungry_mestorf on network bridge: iptables failed: iptables -t filter -A DOCKER ! -i docker0 -o docker0 -p tcp -d 172.17.0.2 --dport 4000 -j ACCEPT: iptables: No chain/target/match by that name.
SOLUTION:
````bash
iptables -N DOCKER
````


**References:**
https://fralef.me/docker-and-iptables.html
http://blog.oddbit.com/2014/08/11/four-ways-to-connect-a-docker/
http://odino.org/cannot-connect-to-the-internet-from-your-docker-containers/
https://docs.docker.com/engine/installation/ubuntulinux/#enable-ufw-forwarding






