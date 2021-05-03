---
title: Netgear R6850刷openwrt
date: 2021-05-03 17:31:31+0800
tags:
    - openwrt
---

五一假期不想出去看人山人海，宅在家中，闲来无事更新家里的Netgear R6850路由器的OpenWrt。

之前已经把路由器刷成OpenWrt固件了，但是当时官方尚未正式支持R6850，所以动手制作了一个snapshot镜像，并刷到机器上。

最近看到openwrt-21.02.0-rc1已经正式支持R6850了（虽然是rc版本......），so...搞起来。

# **1 制作固件**

## **1.1 环境搭建**

由于编译太耗时了，因此选择Image Builder来制作镜像。具体可参照 [官方文档](https://openwrt.org/docs/guide-user/additional-software/imagebuilder)。

为了简化环境搭建、方便后续更新，把Image builder的运行环境封装在Docker镜像中，在Docker中构建OpenWrt镜像

上次刷OpenWrt的时候已经封装了一些脚本，源码放在[lchannng/openwrt-builder](https://github.com/lchannng/openwrt-builder)。

```dockerfile
# openwrt-builder/Dockerfile

FROM debian:buster-slim

MAINTAINER lchannng <lchannng@gmail.com>

RUN apt-get update -qq &&\
    apt-get install -y \
        build-essential \
        curl \
        file \
        gawk \
        gettext \
        git \
        libncurses5-dev \
        libssl-dev \
        python2.7 \
        python3 \
        rsync \
        subversion \
        sudo \
        swig \
        unzip \
        wget \
        zlib1g-dev \
        && apt-get -y autoremove \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/*

RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
RUN useradd -c "OpenWrt Builder" -m -d /home/build -G sudo -s /bin/bash build

COPY --chown=build:build ./openwrt/ /home/build/openwrt/
RUN chown build:build /home/build/openwrt/

USER build
ENV HOME /home/build
WORKDIR /home/build/openwrt
```

```shell
#!/bin/bash
# File  : prepare.sh
# Author: lchannng <l.channng@gmail.com>
# Date  : 2020/08/18 19:24:13

TAG=ijk/openwrt-builder

OPENWRT_ARCH="${OPENWRT_ARCH:-ramips}"
OPENWRT_HOST="${OPENWRT_HOST:-Linux-x86_64}"
OPENWRT_TARGET="${OPENWRT_TARGET:-mt7621}"
OPENWRT_VERSION="${OPENWRT_VERSION:-19.07.3}"

BUILDER_NAME="openwrt-imagebuilder-${OPENWRT_VERSION}-${OPENWRT_ARCH}-${OPENWRT_TARGET}.${OPENWRT_HOST}"
BUILDER_URL="https://downloads.openwrt.org/releases/${OPENWRT_VERSION}/targets/${OPENWRT_ARCH}/${OPENWRT_TARGET}/${BUILDER_NAME}.tar.xz"

if [ ! -f ${BUILDER_NAME}.tar.xz ]; then
    echo "downloading ${BUILDER_URL}..." && \
    curl ${BUILDER_URL} -o ${BUILDER_NAME}.tar.xz && \
    tar xvf ${BUILDER_NAME}.tar.xz
fi

ln -snf ${BUILDER_NAME} openwrt
docker build -t ${TAG} .
```



这两个脚本做的事情很简单，仅仅是把我们需要的版本下载回来，并解压到 openwrt 目录，构建Docker镜像的时候，把 openwrt 目录拷贝到镜像的 /home/build/openwrt 目录。



## **1.2 构建 Docker 镜像**

把 [openwrt-builder](https://github.com/lchannng/openwrt-builder) 拉下来后，修改 build.sh 中填写的 OpenWrt 架构、版本等信息。

本次使用 openwrt-21.02.0-rc1，修改 OPENWRT_VERSION 如下：

```shell
OPENWRT_VERSION="${OPENWRT_VERSION:-21.02.0-rc1}"
```

修改完毕后，无脑执行 build.sh 构建 Docker 镜像，完成后可得镜像 ijk/openwrt-builder。

```shell
$ docker images

REPOSITORY            TAG       IMAGE ID       CREATED        SIZE
ijk/openwrt-builder   latest    2f82e3b0ca31   20 hours ago   919MB
```



## **1.3 制作 OpenWrt 镜像**

启动Image Builder

```shell
 $ docker run -it -v ~/openwrt-builder/bin:/home/build/openwrt/bin ijk/openwrt-builder
```

这里把宿主机的~/openwrt-builder/bin映射到Docker容器的/home/build/openwrt/bin目录，这个目录是 OpenWrt 镜像的输出目录。



构建镜像, PROFILE选择"netgear_R6850"，额外的包只增加luci。

```shell
$ cd /home/build/openwrt
$ make image PROFILE="netgear_R6850" PACKAGES="luci"
```



如无意外，等待几分钟后，在 /home/build/openwrt/bin/targets (宿主机下的 ~/openwrt-builder/bin/targets 目录) 中可以找到成功输出的镜像。

```shell
$ ls -R /home/build/openwrt/bin/targets/

/home/build/openwrt/bin/targets/:
ramips

/home/build/openwrt/bin/targets/ramips:
mt7621

/home/build/openwrt/bin/targets/ramips/mt7621:
openwrt-21.02.0-rc1-ramips-mt7621-netgear_r6850-squashfs-factory.img
openwrt-21.02.0-rc1-ramips-mt7621-netgear_r6850-squashfs-kernel.bin
openwrt-21.02.0-rc1-ramips-mt7621-netgear_r6850-squashfs-rootfs.bin
openwrt-21.02.0-rc1-ramips-mt7621-netgear_r6850-squashfs-sysupgrade.bin
openwrt-21.02.0-rc1-ramips-mt7621-netgear_r6850.manifest
profiles.json
sha256sums
```



# **2 刷机**

## 2.1 **从官方固件刷到 OpenWrt**

如果路由器还是 Netgear 官方固件，可以登陆管理页面，通过管理页面的更新固件入口，上传 OpenWrt 固件进行更新。比较无脑，不做详细描述。等待几分钟后，通过网线连接 lan 口尝试连接路由器并登陆 192.168.1.1 进行配置。

## 2.2 **从 OpenWrt 刷 OpenWrt**

如果路由器还能登陆到管理页面，可以在 System - Backup / Flash firmware 来进行刷机。

此外可以通过 [nmrpflash](https://github.com/jclehner/nmrpflash) 进行刷机。

> `nmrpflash` uses Netgear's [NMRP protocol](http://www.chubb.wattle.id.au/PeterChubb/nmrp.html) to flash a new firmware image to a compatible device. It has been successfully used on a Netgear EX2700, EX6100v2, EX6120, EX6150v2, DNG3700v2, R6100, R6220, R7000, D7000, WNR3500, R6080, R6400 and R6800, R8000, R8500, WNDR3800, but is likely to be compatible with many other Netgear devices.



具体操作如下：

1. 拔掉所有无关网线、无线网卡
2. 使用网线连接到路由器 lan 口
3. 将连接路由器的网卡 ip 设置为 192.168.1.2，子网掩码为255.255.255.0，默认网关 192.168.1.1 (网上大多数教程都是这么说的，暂时没看到官方资料有相关描述，但是不这么设置貌似刷不了机)
4. 关闭路由器电源
5. 运行 nmrpflash -i <interface> -f <openwrt-xxx.img>
6. 打开路由器电源，等待路由器 NMRP 握手并执行更新，如果一直连不上，重复 4 - 6 步骤



```shell
# 查看网络接口
$ nmrpflash -L

eth0  192.168.1.2    33:c6:47:28:70:95
```

```shell
# 运行nmrpflash，指定网口和OpenWrt镜像
$ nmrpflash -i eth0 -f openwrt-21.02.0-rc1-ramips-mt7621-netgear_r6850-squashfs-factory.img

Waiting for physical connection.
Advertising NMRP server on net14 ... /
Received configuration request from 78:d2:94:81:6a:cf.
Sending configuration: 10.164.183.252/24.
Received upload request without filename.
Uploading openwrt-21.02.0-rc1-ramips-mt7621-netgear_r6850-squashfs-factory.img ... OK
Waiting for remote to respond.
Received keep-alive request (2).
Remote finished. Closing connection.
Reboot your device now.

```

等待几分钟，直到出现”Reboot your device now.“ 后，重启路由器，并使用网线连接到路由器 lan 口，登陆 192.168.1.1 设置路由器。

## 2.3 翻车自救

翻车嘛，在所难免......

翻车不可怕，只要nmrp没坏，还是有救的，具体可参照 2.2 中 nmrpflash 刷机步骤进行重刷。

也可以通过 nmrpflash 重新刷回官方固件(辣鸡官方固件，应该没人会刷回去......)。



# 3 OpenWrt 后续配置

## **3.1 修改默认Lan IP**

openwrt 默认lan IP 是 192.168.1.1，与光猫冲突了，改成别的地址

ssh到路由器

```shell
$ uci set network.lan.ipaddr=192.168.111.1
$ uci commit && service network restart
```



## **3.2 SSH 密钥登陆**

参考官方文档 [Dropbear key-based authentication](https://openwrt.org/docs/guide-user/security/dropbear.public-key.auth)

root用户需要把公钥放到 /etc/dropbear/authorized_keys



## **3.3 privoxy 配置**

```config
# /etc/config/privoxy

config	privoxy	'privoxy'
	option	confdir		'/etc/privoxy'
	option	logdir		'/var/log'
	option	logfile		'privoxy.log'
        list listen_address	'192.168.111.1:8118'
        option  forward_socks5  '/ 192.168.111.1:65530 .'

```



## **3.4 出国留学**

使用自用的magic协议，这里不可描述，略过。

此外还要配置pac文件。

使用工具生产pac文件，代理地址为PROXY 192.168.111.1:8118，并把该文件放到路由器的/www/xxx.pac

在手机上把pac地址配置为 http://192.168.111.1/xxx.pac
