# 2.5 arp

## arp协议

ARP协议是“Address Resolution Protocol”（地址解析协议）的缩写。在局域网中，网络中实际传输的是“帧”，帧里面是有目标主机的MAC地址的。在以太网中，一个主机要和另一个主机进行直接通信，必须要知道目标主机的MAC地址。但这个目标MAC地址是如何获得的呢？它就是通过地址解析协议获得的。所谓“地址解析”就是主机在发送帧前将目标IP地址转换成目标MAC地址的过程。ARP协议的基本功能就是通过目标设备的IP地址，查询目标设备的MAC地址，以保证通信的顺利进行。

- （1）ARP负责将IP地址映射到对应的硬件地址
- （2）RARP负责相反的过程，通常用于无盘系统。

## arp缓存表

每台安装有TCP/IP协议的电脑里都有一个ARP缓存表，表里的IP地址与MAC地址是一一对应的。

```
$ ip neigh
10.10.105.254 dev eth0 lladdr 84:5b:12:32:57:f6 REACHABLE

$ arp -a
? (10.10.105.254) 位于 84:5b:12:32:57:f6 [ether] 在 eth0
```

ip neigh 和arp 命令都可以管理操作arp缓存表

### ip neigh 命令

```sh

$ ip neigh help
Usage: ip neigh { add | del | change | replace } { ADDR [ lladdr LLADDR ]
          [ nud { permanent | noarp | stale | reachable } ]
          | proxy ADDR } [ dev DEV ]
       ip neigh {show|flush} [ to PREFIX ] [ dev DEV ] [ nud STATE ]

# 在设备eth0上，为地址10.0.0.3添加一个permanent ARP条目： 

    ip neigh add 10.0.0.3 lladdr 0:0:0:0:0:1 dev eth0 nud perm 

# 把状态改为reachable 

    ip neigh chg 10.0.0.3 dev eth0 nud reachable

# 删除设备eth0上的一个ARP条目10.0.0.3 

    ip neigh del 10.0.0.3 dev eth0
```
### arp命令

```sh

$ arp --help
用法：
  arp [-vn]  [<HW>] [-i <if>] [-a] [<hostname>]                 <-显示 ARP 缓存
  arp [-v]  [-i <if>] -d  <host> [pub]  <- 删除ARP记录
  arp [-vnD] [<HW>] [-i <if>] -f  [<filename>]      <- 从文件添加记录
  arp [-v]   [<HW>] [-i <if>] -s  <host> <hwaddr> [temp]   <-添加记录
  arp [-v]   [<HW>] [-i <if>] -Ds <host> <if> [netmask <nm>] pub          <-''-

        -a                       以另一种（BSD）风格显示（所有）主机
        -s, --set                设置一个新的 ARP 记录
        -d, --delete             删除指定记录
        -v, --verbose            显示详细信息
        -n, --numeric            不解析名称
        -i, --device             指定网络接口（如 eth0）
        -D, --use-device         读取所给定设备的硬件地址
        -A, -p, --protocol       指定协议族
        -f, --file               从文件或 /etc/ethers 中读取新记录

  <HW>=使用 '-H <hw>' 指定硬件地址类型。默认：ether
  所有可能硬件类型列表:
    ash (Ash) ether (以太网) ax25 (AMPR AX.25) 
    netrom (AMPR NET/ROM) rose (AMPR ROSE) arcnet (ARCnet) 
    dlci (Frame Relay DLCI) fddi (Fiber Distributed Data Interface) hippi (HIPPI) 
    irda (IrLAP) x25 (generic X.25) eui64 (Generic EUI-64)

arp 显示当前的ARP缓存列表。
arp -s ip mac 添加静态ARP记录，如果需要永久保存，应该编辑/etc/ethers文件。
arp -f 使/etc/ethers中的静态ARP记录生效。
arp -d 删除ARP记录。
```


## arp工作原理

我们以主机A（192.168.1.1）向主机B（192.168.1.2）发送数据为例。
1. 当发送数据时，主机A会在自己的ARP缓存表中寻找是否有目标IP地址。如果找到了，也就知道了目标MAC地址，直接把目标MAC地址写入帧里面发送就可以了。
2. 如果在ARP缓存表中没有找到相对应的IP地址，主机A就会在网络上发送一个广播，目标MAC地址是“FF.FF.FF.FF.FF.FF”，这表示向同一网段内的所有主机发出这样的询问：“192.168.1.2的MAC地址是什么？”
3. 网络上其他主机并不响应ARP询问，只有主机B接收到这个帧时，才向主机A做出这样的回应：“192.168.1.2的MAC地址是00-aa-00-62-c6-09”。
4. 主机A就知道了主机B的MAC地址，它就可以向主机B发送信息了。同时主机A还更新了自己的ARP缓存表，下次再向主机B发送信息时，直接从ARP缓存表里查找就可以了。与此同时，主机B也会更新ARP缓存表。这样主机A与主机B可以双向直接通信了。

ARP缓存表采用了老化机制，在一段时间内如果表中的某一行没有使用，就会被删除，这样可以大大减少ARP缓存表的长度，加快查询速度。

## arp分组格式

![](images/arp.jpg)

其中：

* ARP协议的帧类型为0x0806
* 硬件类型：1表示以太网地址
* 协议类型：0x800表示IP协议
* 硬件地址长度：值为6
* 协议地址长度：值为4
* op：1为ARP请求，2为ARP应答，3为RARP请求，4为RARP应答
* 对于ARP请求来说，目的端硬件地址为广播地址f:ff:ff:ff:ff:ff），由ARP相应的主机填入。

一个完整ARP请求应答的抓包：

```sh
# tcpdump -e -p arp -n -vv
21:08:10.329163 00:16:3e:01:79:43 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.14.23 tell 192.168.13.43, length 28
21:08:10.329626 00:16:3e:01:7b:17 > 00:16:3e:01:79:43, ethertype ARP (0x0806), length 60: Ethernet (len 6), IPv4 (len 4), Reply 192.168.14.23 is-at 00:16:3e:01:7b:17, length 46
```

## 参考文档

* [1] https://feisky.gitbooks.io/sdn/basic/arp.html
