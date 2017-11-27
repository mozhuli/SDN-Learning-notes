# 4.3 ovn-controller-vtep

> 本文翻译自[ovs官方手册](http://openvswitch.org/support/dist-docs/ovn-controller-vtep.8.html)

ovn-controller-vtep是对于开启了vtep的物理交换机的OVN本地控制器

## 概要

**ovn-controller-vtep**   [options]   [--vtep-db=vtep-database]  [--ovnsb-db=ovnsb-database]

## 描述

ovn-controller-vtep是OVN中的本地控制器守护程序，用于支持VTEP的物理交换机。它通过OVSDB协议连接到OVN Southbound数据库（请参阅ovn-sb），并通过OVSDB协议连接到VTEP数据库（请参阅vtep）。

## 配置

ovn-controller-vtep从ovnsb和vtep数据库中检索配置信息。如果数据库位置不是从命令行提供的，则缺省值是本地OVSDB“run”目录中的db.sock。数据路径位置必须采用以下形式之一：

   - **ssl**:ip:port

        指定的ip上的主机上指定的SSL端口，必须表示为IPv4或IPv6地址格式的IP地址（而不是DNS名称）。如果ip是一个IPv6地址，那么用方括号包住ip，例如：ssl:[::1]:6640。当指定ssl协议时--private-key, --certificate, 和--ca-cert三个参数不可省略。

    - **tcp**:ip:port

        连接到给定ip的TCP端口，其中ip可以是IPv4或IPv6地址。如果ip是一个IPv6地址，那么用方括号把ip包起来。例如：ssl:[::1]:6640

    - **unix**:file

        在POSIX上，连接到名为file的Unix域服务器套接字。

        在Windows上，连接到其值为file的本地TCP端口。