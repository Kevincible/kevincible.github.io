---
title: OpenWrt 透明代理的食用指南
date: 2019-04-04 18:10:16
tags:
categories: Tutorial
updated: 2019-04-23 22:24:45
typora-root-url: OpenWrt
---

![](logo.png)

本文主要介绍了如何在 Openwrt 环境下进行透明代理的搭建，在 Openwrt 所支持的 luci 图形化界面下进行配置大大简化了配置的过程。在配置完成后，所有接入该路由器的设备都可以科学访问被墙网站。

<!-- more -->

## 一、所需软件的安装

从原作者 [aa65535](https://github.com/aa65535) 的源 http://openwrt-dist.sourceforge.net/ 进行安装  
首先添加 Public-key

```bash
wget http://openwrt-dist.sourceforge.net/openwrt-dist.pub
opkg-key add openwrt-dist.pub
```

然后通过下面的命令获取处理器架构
```text
opkg print-architecture | awk '{print $2}'
```

然后向 `/etc/opkg/customfeeds.conf` 中添加
```text
src/gz openwrt_dist http://openwrt-dist.sourceforge.net/packages/base/{architecture}
src/gz openwrt_dist_luci http://openwrt-dist.sourceforge.net/packages/luci
```
-   例如我所使用的小米路由器 3G
    ```text
    root@OpenWrt:~# opkg print-architecture | awk '{print $2}'
    all
    noarch
    mipsel_24kc
    ```
    那么我向 `/etc/opkg/customfeeds.conf` 中添加
    ```text
    src/gz openwrt_dist http://openwrt-dist.sourceforge.net/packages/base/mipsel_24kc
    src/gz openwrt_dist_luci http://openwrt-dist.sourceforge.net/packages/luci
    ```

然后
```text
opkg update
opkg install ChinaDNS luci-app-chinadns
opkg install dns-forwarder luci-app-dns-forwarder
opkg install shadowsocks-libev luci-app-shadowsocks
```

---

## 二、Shadowsocks 配置

切换到 Services -> ShadowSocks

<img src="opmenu.jpg" style="width: 67%;" />

可以看到有三个服务可以选择

- Transparent Proxy
- SOCKS5 Proxy
- Port Forward

其中 Transparent Proxy 是我们这次的主角，其本质是通过 iptables 和 ipset 的规则以及 ss-redir 来实现对流量的转发。SOCKS5 Proxy 不做过多解释，最后的 Port Forward 可以用来进行转发 DNS 的查询。  
在配置完所需服务和服务器后切换到 Access Control 选项卡，也就是对进行访问控制的管理，本质上就是各种 iptables 和ipsets 的参数。

![](acctr.jpg)

其中 Bypass IP List 选择 ChinaDNS CHNRoute 也就是国内的路由表，这是 [APNIC](https://www.apnic.net/) 分配给中国的所有 IP 段，这样所有访问国内网站的流量就会直连而只有访问国外网站的流量才会被 ss-redir 转发，进而提升国内网站的访问效率。CHNRoute 是跟随[ChinaDNS](https://github.com/aa65535/openwrt-chinadns)一同安装的，但默认安装的 CHNRoute 版本较老，可以用以下命令进行更新

``` bash
$ wget -O- http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/chinadns_chnroute.txt
```

Zone LAN 中 Proxy Type 有 Direct Global Normal 三种，对应直连、全局代理、透明代理，其中 self proxy 指的是是否代理路由器自身产生的流量。

---

## 三、解决 DNS 污染问题

至此，ShadowSocks 的配置已经全部完成了，但是如果不解决 DNS 污染的问题，依旧会对日常的使用造成很大的影响。  
从原理上来讲，ChinaDNS 起到的作用只是一个优化路线的功能，尤其针对某些网站在国内和国外都有 CDN 的情况。ChinaDNS 会将 DNS 请求同时发给国内 DNS 和可信 DNS（通常为国外 DNS），如果国内 DNS 返回的结果在 CHNRoute 中，则采用，如果国内 DNS 返回的结果不在 CHNRoute 中，则丢弃其结果并转而采用可信 DNS 返回的结果，从而起到一个优化路线的作用。  

这样的方法有一个明显的缺陷，如果 GFW 放弃将域名污染到随机无效 IP 的方法而转而采用将域名污染到国内有效 IP 的情况下，ChinaDNS 就会失效。但就目前来看，ChinaDNS 的方案还是可行的。  

还有一点需要注意的是，在 ChinaDNS 中，可信 DNS 的优先级高于国内 DNS，也就是说如果可信 DNS 先于国内 DNS 返回结果，那么无论怎么样都会采用可信 DNS 的结果，因为这个特性，在 ChinaDNS 的上游是不应该有 DNS 缓存的，DNS 缓存应当设置在 ChinaDNS 下游。

### DNS-forwarder 的配置

从上文的描述中我们知道，ChinaDNS 上游的可信 DNS 是非常重要的，但是在国内的大多数 ISP 都有针对 53 端口的 DNS 查询都有投毒的情况，而且国内 ISP 的 UDP 不稳定，所以我选择用 DNS-forwarder 把 DNS 查询流量转换为 TCP 流量，并通过 ss-redir 进行转发到国外的 DNS 服务器，如果嫌麻烦的朋友也可以直接使用 ss 自带的 port forward 进行转发。

![](dns-forwarder.jpg)
监听端口 5353，我选择的 DNS 是 Cloudflare 的 1.1.1.1，设置好了后我们将 1.1.1.1 手动添加到 ShadowSocks 的 Access Control 中的forward list 中，保证 DNS-forwarder 的查询流量是经过 ss-redir 转发的。

### ChinaDNS 配置

接下来就是配置 ChinaDNS 了

![](chinadns.jpg)
其中有一个 Bidirection Filter 的选项，打开后如果国外 DNS 返回的结果在 CHNRoute 中，那么也不会被采用，通常情况下并不建议打开。

Upstream Servers 配置 ChinaDNS 的上游服务器，其中必须要包含至少一个国内 DNS 和一个国外 DNS。

顺带一提 ChinaDNS 判断是不是国内 DNS 是通过 IP 地址在不在 CHNRoute 中来判断的，所以如果想要用某些本地 ISP 提供的 DNS 作为国内 DNS 的话，一定要检查其 IP 地址是否在 CHNRoute 中。

这里国内 DNS 我用的是 114DNS，国外 DNS 经过 DNS-forwarder 转发，所以填上 DNS-forwarder 的监听地址和端口。

还有一点需要注意的是地址和端口号之间使用 "**#**" 分隔而不是 "**:**"

---

## 四、dnsmasq 配置

现在还剩下两个问题，一个是 DHCP 广播的 DNS 服务器，另一个是 OpenWrt 上本机的 DNS 服务器。

### DHCP 配置

我们可以通过配置 dnsmasq 来进行转发 DNS 请求并缓存 DNS 查询的结果。

切换到 Network -> DHCP and DNS

首先 DNS forwardings 填入 **127.0.0.1#5353**，也就是 Chinadns 的监听地址

![](dhcp-2.jpg)

接下来切换到 Resolv and Hosts Files，勾选 Ignore resolve file，否则 DHCP 会使用 WAN 口收到的公告的 DNS 服务器。

![](dhcp-1.jpg)

这样相当于我们所有的 DNS 查询请求都首先交给 dnsmasq 进行转发并缓存，而 dnsmasq 则进一步将这些请求转发给 Chinadns。

这里所有的配置均在 /etc/config/dhcp 中

至此所有连接到路由器的设备就应该可以正常使用了

### OpenWrt 本机的 DNS 服务器设置

现在就剩下路由器本身的 DNS 查询请求了

切换到 Network -> Interfaces，进入到 WAN 口的配置中，切换到 Advanced Settings 选项卡，取消勾选 **Use DNS Servers advertised by peer** 并在 **Use custom DNS servers** 中填入 127.0.0.1，这样的话所有路由器自身的DNS查询也会交给 dnsmasq。

![](wan-conf.jpg)

这样所有从OpenWrt上产生的DNS查询也会交由dnsmasq进行转发。

---

## 五、番外

### 1、自定义不代理的域名

在 dnsmasq 的 manpage 中我们可以看到一个参数 `--ipset=/<domain>[/<domain>...]/<ipset>[,<ipset>...]` 这个参数的含义是将指定域名的查询结果加入到指定的 ipset 中，我们可以利用这个参数将我们自定义不代理的域名的解析结果加入到 ss_spec_dst_bp 这个 ipset 中便可以实现强制直连。

#### 安装 dnsmasq-full

```text
opkg update
opkg install dnsmasq-full
```

这里会提示失败因为已经安装了 `dnsmasq`，可以先将用 `opkg download dnsmasq-full` 将 `dnsmasq-full` 的包下载到本地，然后

```text
opkg remove dnsmasq && opkg install ./dnsmasq-full*.ipk
```

在 `/etc/dnsmasq.conf` 文件的末尾加上一行

```text
conf-dir=/etc/dnsmasq.d/
```

来使 dnsmasq 从指定的目录读取配置文件，如果这个文件夹不存在可以通过 `mkdir -p /etc/dnsmasq.d` 来建立。

#### 添加 china-list 配置文件

这里主要用到的是 [china-list](https://github.com/felixonmars/dnsmasq-china-list) 也就是国内的域名列表，这个域名列表只包含指定 dns 服务器的内容，并没有我们需要的指定 ipset 的功能，所以我们需要对其进行一点修改，可以通过下面这行命令来完成这个操作

```bash
wget -O- https://raw.githubusercontent.com/felixonmars/dnsmasq-china-list/master/accelerated-domains.china.conf | awk -F "/" '{print $0; print "ipset=/"$2"/ss_spec_dst_bp"}' > /etc/dnsmasq.d/china_list.conf
```

主要用到是 awk。

最后 `/etc/init.d/dnsmasq restart` 来重启 dnsmasq

### 2、使用 cron 自动化

cron 是一个用于周期性被执行的指令的工具。

通过 `crontab -e` 可以编辑需要执行的指令

```text
 ┌───────────── minute (0 - 59)
 │ ┌───────────── hour (0 - 23)
 │ │ ┌───────────── day of the month (1 - 31)
 │ │ │ ┌───────────── month (1 - 12)
 │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday; 7 is also Sunday on some systems)
 │ │ │ │ │                                   
 │ │ │ │ │
 │ │ │ │ │
 * * * * * command to execute
```

例如我们想要每天更新一次 china-list

```text
30 3 * * * wget -O- https://raw.githubusercontent.com/felixonmars/dnsmasq-china-list/master/accelerated-domains.china.conf | awk -F "/" '{print $0; print "ipset=/"$2"/ss_spec_dst_bp"}' > /etc/dnsmasq.d/china_list.conf && /etc/init.d/dnsmasq restart
```

