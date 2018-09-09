---
title: Build AOSP with Docker
tags: build aosp docker dockerfile ubuntu sony xperia open device program
---

## Build AOSP with Docker

I was surprised to find very little on the subject.  
Some time ago I bought a device which is part of the [Sony Open Device
Program](https://developer.sony.com/develop/open-devices/#overview-content) to
free it from any Google services. So now I'm only using FOSS. Yeah!

I pulled 60 Gigabytes of Android sources but couldn't build it without errors.
This is where Docker comes in, it is a very good solution for such cases. I
couldn't find any up to date docker files except for [this
project](https://github.com/kylemanna/docker-aosp). The project is trying to do
everything itself for you, including downloading the source files. But I already had
that und just wanted to build them. So I made my own docker container.

[Here](https://gitlab.com/ploth/Dockerfiles/tree/master/aosp) can you find my
Dockerfile:

```Docker
FROM ubuntu:16.04

# reconfigure to use bash
RUN echo "dash dash/sh boolean false" | debconf-set-selections && \
    dpkg-reconfigure -p critical dash

RUN apt-get update && \
    apt-get install -y bc \
                       bison \
                       bsdmainutils \
                       build-essential \
                       curl \
                       flex \
                       g++-multilib \
                       gcc-multilib \
                       git \
                       make \
                       liblz4-tool \
                       gnupg \
                       gperf \
                       lib32ncurses5-dev \
                       lib32z1-dev \
                       libesd0-dev \
                       libncurses5-dev \
                       libsdl1.2-dev \
                       libwxgtk3.0-dev \
                       libxml2-utils \
                       lzop \
                       sudo \
                       openjdk-8-jdk \
                       pngcrush \
                       schedtool \
                       xsltproc \
                       zip \
                       zlib1g-dev \
                       graphviz \
                       python \
                       python3 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
RUN java -version
# fix ninja USER: unbound variable
ENV USER=root
```

Simply follow these steps:

```bash
docker build -t android - < /path/to/Dockerfile
docker run -it --mount type=bind,source=/path/to/android,target=/mnt android bash
# (inside container)
cd /mnt
source build/envsetup.sh && lunch
make –j <insert the cpu thread number of your computer>
```

This image is capable of compiling some android source branches. Tested with
`aosp_h3213-eng`, `aosp_h3213-userdebug`, `aosp_f5321-eng` and `aosp_f5321-userdebug`

| Branch 	        | Status        |
| :-                | :-            |
| android-8.1.0_r35 | Working       |
| android-9.0.0_r1 	| Not working   |
