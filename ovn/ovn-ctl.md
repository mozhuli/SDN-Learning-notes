# 4.4 ovn-ctl

> 本文翻译自[ovs官方手册](http://openvswitch.org/support/dist-docs/ovn-ctl.8.html)

ovn-ctl northd生命周期管理程序

## 概要

**ovn-ctl** [options] command

## 描述

此程序主要由OVN的启动脚本在内部调用，系统管理员通常不应该直接调用它。

## COMMANDS

       start_northd
       start_controller
       start_controller_vtep
       stop_northd
       stop_controller
       stop_controller_vtep
       restart_northd
       restart_controller
       restart_controller_vtep
       promote_ovnnb
       promote_ovnsb
       demote_ovnnb
       demote_ovnsb
       status_ovnnb
       status_ovnsb

## OPTIONS

```
       --ovn-northd-priority=NICE

       --ovn-northd-wrapper=WRAPPER

       --ovn-controller-priority=NICE

       --ovn-controller-wrapper=WRAPPER

       -h | --help
```

## FILE LOCATION OPTIONS

```
       --db-sock=SOCKET

       --db-nb-file=FILE

       --db-sb-file=FILE

       --db-nb-schema=FILE

       --db-sb-schema=FILE

       --db-sb-create-insecure-remote=yes|no

       --db-nb-create-insecure-remote=yes|no

       --ovn-controller-ssl-key=KEY

       --ovn-controller-ssl-cert=CERT

       --ovn-controller-ssl-ca-cert=CERT

       --ovn-controller-ssl-bootstrap-ca-cert=CERT
```

## ADDRESS AND PORT OPTIONS

```
       --db-nb-sync-from-addr=IP ADDRESS

       --db-nb-sync-from-port=PORT NUMBER

       --db-nb-sync-from-proto=PROTO

       --db-sb-sync-from-addr=IP ADDRESS

       --db-sb-sync-from-port=PORT NUMBER

       --db-sb-sync-from-proto=PROTO
```

## 配置文件

以下是可选的配置文件。如果存在，它应该位于etc目录下:

   - **ovnnb-active.conf**
    
        如果存在，这个文件应该保存url来连接到活动的北向数据库服务器。

        tcp:x.x.x.x:6641

    - **ovnsb-active.conf**

        如果存在，这个文件应该保存url来连接到活动的南向数据库服务器。

        tcp:x.x.x.x:6642

    - **ovn-northd-db-params.conf**

        如果存在，即使--ovn-manage-ovsdb = yes，start_northd也不会启动数据库服务器。这个文件应该把数据库的url参数传递给ovn-northd。

        --ovnnb-db=tcp:x.x.x.x:6641 --ovnsb-db=tcp:x.x.x.x:6642

## 示例用法

```
   Run ovn-controller on a host already running OVS
       # ovn-ctl start_controller

   Run ovn-northd on a host already running OVS
       # ovn-ctl start_northd

   All-in-one OVS+OVN for testing
       # ovs-ctl start --system-id="random"

       # ovn-ctl start_northd

       # ovn-ctl start_controller

   Promote and demote ovsdb servers
       # ovn-ctl promote_ovnnb

       # ovn-ctl promote_ovnsb

       # ovn-ctl  --db-nb-sync-from-addr=x.x.x.x  --db-nb-sync-from-port=6641 demote_ovnnb

       # ovn-ctl  --db-sb-sync-from-addr=x.x.x.x  --db-sb-sync-from-port=6642 demote_ovnsb
```