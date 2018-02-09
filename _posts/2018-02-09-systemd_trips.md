---
layout: post
title: 谈一谈在配置systemd的服务的时候遇到的坑
---

{{ page.title }}
================

<p class="meta">9 Feb 2018 - San Francisco</p>

在开发智能客服的过程中，需要在阳光保险的新乡市服务器上部署一个程序intellianswer，我希望它可以一直存在，所以就使用sytemd来管理。

但是，在解决的过程中，遇到了很多坑。

首先编写/usr/lib/systemd/system/intellianswer.service：

----------
[Unit]
Description=intellianswer
After=network.target
Wants=network-online.target

[Service]
User=odin
Group=odin
Type=simple
WorkingDirectory=/search/odin/intelli/
ExecStartPre=-/bin/mkdir -p /search/odin/intelli/
ExecStart=/search/odin/intelli/intellianswer --serviceConfig=/usr/local/etc/ia_services.json --logoutdir=/search/odin/intelli/logs/ --listenport=9098 --forceWriteLogfile
Restart=always
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target

----------


创建启动用户：
groupadd odin
useradd -g odin odin

启动该程序：
systemctl daemon-reload
systemctl stop intellianswer
systemctl enable intellianswer
systemctl start intellianswer
systemctl status intellianswer
journalctl -xe



因为systemd的报错非常“风马牛不相及”，遇到了坑，也难以解决。

在解决的过程中遇到的几个坑，记录下来，作为以后的经验：

1，目录权限问题，

一定要用odin用户创建/search/odin/intelli/ 目录，否则就完蛋了

可以先用root用户创建，但一定要
chown odin:odin /search/odin/intelli/

2，程序本身不能进行daemonize

一开始，我的程序命令是：/search/odin/intelli/intellianswer --serviceConfig=/usr/local/etc/ia_services.json --logoutdir=/search/odin/intelli/logs/ --listenport=9098 --daemonize

systemctl start intellianswer
一执行，主进程就退出了，百思不得其解。

后来才意识到，systemd的PID是1，所谓的守护进程，其实也就是systemd的子进程，所以程序内部不能进行daemonize的操作，否则，怎么也无法启动成功的

增加了一个参数--forceWriteLogfile，才解决了问题


3，程序本身不要往stdout写东西

如果程序往标准输出写东西，那么该程序的启动状态会一直处于未启动完成的状态，这并不是失败，而是没有启动完

通过增加参数--forceWriteLogfile，让程序即使没有--daemonize，也可以把日志输出到文件，这样，就可以启动成功了
--

