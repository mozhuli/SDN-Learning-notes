# 4.6 ovn-ctl

> 本文翻译自[ovs官方手册](http://openvswitch.org/support/dist-docs/ovn-nb.5.html)

## 概要

本文档主要介绍OVN_Northbound数据库模式。

这个数据库是OVN和云管理系统（CMS）之间的接口，比如运行在上面的OpenStack。CMS产生几乎所有的数据库内容。ovn-northd程序监视数据库内容，将其转换并存储到OVN_Southbound数据库中。

我们常说一个“CMS”，但也可以是多个CMS管理OVN部署的不同部分。

**外部ID**

    此数据库中的每个表都包含一个名为external_ids的特殊列。这个列在每个出现的地方都有相同的形式和目的。

    external_ids是供CMS使用的键值对。例如，CMS可能使用特定的键值对来标识其自身配置中与此数据库中相对应的实体。

## 数据库表摘要

以下列表总结了OVN_Northbound数据库中每个表的用途。每个表在后面的页面中都有更详细的描述。

| 表                | 目的 |
| :-----------------  | :----------- |
| NB_Global           | 北向数据库配置 |
| Logical_Switch      | L2逻辑交换机  |
| Logical_Switch_Port | L2逻辑交换机端口 |
| Address_Set         | 地址集  |
| Load_Balancer       | 负载均衡器  |
| ACL                 | 访问控制列表规则  |
| Logical_Router      | L3逻辑路由器 |
| QoS                 | 服务质量  |
| Logical_Router_Port | L3逻辑路由器端口  |
| Logical_Router_Static_Route | 逻辑路由器静态路由 |
| NAT                 | NAT规则 |
| DHCP_Options        | DHCP选项 |
| Connection          | OVSDB客户端连接  |
| DNS                 | 本地DNS解析  |
| SSL                 | SSL配置 |
| Gateway_Chassis     | Gateway_Chassis配置 |
