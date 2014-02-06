---
layout: post
title: "使用 SSH 隧道作端口转发"
description: "使用 SSH 隧道作端口转发"
category: "Linux"
tags: [ssh]

---

#SSH 隧道
考虑组网图如下([图片看不了的话，请打开这里](http://cl.ly/image/3t0F18232g2d))：

![组网图](http://cl.ly/image/3t0F18232g2d)

其中 gw03 在公网上，gw03 连接着 web09 等一系列内网的机器(当然gw03拥有外网和内网两个ip)，这些内网机器上显然是没有公网ip的,
那么我们的机器不能直接连接到 web09 等内网的机器上，只能通过公网上的 web09 作跳板建立 ssh 隧道做转发通行。

>ssh 隧道主要是以下几个命令:

1. 本地转发:

        ssh -C -f -N -g -L listen_port:DST_Host:DST_port user@Tunnel_Host -p ssh_port

2. 远程转发:

        ssh -C -f -N -g -R listen_port:DST_Host:DST_port user@Tunnel_Host -p ssh_port

相关参数的解释：

    -f (Fork into background after authentication.)
后台认证用户/密码，通常和 `-N` 连用，不用登录到远程主机。

    -L localport:host:hostport
将本地机(客户机)的端口 `localport` 转发到远端指定机器的指定端口。

工作原理是这样的: 本地机器上分配了一个
socket 侦听 `localport` 端口, 一旦这个端口上有了数据连接, 该连接就经过安全通道转发出去, 同时和远程主机
`host` 的 `hostport` 端口建立连接。

    -R localport:host:hostport
将远程主机(服务器) `host` 的指定端口 `hostport` 转发到本地端的指定端口 `localport`。

工作原理是这样的, 远程主机 `host` 上分配了一个 socket 侦听 `hostport` 端口, 一旦这个端口上有了连接,
该连接就经过安全通道转发出去, 同时本地主机和 host 的 hostport 端口建立连接。

    -C
压缩数据传输。

    -N(Do not execute a remote command.)
不执行脚本或命令，通常与-f连用。

    -g
在-L/-R/-D参数中，允许远程主机连接到建立转发的端口上，如果不加这个参数，只允许本地主机建立连接。

    -p ssh_port
指定对端 sshd 端口号 `ssh_port`

端口映射: 将本地 3399 端口映射到 www@121.8.9.2 27017 的端口上

    ssh -NCf -p223 www@121.8.9.2 -L 3399:127.0.0.1:27017

1. 我们想到的第一个是, 本地端口转发：

        ssh -C -f -N -g -L 37788:192.168.2.9:98 -p223 www@121.8.9.2
将本地37788端品上的数据转发到 192.168.2.9:98 端口上, 这个方法的缺点呢就是本地必须起一个监听端口号，
如果 37788 这个端口上数据传输不是很频繁的话就太浪费这个端口号了。

2. ssh 结合 nc 做端口转发, localhost 上 ~/.ssh/config 中加入配置：

        Host gw03
         HostName 218.108.129.143
         port 98
         User www
         ForwardAgent yes
        Host web09 192.168.2.9
         HostName 192.168.2.9
         port 98
         User www
         ProxyCommand ssh gw03 exec nc %h %p

         ForwardAgent yes


这样直接就可以在 localhost 中 `$ ssh web09` 登录 web09 机器了。


