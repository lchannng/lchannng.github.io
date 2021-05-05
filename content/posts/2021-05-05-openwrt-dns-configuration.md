---
title: openwrt dns 配置
date: 2021-05-05 18:08:44+0800
tags:
    - openwrt
---

## 安装依赖

自带的dnsmasq 不支持 ipset，需要安装 dnsmasq-full 才能支持所有功能，安装前先删除原来的 dnsmasq

```shell
$ opkg install dnsmasq-full ipset
```



## 配置dnsmasq

创建 dnsmasq.d 目录

```shell
$ mkdir /etc/dnsmasq.d
```



在 /etc/dnsmasq.conf 末尾增加 conf-dir

```
...

# include /etc/dnsmasq.d
conf-dir=/etc/dnsmasq.d/,*.conf
```



## 配置 DNS 服务器

新建文件/etc/dnsmasq.d/dns.conf

```
# ignore /etc/resolv.conf
no-resolv

server=119.29.29.29
server=223.5.5.5
```



## ipset 配置

dnsmasq 在 2.66 版之后加入了对 ipset 的支持，可将指定域名的 IP 解析后自动加入某一 ipset 中，可以结合魔法使用。

通过工具生成 dnsmasq 转换 ipset 规则：[传送门](https://github.com/lchannng/gfwlist2dnsmasq)

生成文件放到/etc/dnsmasq.d/xxx.conf

由于上面已经配置好了一个相对可靠的 dns 服务器，xxx.conf 中可以去掉针对特定域名的配置特定的 dns 服务器

```shell
# 仅保留 ipset 配置
$ cat /etc/dnsmasq.d/xxx.conf | grep ipset > /etc/dnsmasq.d/xxx.conf
```



/etc/rc.local 

```
...
# 启动后自动创建xxx集合
ipset create xxx hash:ip
```



## 最后

重启 dnsmasq

```
$ /etc/init.d/dnsmasq restart
```



测试域名解析

```shell
$ nslookup openwrt.org
```

