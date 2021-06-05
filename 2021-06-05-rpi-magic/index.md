# 树莓派折腾笔记：通过iptables + dnsmasq + magic开启魔法




喂！三点几啦，饮茶先啦...

接着上一篇，这次要在树莓派上部署一下魔法。

大体的思路是在本地开启一个透明代理，通过iptables把某些神秘地址转发到透明代理，由透明代理发到一个跳板机器上。

由于使用了透明代理，本地的应用不知道有代理的存在，会提前进行dns解析，神秘地址会被定向到一些不存在的ip上，要想拿到正确的ip，神秘地址的dns解析请求要通过安全的方式进行，比如tls、https，可以使用cloudflare的dns服务器。

![transparent_proxy](/images/2021-06-05-rpi-magic/transparent_proxy.png)

## 准备

上一篇已经把dnsmasq装上了，这次剩下需要用到的工具安排上。

```shell
$ apt install ipset netfilter-persistent iptables-persistent
```

[ipset](https://ipset.netfilter.org/)是[iptables](https://wiki.archlinux.org/title/Iptables) 的一个协助工具，用来维护特定的ip集合，iptables可以对这些集合进行屏蔽、转发。

netfilter-persistent、iptables-persistent主要用来持久化iptables规则和开机自动恢复。

## ipset 配置
### ipset作用

需要创建三个ipset：reserved、china、magic

reserved: 保留地址，需要跳过发到这些地址的流量

china：大局域网内部所有地址，需要跳过发到这些地址的流量

maigc：跳板机的地址，需要跳过发到这些地址的流量

### ipset配置文件
创建 /etc/ipset.d目录，并创建ipset配置文件：reserved.conf，magic.conf，china.conf

```
# /etc/ipset.d/reserved.conf

create reserved hash:net family inet hashsize 256 maxelem 1024
add reserved 0.0.0.0/8
add reserved 10.0.0.0/8
add reserved 100.64.0.0/10
add reserved 127.0.0.0/8
add reserved 169.254.0.0/16
add reserved 172.16.0.0/12
add reserved 192.168.0.0/16
add reserved 224.0.0.0/4
add reserved 233.252.0.0/24
add reserved 240.0.0.0/4
```

```
# /etc/ipset.d/china.conf

create magic hash:net family inet hashsize 128 maxelem 512

# 跳板机的地址
add magic xxx.xxx.xxx.xxx/32
add magic xxx.xxx.xxx.xxx/32
add magic xxx.xxx.xxx.xxx/32
```



通过**IP**deny提供的整个国家的ip列表生成china.conf 

先创建一个脚本 ~/cnipset.sh

```
#!/bin/bash -e
curl -O http://www.ipdeny.com/ipblocks/data/countries/cn.zone
ipset_file=china.conf
echo "create china hash:net family inet hashsize 4096 maxelem 65536" > ${ipset_file}
for i in $(cat ./cn.zone ); do
    echo "add china ${i}" >> ${ipset_file}
done
```

执行脚本生成china.conf，再把china.conf拷贝到/etc/ipset.d/china.conf

遍历/etc/ipset.d下的文件，并创建ipset

```shell
$ for f in `find /etc/ipset.d -type f`; do ipset restore -exist -file $f; done
```

### 开机自动恢复ipset
创建systemd服务， 重启后自动恢复ipset，注意ipset-persistent需要在netfilter-persistent之前启动

```
# /etc/systemd/system/ipset-persistent.service

[Unit]
Description=ipset persistent configuration
Before=network.target

# ipset sets should be loaded before iptables
# Because creating iptables rules with names of non-existent sets is not possible
Before=netfilter-persistent.service

# ConditionFileNotEmpty=/etc/ipset.conf
ConditionDirectoryNotEmpty=/etc/ipset.d

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c "for f in `find /etc/ipset.d -type f`; do ipset restore -exist -file $f; done"

[Install]
WantedBy=multi-user.target
RequiredBy=netfilter-persistent.service
```

启动服务
```shell
$ systemctl enable ipset-persistent
```



## iptables配置

修改/etc/iptables/rules.v4

```
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT

*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:MAGIC - [0:0]
-A OUTPUT -j MAGIC
-A PREROUTING -j MAGIC
-A POSTROUTING -s 172.17.0.0/16 ! -o brlan -j MASQUERADE

# 跳过跳板机的地址集合
-A MAGIC -m set --match-set magic dst -j RETURN

# 跳过保留地址集合
-A MAGIC -m set --match-set reserved dst -j RETURN

# 跳过大局域网地址集合
-A MAGIC -p tcp -m set --match-set china dst -j RETURN

# 其他tcp请求都转发到透明代理端口
-A MAGIC -p tcp -j REDIRECT --to-port 12345

COMMIT
```

重启netfilter-persistent

```shell
$ systemctl restart iptables
```




## 配置dnsmasq

本地先用工具启动一个dns代理工具，监听1053端口，把通过该端口的dns请求转发到1.1.1.1:853/tls上，去往1.1.1.1这个ip的流量会由iptables转发到透明代理上，实际上最终的dns请求会由跳板机完成。



基础配置 /etc/dnsmasq.d/dns.conf，屏蔽resolv.conf提供的dns服务器，上游dns服务器设置为腾讯云公共dns

```
no-resolv
all-servers
cache-size=4096
clear-on-reload
server=119.29.29.29
server=223.5.5.5
```



某[黑名单](https://github.com/gfwlist/gfwlist/raw/master/gfwlist.txt)的域名使用本地1053端口的dns接管，可过[工具](https://github.com/cokebar/gfwlist2dnsmasq)生成  /etc/dnsmasq.d/gfwlist.conf

```
server=/about.google/127.0.0.1#1053
server=/aboutgfw.com/127.0.0.1#1053
server=/abs.edu/127.0.0.1#1053
server=/ac.jiruan.net/127.0.0.1#1053
...
```



重启dnsmasq

```shell
$ systemctl restart dnsmasq
```



## 透明代理

本地透明代理和dns代理自行实现或使用某现成工具，不多BB，自求多福吧。
