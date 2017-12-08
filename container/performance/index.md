# 7.3 性能对比

## 环境准备

```text
OS: Ubuntu 14.04
kubernetes 版本：1.3.5
flannel 版本： 0.5.5
calico 版本: 1.6.0
neutron 版本：6.0.0
iperf 版本：2.0.5
```

**物理机配置：**

```text
4 Intel(R) Core(TM) i5-4460S CPU @ 2.90GHz
8G memory
千兆网卡
```

**搭建测试集群如下**

| 节点类型                 | ip地址         | 备注 |
| :-----------------      | :-------------- | :------------ |
| master/node             | 10.100.100.244 | k8s的master跟node节点共用 |
| node                    | 10.100.100.245 |  |
| neutron网络节点          | 10.100.100.246 | 为k8s提供neutron网络 |

```bash
root@hc0:~# kubectl get node
NAME           STATUS AGE
10.100.100.244 Ready 6d
10.100.100.245 Ready 6d
```

## 性能评测指标

- ping延迟: 用ping测试hosts之间和pods之间的延迟
- 带宽测试: 用iperf测试hosts之间和pods之间的带宽
- HTTP性能测试: 部署单进程nginx server并使用apache benchmark(ab)测试

## 物理机性能评测

### ping延迟

登录到10.100.100.244，运行ping 10.100.100.245测试延迟：

```sh
root@hc0:~# ping 10.100.100.245
PING 10.100.100.245 (10.100.100.245) 56(84) bytes of data.
64 bytes from 10.100.100.245: icmp_seq=1 ttl=64 time=0.247 ms
64 bytes from 10.100.100.245: icmp_seq=2 ttl=64 time=0.221 ms
64 bytes from 10.100.100.245: icmp_seq=3 ttl=64 time=0.200 ms
64 bytes from 10.100.100.245: icmp_seq=4 ttl=64 time=0.234 ms
64 bytes from 10.100.100.245: icmp_seq=5 ttl=64 time=0.211 ms
64 bytes from 10.100.100.245: icmp_seq=6 ttl=64 time=0.212 ms
64 bytes from 10.100.100.245: icmp_seq=7 ttl=64 time=0.249 ms
64 bytes from 10.100.100.245: icmp_seq=8 ttl=64 time=0.227 ms
64 bytes from 10.100.100.245: icmp_seq=9 ttl=64 time=0.245 ms
64 bytes from 10.100.100.245: icmp_seq=10 ttl=64 time=0.278 ms
64 bytes from 10.100.100.245: icmp_seq=11 ttl=64 time=0.206 ms
```

**平均值为： 0.244842ms**

> 注：可以通过下面的脚本例子计算ping延迟的平均值

```bash
root@hc0:~# ping 10.100.100.245 | head -n 20 | gawk '/time/ {split($7, ss, "="); sum+=ss[2]; count+=1;} END{print sum/count "ms";}'
0.244842ms
```

**相互之间多次ping测试。延迟一般在0.22ms-0.26ms之间**

### iperf带宽

apt-get install iperf 安装iperf后，在10.100.100.245上启动iperf server: iperf -s，在10.100.100.244上启动客户端，执行下面命令:

```bash
root@hc0:~# iperf -c 10.100.100.245 -i 1
------------------------------------------------------------
Client connecting to 10.100.100.245, TCP port 5001
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[ 3] local 10.100.100.244 port 49980 connected with 10.100.100.245 port 5001
[ ID] Interval Transfer Bandwidth
[ 3] 0.0- 1.0 sec 113 MBytes 945 Mbits/sec
[ 3] 1.0- 2.0 sec 112 MBytes 942 Mbits/sec
[ 3] 2.0- 3.0 sec 112 MBytes 941 Mbits/sec
[ 3] 3.0- 4.0 sec 112 MBytes 942 Mbits/sec
[ 3] 4.0- 5.0 sec 112 MBytes 943 Mbits/sec
[ 3] 5.0- 6.0 sec 112 MBytes 943 Mbits/sec
[ 3] 6.0- 7.0 sec 112 MBytes 941 Mbits/sec
[ 3] 7.0- 8.0 sec 112 MBytes 942 Mbits/sec
[ 3] 8.0- 9.0 sec 112 MBytes 941 Mbits/sec
[ 3] 9.0-10.0 sec 112 MBytes 943 Mbits/sec
[ 3] 0.0-10.0 sec 1.10 GBytes 942 Mbits/sec

root@hc0:~# iperf -c 10.100.100.245 -u -t 60 -b 1000M
------------------------------------------------------------
Client connecting to 10.100.100.245, UDP port 5001
Sending 1470 byte datagrams
UDP buffer size: 208 KByte (default)
------------------------------------------------------------
[ 3] local 10.100.100.244 port 53440 connected with 10.100.100.245 port 5001
[ ID] Interval Transfer Bandwidth
[ 3] 0.0-60.0 sec 5.67 GBytes 811 Mbits/sec
[ 3] Sent 4139127 datagrams
[ 3] Server Report:
[ 3] 0.0-60.0 sec 5.66 GBytes810 Mbits/sec 0.009 ms 7469/4139126 (0.18%)
[ 3] 0.0-60.0 sec 1 datagrams received out-of-order

root@hc1:~# iperf -c 10.100.100.244 -u -t 60 -b 2000M
------------------------------------------------------------
Client connecting to 10.100.100.244, UDP port 5001
Sending 1470 byte datagrams
UDP buffer size: 208 KByte (default)
------------------------------------------------------------
[ 3] local 10.100.100.245 port 46402 connected with 10.100.100.244 port 5001
[ ID] Interval Transfer Bandwidth
[ 3] 0.0-60.0 sec 5.67 GBytes 811 Mbits/sec
[ 3] Sent 4138997 datagrams
[ 3] Server Report:
[ 3] 0.0-60.0 sec 5.66 GBytes810 Mbits/sec 0.043 ms 3929/4138996 (0.095%)
[ 3] 0.0-60.0 sec 1 datagrams received out-of-order

```

**物理机iperf带宽 942 Mbits/sec**

### nginx benchmark

先在10.100.100.244上，执行docker run --net=host nginx启动一个单进程的nginx服务，然后在10.100.100.245上使用`ab -n 90000 -c 50 http://10.100.100.244/`开始压测，结果如下：

```bash
root@hc1:~# ab -n 90000 -c 50 http://10.100.100.244/
This is ApacheBench, Version 2.3 <$Revision: 1528965 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 10.100.100.244 (be patient)
Completed 9000 requests
Completed 18000 requests
Completed 27000 requests
Completed 36000 requests
Completed 45000 requests
Completed 54000 requests
Completed 63000 requests
Completed 72000 requests
Completed 81000 requests
Completed 90000 requests
Finished 90000 requests


Server Software: nginx/1.11.5
Server Hostname: 10.100.100.244
Server Port: 80

Document Path: /
Document Length: 612 bytes

Concurrency Level: 50
Time taken for tests: 7.235 seconds
Complete requests: 90000
Failed requests: 0
Total transferred: 76050000 bytes
HTML transferred: 55080000 bytes
Requests per second: 12439.61 [#/sec] (mean)
Time per request: 4.019 [ms] (mean)
Time per request: 0.080 [ms] (mean, across all concurrent requests)
Transfer rate: 10265.11 [Kbytes/sec] received

Connection Times (ms)
  min mean[+/-sd] median max
Connect: 0 0 0.1 0 2
Processing: 1 4 1.0 4 9
Waiting: 1 4 1.0 4 9
Total: 1 4 0.9 4 9

Percentage of the requests served within a certain time (ms)
  50% 4
  66% 4
  75% 4
  80% 5
  90% 5
  95% 5
  98% 5
  99% 6
 100% 9 (longest request)

```

## neutron网络评测

### ping延迟

在pod之间的ping延迟，启动两个iperf pod，启动在不同的host上。然后执行下面的命令获得ping延迟:

```bash
root@hc0:~/yaml# kubectl get pod -owide --all-namespaces
NAMESPACE NAME READY STATUS RESTARTS AGE IP NODE
net1-sub1 net-test-4v9av 1/1 Running 0 29s 192.168.11.33 10.100.100.245
net1-sub1 net-test-ztu5r   1/1 Running 0 29s 192.168.11.34 10.100.100.244
```

```bash
root@hc0:~# kubectl exec -it net-test-ztu5r bash --namespace=net1-sub1
root@net-test-ztu5r:/# ping 192.168.11.33 | head -n 20 | gawk '/time/ {split($7, ss, "="); sum+=ss[2]; count+=1;} END{print sum/count "ms";}'
0.747789ms
```

**相互之间多次ping测试，延迟一般在0.71ms-0.77ms之间**


### iperf带宽

pod之间的iperf带宽:net-test-4v9av作为server。然后从net-test-ztu5r访问net-test-4v9av的IP来测试带宽。

```bash
root@net-test-ztu5r:/# iperf -c 192.168.11.33 -u -t 60 -b 2000M
------------------------------------------------------------
Client connecting to 192.168.11.33, UDP port 5001
Sending 1470 byte datagrams
UDP buffer size: 208 KByte (default)
------------------------------------------------------------
[ 3] local 192.168.11.34 port 40863 connected with 192.168.11.33 port 5001
[ ID] Interval Transfer Bandwidth
[ 3] 0.0-60.0 sec 5.66 GBytes 810 Mbits/sec
[ 3] Sent 4131942 datagrams
[ 3] WARNING: did not receive ack of last datagram after 10 tries.
```

### nginx benchmark

pod内的nginx性能：

用RC创建Nginx pod，以及service:
```bash
kubectl create -f nginx-rc.yaml
kubectl create -f nginx-svc.yaml

root@hc0:~# kubectl get pod --namespace=net1-sub1 -owide
NAME READY STATUS RESTARTS AGE IP NODE
net-test-4v9av 1/1 Running 0 25m 192.168.11.33 10.100.100.245
net-test-ztu5r 1/1 Running 0 25m 192.168.11.34 10.100.100.244
nginx-0v5r8   1/1 Running 0 30s 192.168.11.36 10.100.100.244
nginx-pnxzt   1/1 Running 0 30s 192.168.11.35 10.100.100.245

net-test-4v9av pod使用podIP访问ab -n 90000 -c 50 http://192.168.11.36/，性能相近。：
root@net-test-4v9av:/# ab -n 90000 -c 50 http://192.168.11.36/
This is ApacheBench, Version 2.3 <$Revision: 1528965 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 192.168.11.36 (be patient)
Completed 9000 requests
Completed 18000 requests
Completed 27000 requests
Completed 36000 requests
Completed 45000 requests
Completed 54000 requests
Completed 63000 requests
Completed 72000 requests
Completed 81000 requests
Completed 90000 requests
Finished 90000 requests


Server Software: nginx/1.11.5
Server Hostname: 192.168.11.36
Server Port: 80

Document Path: /
Document Length: 612 bytes

Concurrency Level: 50
Time taken for tests: 9.928 seconds
Complete requests: 90000
Failed requests: 0
Total transferred: 76050000 bytes
HTML transferred: 55080000 bytes
Requests per second: 9065.38 [#/sec] (mean)
Time per request: 5.515 [ms] (mean)
Time per request: 0.110 [ms] (mean, across all concurrent requests)
Transfer rate: 7480.71 [Kbytes/sec] received

Connection Times (ms)
  min mean[+/-sd] median max
Connect: 0 1 1.3 0 12
Processing: 0 4 1.7 5 11
Waiting: 0 4 1.7 5 11
Total: 0 5 1.4 6 21

Percentage of the requests served within a certain time (ms)
  50% 6
  66% 6
  75% 6
  80% 6
  90% 7
  95% 7
  98% 9
  99% 9
 100% 21 (longest request)
```

## 使用flannel vxlan模式评测：

### ping延迟：

操作步骤同上

```bash
root@hc0:~/yaml# kubectl get pod -owide --all-namespaces
NAMESPACE NAME READY STATUS RESTARTS AGE IP NODE
net1-sub1 net-test-ipgc5 1/1 Running 0 9s 172.16.21.2 10.100.100.244
net1-sub1 net-test-iyfds 1/1 Running 0 9s 172.16.78.2 10.100.100.245
```

```bash
root@hc0:~# kubectl exec -it net-test-ipgc5 bash --namespace=net1-sub1
root@net-test-ipgc5:/# ping 172.16.78.2 | head -n 20 | gawk '/time/ {split($7, ss, "="); sum+=ss[2]; count+=1;} END{print sum/count "ms";}'
0.379456ms
```

**平均延迟：0.379456ms**

### iperf带宽

操作步骤同上

```bash
root@net-test-iyfds:/# iperf -c 172.16.21.2 -t 60
------------------------------------------------------------
Client connecting to 172.16.21.2, TCP port 5001
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[ 3] local172.16.78.2 port 45646 connected with 192.168.202.4 port 5001
[ ID] Interval Transfer Bandwidth
[ 3] 0.0-60.0 sec 6.28 GBytes 910 Mbits/sec
```

**flannel的iperf带宽 910 Mbits/sec**

### nginx benchmark

操作步骤同上

```bash
root@net-test-b917n:/# ab -n 90000 -c 50 http://172.16.21.3/
This is ApacheBench, Version 2.3 <$Revision: 1528965 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 172.16.21.3 (be patient)
Completed 9000 requests
Completed 18000 requests
Completed 27000 requests
Completed 36000 requests
Completed 45000 requests
Completed 54000 requests
Completed 63000 requests
Completed 72000 requests
Completed 81000 requests
Completed 90000 requests
Finished 90000 requests


Server Software: nginx/1.11.5
Server Hostname: 172.16.21.3
Server Port: 80

Document Path: /
Document Length: 612 bytes

Concurrency Level: 50
Time taken for tests: 8.945 seconds
Complete requests: 90000
Failed requests: 0
Total transferred: 76050000 bytes
HTML transferred: 55080000 bytes
Requests per second: 10804.44 [#/sec] (mean)
Time per request: 4.928 [ms] (mean)
Time per request: 0.093 [ms] (mean, across all concurrent requests)
Transfer rate: 8814.89 [Kbytes/sec] received

Connection Times (ms)
  min mean[+/-sd] median max
Connect: 0 0 0.2 0 5
Processing: 1 4 1.3 5 10
Waiting: 1 4 1.3 5 9
Total: 2 5 1.3 5 10

Percentage of the requests served within a certain time (ms)
  50% 5
  66% 5
  75% 5
  80% 5
  90% 5
  95% 6
  98% 6
  99% 6
 100% 10 (longest request)
```

## Calico网络性能

根据http://docs.projectcalico.org/v1.6/getting-started/kubernetes/installation/#configuring-kubernetes完成集群的部署

### ping延迟

评测方法同上

```bash
root@hc0:# kubectl get pod -owide --namespace=net1-sub1
NAME READY STATUS RESTARTS AGE IP NODE
net-test-b917n 1/1 Running 0 11m 192.168.145.132 10.100.100.245
net-test-w98x9 1/1 Running 0 11m 192.168.202.4   10.100.100.244
```

```bash
root@net-test-w98x9:/# ping 192.168.145.132 | head -n 20 | gawk '/time/ {split($7, ss, "="); sum+=ss[2]; count+=1;} END{print sum/count "ms";}'
0.267684ms
```

**多次测试平均延迟：0.25ms-0.31ms之间**

### iperf带宽

评测方法同上

tcp：
```bash
root@net-test-w98x9:/# iperf -s 
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size: 85.3 KByte (default)
------------------------------------------------------------
[ 4] local 192.168.202.4 port 5001 connected with 192.168.145.132 port 54228
[ ID] Interval Transfer Bandwidth
[ 4] 0.0-30.0 sec 3.29 GBytes 941 Mbits/sec

root@net-test-b917n:/# iperf -c 192.168.202.4 -t 60
------------------------------------------------------------
Client connecting to 192.168.202.4, TCP port 5001
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[ 3] local 192.168.145.132 port 54255 connected with 192.168.202.4 port 5001
[ ID] Interval Transfer Bandwidth
[ 3] 0.0-60.0 sec 6.58 GBytes 942 Mbits/sec
```

udp：
```bash
root@net-test-b917n:/# iperf -c 192.168.202.4 -u -b 1000M -t 10
------------------------------------------------------------
Client connecting to 192.168.202.4, UDP port 5001
Sending 1470 byte datagrams
UDP buffer size: 208 KByte (default)
------------------------------------------------------------
[ 3] local 192.168.145.132 port 60316 connected with 192.168.202.4 port 5001
[ ID] Interval Transfer Bandwidth
[ 3] 0.0-10.0 sec 968 MBytes 812 Mbits/sec
[ 3] Sent 690167 datagrams
[ 3] Server Report:
[ 3] 0.0-10.0 sec 966 MBytes 811 Mbits/sec 0.028 ms 762/690166 (0.11%)
[ 3] 0.0-10.0 sec 1 datagrams received out-of-order
```

**calico的iperf带宽 941 Mbits/sec**

### nginx benchmark

评测方法同上

```bash
root@net-test-b917n:/# ab -n 90000 -c 50 http://192.168.202.6/
This is ApacheBench, Version 2.3 <$Revision: 1528965 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 192.168.202.6 (be patient)
Completed 9000 requests
Completed 18000 requests
Completed 27000 requests
Completed 36000 requests
Completed 45000 requests
Completed 54000 requests
Completed 63000 requests
Completed 72000 requests
Completed 81000 requests
Completed 90000 requests
Finished 90000 requests


Server Software: nginx/1.11.5
Server Hostname: 192.168.202.6
Server Port: 80

Document Path: /
Document Length: 612 bytes

Concurrency Level: 50
Time taken for tests: 8.330 seconds
Complete requests: 90000
Failed requests: 0
Total transferred: 76050000 bytes
HTML transferred: 55080000 bytes
Requests per second: 10031.23 [#/sec] (mean)
Time per request: 4.886 [ms] (mean)
Time per request: 0.093 [ms] (mean, across all concurrent requests)
Transfer rate: 8915.77 [Kbytes/sec] received

Connection Times (ms)
  min mean[+/-sd] median max
Connect: 0 0 0.2 0 5
Processing: 1 4 1.3 5 10
Waiting: 1 4 1.3 5 9
Total: 2 5 1.3 5 10

Percentage of the requests served within a certain time (ms)
  50% 5
  66% 5
  75% 5
  80% 5
  90% 5
  95% 6
  98% 6
  99% 6
 100% 10 (longest request)
```

## ovs vxlan网络性能

### ping延迟

评测方法同上

```bash
root@hc0:# kubectl get pod -owide
NAME READY STATUS RESTARTS AGE IP NODE
net-test-278vs 1/1 Running 0 25s 172.16.75.2 10.100.100.244
net-test-vtva6 1/1 Running 0 25s 172.16.61.2 10.100.100.245
```

```bash
root@net-test-278vs:/# ping 172.16.61.2 | head -n 20 | gawk '/time/ {split($7, ss, "="); sum+=ss[2]; count+=1;} END{print sum/count "ms";}'
0.35475ms
```

### iperf带宽

评测方法同上

tcp：
```bash
root@net-test-278vs:/# iperf -c 172.16.61.2 -t 20
------------------------------------------------------------
Client connecting to 172.16.61.2, TCP port 5001
TCP window size: 45.0 KByte (default)
------------------------------------------------------------
[ 3] local 172.16.75.2 port 39574 connected with 172.16.61.2 port 5001
[ ID] Interval Transfer Bandwidth
[ 3] 0.0-20.0 sec 2.14 GBytes 917 Mbits/sec
```
**ovs的iperf带宽 917 Mbits/sec**

### nginx benchmark

评测方法同上

```bash
root@net-test-278vs:/# ab -n 90000 -c 50 http://172.16.61.3/
This is ApacheBench, Version 2.3 <$Revision: 1528965 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 172.16.61.3 (be patient)
Completed 9000 requests
Completed 18000 requests
Completed 27000 requests
Completed 36000 requests
Completed 45000 requests
Completed 54000 requests
Completed 63000 requests
Completed 72000 requests
Completed 81000 requests
Completed 90000 requests
Finished 90000 requests


Server Software: nginx/1.11.5
Server Hostname: 172.16.80.2
Server Port: 80

Document Path: /
Document Length: 612 bytes

Concurrency Level: 50
Time taken for tests: 7.855 seconds
Complete requests: 90000
Failed requests: 0
Total transferred: 76050000 bytes
HTML transferred: 55080000 bytes
Requests per second:11457.32 [#/sec] (mean)
Time per request:4.364 [ms] (mean)
Time per request: 0.087 [ms] (mean, across all concurrent requests)
Transfer rate: 9454.53 [Kbytes/sec] received

Connection Times (ms)
  min mean[+/-sd] median max
Connect: 0 2 0.3 2 4
Processing: 1 2 0.5 2 13
Waiting: 1 2 0.5 2 13
Total: 2 4 0.4 4 15

Percentage of the requests served within a certain time (ms)
  50% 4
  66% 4
  75% 5
  80% 5
  90% 5
  95% 5
  98% 5
  99% 5
 100% 15 (longest request)
```

## 结论

| 网络类型        | 延迟         | 带宽 | nginx(QPS/延迟) |
| :-----------   | :-------------- | :------------ | :--------|
| 物理 | 0.244842ms | 942Mb/s | 12439.61/4.019 |
| neutron | 0.747789ms | 907Mb/s | 9065.38/5.515 |
| flannel vxlan | 0.727158ms | 910Mb/s | 10031.23/4.886|
| Calico | 0.267684ms | 941Mb/s | 10804.44/4.628 |
| ovs vxlan | 0.35475ms | 917Mb/s | 11457.32/4.364 |
