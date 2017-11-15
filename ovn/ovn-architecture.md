# 4.1 ovn-architecture

> 本文翻译自[ovs官方手册](http://openvswitch.org/support/dist-docs/ovn-architecture.7.html)，有删减

## OVN架构

OVN（即Open Virtual Network）是一款支持虚拟网络抽象的软件系统。OVN在OVS现有功能的基础上原生支持虚拟网络抽象，例如虚拟L2，L3覆盖网络以及完全组。诸如DHCP，DNS的服务也是其关注的内容。就像OVS一样，OVN的设计目标是可以大规模运行的高质量生产级实施方案。

OVN部署由以下几个组件组成：

* CMS（云管理系统）。这是OVN的最终用户（通过其用户和管理员）。与OVN集成需要安装与CMS特定的插件和相关软件（见下文）。OVN最初的目标CMS是OpenStack。我们通常会说一个CMS，但多个CMS也可以管理一个OVN的不同部分。

* 安装在一个中央位置的OVN数据库，可以是物理节点，虚拟节点，甚至是一个集群。

* 一个或多个（通常是很多）虚拟机管理程序（hypervisors）。hypervisors必须运行Open vSwitch并执行IntegrationGuide.rst在OVS源代码树中所述的接口。任何支持的OVS的hypervisor平台都是可以接受的。

* 零个或多个网关。 网关通过在隧道和物理以太网端口之间双向转发数据包，将基于隧道的逻辑网络扩展到物理网络。这允许非虚拟机器参与逻辑网络。网关可能是物理机，虚拟机或基于ASIC同时支持vtep模式的硬件交换机。

> hypervisor和网关一起被称为传输节点或chassis

下图显示了OVN和相关的主要组件的软件交互:

```
                                         CMS
                                          |
                                          |
                              +-----------|-----------+
                              |           |           |
                              |     OVN/CMS Plugin    |
                              |           |           |
                              |           |           |
                              |   OVN Northbound DB   |
                              |           |           |
                              |           |           |
                              |       ovn-northd      |
                              |           |           |
                              +-----------|-----------+
                                          |
                                          |
                                +-------------------+
                                | OVN Southbound DB |
                                +-------------------+
                                          |
                                          |
                       +------------------+------------------+
                       |                  |                  |
         HV 1          |                  |    HV n          |
       +---------------|---------------+  .  +---------------|---------------+
       |               |               |  .  |               |               |
       |        ovn-controller         |  .  |        ovn-controller         |
       |         |          |          |  .  |         |          |          |
       |         |          |          |     |         |          |          |
       |  ovs-vswitchd   ovsdb-server  |     |  ovs-vswitchd   ovsdb-server  |
       |                               |     |                               |
       +-------------------------------+     +-------------------------------+
```

从图的顶部开始，依次为：

1. 首先是如上文所述的云管理系统。

2. OVN/CMS插件是连接到OVN的CMS组件。 在OpenStack中，这是一个Neutron插件。该插件的主要目的是转换CMS中的逻辑网络的配置为OVN可以理解的中间表示。这个组件是必须是CMS特定的，所以对接一个新的CMS需要开发新的插件对接到OVN。所有在这个组件下面的其他组件是与CMS无关的。

3. OVN北向数据库接收由OVN/CMS插件传递的逻辑网络配置的中间表示。数据库模式与CMS中使用的概念是“阻抗匹配的”，因此它直接支持逻辑交换机，路由器，ACL等概念。有关详细信息，请参见ovn-nb。（OVN北向数据库只有两个客户端：在它上面的OVN/CMS插件和在它下面的ovn-northd）

4. ovn-northd用于连接到它上面的OVN北行数据库和它下面的OVN南行数据库。它将传统网络概念（从OVN北行数据库中取得）的逻辑网络配置转换为其下面的OVN南行数据库中的逻辑数据路径流。

5. OVN南行数据库是系统的中心。（OVN南向数据库也只有两个客户端：它上面是ovn-northd以及在它下面的每个传输节点上的ovn-controller）。OVN南向数据库包含三种类型的数据：指定如何到达hypervisor和其他节点的物理网络（PN）表；用于描述逻辑网络的逻辑数据路径流的逻辑网络（LN）表；用于将逻辑网络组件的位置链接到物理网络的绑定表。hypervisor填充PN和Port_Binding表，而ovn-northd填充LN表。

   > 为了集群的可用性，OVN南行数据库性能必须随传输节点的数量而扩展。这可能需要一些ovsdb-server上的工作。

**其余组件在每个hypervisor中都是一样的**

6. ovn-controller是每个hypervisor和软件网关上的OVN代理。北向，它连接到OVN南行数据库以了解OVN配置和状态，并把hypervisor的状态填充绑定表中的Chassis列以及PN表。南向，它连接到ovs-vswitchd作为OpenFlow控制器用于控制网络通信，并连接到本地ovsdb-server以允许它监视和控制Open vSwitch的配置。

7. ovs-vswitchd和ovsdb-server是标准的Open vSwitch组件。

## OVN中的信息流

OVN中的配置数据从北向南流动。CMS通过其OVN/CMS插件，通过北向数据库将逻辑网络配置传递给ovn-northd。反过来，ovn-northd将配置信息编译为较低级别的表单，并通过南行数据库将其传递到所有的chassis。

OVN中的状态信息从南向北流动。OVN目前仅提供以下几种形式的状态信息：

   * 首先，ovn-northd在北向Logical_Switch_Port表中填充up列：如果南向Port_Binding表中逻辑端口的chassis列非空，则设置为true，否则设置为false。这使CMS能够检测虚拟机的网络连接何时可用。

   * 其次，OVN向CMS提供关于其配置实现的反馈，即CMS提供的配置是否已经生效。 此功能要求CMS参与序列号协议，其工作方式如下：

     1. 当CMS更新北向数据库中的配置时，作为同一事务的一部分，它会增加NB_Global表中的nb_cfg列的值。（这只有当CMS想知道何时配置已经实现时才是必要的。）
     2. 当ovn-northd根据北向数据库的给定快照更新南行数据库时，作为同一事务的一部分，它将北向NB_Global表中的nb_cfg列复制到南行数据库SB_Global表中。（因此，监视两个数据库的观察者可以确定南行数据库何时与北向数据库一致）
     3. 在ovn-northd从南向数据库服务器收到其更改已提交的确认后，会将北向数据库NB_Global表中的sb_cfg列更新为被推下去的nb_cfg版本号。（因此，CMS或其他观察者可以确定南行数据库何时被捕获而没有连接到南行数据库）
     4. 每个chassis上的ovn-controller进程会接收更新的南行数据库，并更新nb_cfg。该过程依次更新chassis上的Open vSwitch实例中安装的物理流。从Open vSwitch收到确认物理流已更新的确认后，它会在南行数据库的自己的chassis记录中更新nb_cfg。
     5. ovn-northd监视南向数据库中所有chassis记录中的nb_cfg列。它跟踪所有记录中的最小值，并将其复制到北行NB_Global表中的hv_cfg列中。（因此，CMS或其他观察者可以确定所有的hypervisor何时与北向数据库里的配置一致）

## Chassis启动

OVN部署中的每个chassis必须配置专用于OVN使用的Open vSwitch网桥，称为集成网桥。如果需要，系统启动脚本可以在启动ovn-controller之前创建此网桥。如果ovn-controller启动时这个网桥不存在，它会自动创建，使用下面建议的默认配置。 集成桥上的端口包括：

   * 在任何chassis上，OVN用来维护逻辑网络连接的隧道端口。ovn-controller负责添加，更新并删除这些隧道端口。
   * 在hypervisor上，将要连接到逻辑网络的任何VIF。hypervisor本身或Open vSwitch与hypervisor（IntegrationGuide.rst中描述的）之间的集成将处理此问题。（这不是OVN的一部分，也不是OVN的新功能;这是在支持OVS的虚拟机管理程序上已经完成的预先集成工作）
   * 在网关上，用于逻辑网络连接的物理端口。系统启动脚本在启动ovn-controller之前将此端口添加到网桥。在更复杂的设置中，这可以是到另一个桥的patch端口，而不是物理端口。

> 其他端口不应连接到集成网桥。特别是，连接到底层网络的物理端口（与把物理端口连接到逻辑网络的的网关端口相反）不能连接到集成网桥。 底层物理端口应该连接到单独的Open vSwitch网桥（实际上它们根本不需要连接到任何网桥）

集成桥应按照如下所述进行配置。每个这些设置的效果都记录在文件OVS-vswitchd.conf.db中：

**fail-mode=secure**：在ovn-controller启动之前，避免在隔离的逻辑网络之间切换数据包。有关更多信息，请参阅ovs-vsctl中的Controller Failure Settings部分。

**other-config:disable-in-band=true**：抑制集成网桥的带内控制流。由于OVN使用本地控制器（通过Unix域套接字）而不是远程控制器，所以这样的控制流显然是不常见的。然而，对于同一系统中的某个其他桥可能有一个带内远程控制器，在这种情况下，这可能会抑制带内控制器通常建立的流量。请参阅有关文档以获取更多信息。

> 集成网桥的习惯命名是br-int，但是其他名字也可能被使用的。

## 逻辑网络

逻辑网络实现与物理网络相同的概念，但它们通过隧道或其他封装与物理网络隔离。这允许逻辑网络具有单独的IP和其他地址空间，这些地址空间与用于物理网络的地址空间可以重叠并且没有冲突。逻辑网络拓扑结构可以不考虑底层物理网络的拓扑结构。

OVN中的逻辑网络概念包括：
   * 逻辑交换机，以太网交换机的逻辑版本。
   * 逻辑路由器，IP路由器的逻辑版本。逻辑交换机和路由器可以连接成复杂的拓扑结构。
   * 逻辑数据路径，OpenFlow交换机的逻辑版本。 逻辑交换机和路由器都是作为逻辑数据路径实现的。
   * 逻辑端口，逻辑交换机和逻辑路由器内外的连接点。 一些常见的逻辑端口类型是：
        
      * 表示VIF的逻辑端口
      * Localnet端口，代表逻辑交换机和物理网络之间的连接点。在集成网桥与底层物理端口附加到的独立Open vSwitch网桥之间的OVS patch端口即是Localnet端口。
      * 逻辑patch端口，表示逻辑交换机和逻辑路由器之间的连接点，在某些情况下，则是对等逻辑路由器之间的连接点。每个连接点都有一对逻辑连接端口，每端连接一个端口。
      * Localport端口，代表逻辑交换机和VIF之间的本地连接点。这些端口存在于每个chassis（不限于任何特定的chassis），并且来自它们的流量将永远不会穿过隧道。Localport端口仅生成目的地为本地目的地的流量，通常响应于其收到的请求。一个用例是OpenStack Neutron如何使用Localport端口将元数据提供给驻留在每个hypervisor上的虚拟机。元数据代理进程连接到每个主机上的此端口，并且同一网络中的所有虚拟机将使用相同的IP/MAC地址到达它，而不通过隧道发送任何流量。更多细节可以在[https://docs.openstack.org/developer/networking/ovn/design/metadata_api.html](https://docs.openstack.org/developer/networking/ovn/design/metadata_api.html)上看到。

## VIF的生命周期

数据库表和他们的模式是孤立的，很难理解。这是一个例子。

hypervisor上的VIF是连接到在该hypervisor上直接运行的虚拟机或容器的虚拟网络接口（这与运行在虚拟机内的容器的接口不同）

本示例中的步骤通常涉及OVN和OVN北向数据库模式的详细信息。请分别参阅ovn-sb和ovn-nb部分了解这些数据库的完整内容。
   
1. 当CMS管理员使用CMS用户界面或API创建新的VIF并将其添加到交换机（由OVN作为逻辑交换机实现的交换机）时，VIF的生命周期开始。CMS更新其自己的配置，主要包括将VIF唯一的持久标识符vif-id和以太网地址mac相关联。
2. CMS插件通过向Logical_Switch_Port表添加一行来更新OVN北向数据库以包括新的VIF信息。在新行中，名称是vif-id，mac是mac，交换机指向OVN逻辑交换机的Logical_Switch记录，而其他列被适当地初始化。
3. ovn-northd收到OVN北向数据库的更新。然后通过添加新行到OVN南向数据库Logical_Flow表中反映新的端口来对OVN南向数据库进行相应的更新，例如，添加一个流来识别发往新端口的MAC地址的数据包应该传递给它，并且更新传递广播和多播数据包的流以包括新的端口。它还在“绑定”表中创建一个记录，并填充除识别chassis列之外的所有列。
4. 在每个hypervisor上，ovn-controller接收上一步ovn-northd在Logical_Flow表所做的更新。但只要拥有VIF的虚拟机关机，ovn-controller也无能为力。例如，它不能发送数据包或从VIF接收数据包，因为VIF实际上并不存在于任何地方。
5. 最终，用户启动拥有该VIF的VM。在VM启动的hypervisor上，hypervisor与Open vSwitch（IntegrationGuide.rst中所描述的）之间的集成是通过将VIF添加到OVN集成网桥上，并在external_ids：iface-id中存储vif-id，以指示该接口是新VIF的实例。（这些代码在OVN中都不是新功能;这是已经在支持OVS的虚拟机管理程序上已经完成的预先集成工作。）
6. 在启动VM的hypervisor上，ovn-controller在新的接口中注意到external_ids：iface-id。作为响应，在OVN南向数据库中，它将更新绑定表的chassis列中链接逻辑端口从external_ids：iface-id到hypervisor的行。之后，ovn-controller更新本地虚拟机hypervisor的OpenFlow表，以便正确处理去往和来自VIF的数据包。
7. 一些CMS系统，包括OpenStack，只有在网络准备就绪的情况下才能完全启动虚拟机。为了支持这个功能，ovn-northd发现Binding表中的chassis列的某行更新了，则通过更新OVN北向数据库的Logical_Switch_Port表中的up列来向上指示这个变化，以指示VIF现在已经启动。如果使用此功能，CMS则可以通过允许VM执行继续进行后续的反应。
8. 在VIF所在的每个hypervisor上，ovn-controller注意到绑定表中完全填充的行。这为ovn-controller提供了逻辑端口的物理位置，因此每个实例都会更新其交换机的OpenFlow流表（基于OVN数据库Logical_Flow表中的逻辑数据路径流），以便通过隧道正确处理去往和来自VIF的数据包。
9. 最终，用户关闭拥有该VIF的VM。在VM关闭的hypervisor中，VIF将从OVN集成网桥中删除。
10. 在VM关闭的hypervisor上，ovn-controller注意到VIF已被删除。作为响应，它将删除逻辑端口绑定表中的Chassis列的内容。
11. 在每个hypervisor中，ovn-controller都会注意到绑定表行中空的Chassis列。这意味着ovn-controller不再知道逻辑端口的物理位置，因此每个实例都会更新其OpenFlow表以反映这一点。
12. 最终，当VIF（或其整个VM）不再被任何人需要时，管理员使用CMS用户界面或API删除VIF。CMS更新其自己的配置。
13. CMS插件通过删除Logical_Switch_Port表中的相关行来从OVN北向数据库中删除VIF。
14. ovn-northd收到OVN北向数据库的更新，然后相应地更新OVN南向数据库，方法是从OVN南向d数据库Logical_Flow表和绑定表中删除或更新与已销毁的VIF相关的行。
15. 在每个hypervisor上，ovn-controller接收在上一步中的Logical_Flow表更新并更新OpenFlow表。尽管可能没有太多要做，因为VIF已经变得无法访问，它在上一步中从绑定表中删除。


## VM内部容器接口的生命周期

OVN通过将写在OVN_NB数据库中的信息转换成每个hypervisor中的OpenFlow流表来提供虚拟网络抽象。如果OVN控制器是唯一可以修改Open vSwitch中的流的实体，则只能为多租户提供安全的虚拟网络连接。当Open vSwitch集成网桥驻留在虚拟机管理程序中时，虚拟机内运行的租户工作负载无法对Open vSwitch流进行任何更改的。

如果基础架构提供程序信任容器内的应用程序不会中断和修改Open vSwitch流，则可以在hypervisors中运行容器。当容器在虚拟机中运行时也是如此，此时需要有一个驻留在同一个VM中由OVN控制器控制的Open vSwitch集成网桥。对于上述两种情况，工作流程与上一节（“VIF的生命周期”）中的示例所解释的相同。

本节讨论在虚拟机中创建容器并将Open vSwitch集成网桥驻留在虚拟机管理程序中时，容器接口（CIF）的生命周期。在这种情况下，即使容器应用程序发生故障，其他租户也不会受到影响，因为运行在VM内的容器无法修改Open vSwitch集成桥中的流。

在虚拟机内创建多个容器时，会有多个CIF关联。为了便于OVN支持虚拟网络抽象，与这些CIF关联的网络流量需要到达hypervisor中运行的Open vSwitch集成网桥。OVN还应该能够区分来自不同CIF的网络流量。有两种方法可以区分CIF的网络流量：

  * 第一种方法是为每个CIF提供一个VIF（1:1）。这意味着hypervisor中可能有很多网络设备。由于管理所有VIF额外的CPU消耗，这会使OVS变慢。 这也意味着在VM中创建容器的实体也应该能够在hypervisor中创建相应的VIF。

  * 第二种方法是为所有CIF提供一个VIF（1:n）。然后，OVN可以通过每个数据包中写入的标签来区分来自不同CIF的网络流量。OVN使用这种机制并使用VLAN作为标记机制。

     1. CIF的生命周期始于在虚拟机内部创建容器开始，该容器可以是由创建虚拟机同一CMS创建或拥有该虚拟机的租户创建，甚至可以是与最初创建虚拟机的CMS不同的容器编制系统所创建。无论创建容器的实体是谁，它需要知道需要知道与虚拟机的网络接口关联的vif-id，容器接口的网络流量将通过该网络接口经过。创建容器接口的实体也需要在该虚拟机内部选择一个未使用的VLAN。
     2. 创建容器的实体（直接或间接通过管理底层基础结构的CMS）通过向Logical_Switch_Port表添加一行来更新OVN南向数据库以包含新的CIF。在新行中，name是任何唯一的标识符，parent_name是CIF网络流量预计要经过的VM的vif-id，tag是标识该CIF网络流量的VLAN标记。
     3. ovn-northd收到OVN北向数据库的更新。反过来，它通过添加相应行到OVN南向数据库的Logical_Flow表来反映新的端口，并通过在Binding表中创建一个新行并填充除了标识列以外的所有列，来相应地更新OVN南向数据库的chassis。
     4. 在每个hypervisor上，ovn-controller订阅绑定表中的更改。当由ovn-northd创建的新行中包含Binding表的parent_port列中的值时，拥有与external_ids：iface-id中的vif-id相同值的ovn集成网桥上的ovn-controller更新本地hypervisor的OpenFlow流表，这样来自具有特定VLAN标记的VIF的数据包得到正确的处理。之后，它会更新绑定表的chassis列以反映物理位置。
     5. 底层网络准备就绪后，只能在容器内启动应用程序。为了支持这个功能，ovn-northd在绑定表中通知更新的chassis列，并更新OVN Northbound数据库的Logical_Switch_Port表中的up列，以表示CIF现在已经启动。负责启动容器应用程序的实体查询此值并启动应用程序。
     6. 最后，创建并启动容器的实体如果要停止它。实体则通过CMS（或直接）删除Logical_Switch_Port表中的行。
     7. ovn-northd接收OVN Northbound更新，并相应地更新OVN Southbound数据库，方法是从OVN Southbound数据库Logical_Flow表中删除或更新与已经销毁的CIF相关的行。同时也会删除该CIF的绑定表中的行。
     8. 在每个hypervisor中，ovn-controller接收在上一步中Logical_Flow表的更新.ovn-controller更新本地OpenFlow表以反映更新。

## 数据包的物理处理生命周期

本节介绍数据包如何通过OVN从一个虚拟机或容器传输到另一个虚拟机或容器。这个描述着重于一个数据包的物理处理。有关数据包逻辑生命周期的描述，请参考ovn-sb中的Logical_Flow表。

为了清楚起见，本节提到了一些数据和元数据字段，总结如下：

  1. **隧道密钥**。当OVN在Geneve或其他隧道中封装数据包时，会附加额外的数据，以使接收的OVN实例正确处理。这取决于特定的封装形式，采取不同的形式，但在每种情况下，我们都将其称为“隧道密钥”。请参阅文末的“隧道封装”以了解详细信息。
  2. **逻辑数据路径字段**。表示数据包正在被处理的逻辑数据路径的字段。OVN使用OpenFlow 1.1+简单地（并且容易混淆）调用“元数据”来存储逻辑数据路径字段。 （该字段作为隧道密钥的一部分通过隧道进行传递。）
  3. **逻辑输入端口字段**。表示数据包从哪个逻辑端口进入逻辑数据路径的字段。OVN将其存储在Open vSwitch扩展寄存器14中。Geneve和STT隧道将这个字段作为隧道密钥的一部分传递。虽然VXLAN隧道没有明确的携带逻辑输入端口，OVN只使用VXLAN与网关进行通信，从OVN的角度来看，只有一个逻辑端口，因此OVN可以在输入到OVN时将逻辑输入端口字段设置为该逻辑输入的逻辑管道。
  4. **逻辑输出端口字段**。表示数据包将离开逻辑数据路径的逻辑端口的字段。在逻辑入口管道的开始，它被初始化为0。 OVN将其存储在Open vSwitch扩展寄存器15中。Geneve和STT隧道将这个字段作为隧道密钥的一部分传递。VXLAN隧道不传输逻辑输出端口字段，由于VXLAN隧道不在隧道密钥中携带逻辑输出端口字段，所以当OVN管理程序从VXLAN隧道接收到数据包时，将数据包重新提交给流表8以确定输出端口。当数据包到达流表32时，通过检查当数据包从VXLAN隧道到达时设置的MLF_RCV_FROM_VXLAN标志，将这些数据包重新提交到表33以供本地传送。
  5. **逻辑端口的conntrack区域字段**。表示逻辑端口的连接跟踪区域的字段。值只在本地有意义，跨chassis之间是没有意义的。在逻辑入口管道的开始，它被初始化为0。OVN将其存储在Open vSwitch扩展寄存器13中。
  6. **路由器的conntrack区域字段**。表示路由器的连接跟踪区域的字段。值只在本地有意义，跨chassis之间是没有意义的。OVN在Open vSwitch扩展寄存器11中存储DNATting的区域信息，在Open vSwitch扩展寄存器12中存储SNATing的区域信息。
  7. **逻辑流标志**。逻辑标志旨在处理保持流表之间的上下文以便确定后续表中的哪些规则匹配。值只在本地有意义，跨chassis之间是没有意义的。OVN将逻辑标志存储在Open vSwitch扩展寄存器10中。
  8. **VLAN ID**。VLAN ID用作OVN和嵌套在VM内的容器之间的接口（有关更多信息，请参阅上面VM内容器接口的生命周期）。

首先，hypervisor上的VM或容器通过连接到OVN集成网桥的端口上发送数据包。详细的生命周期如下：

1. 
   OpenFlow流表0执行物理到逻辑的转换。它匹配数据包的入口端口。其操作通过设置逻辑数据路径字段以标识数据包正在穿越的逻辑数据路径  以 及设置逻辑输入端口字段来标识入口端口，从而实现用逻辑元数据标注数据包。然后重新提交到表8进入逻辑入口管道。

   如果源自嵌套在虚拟机内的容器的数据包有稍微不同的方式处理。可以根据VIF特定的VLAN ID来区分始发容器，因此物理到逻辑转换流程在VLAN ID字段上还需要匹配，并且动作将VLAN标头剥离。在这一步之后，OVN像处理其他数据包一样处理来自容器的数据包。

   流表0还处理从其他chassis到达的数据包。它通过入口端口（这是一个隧道）将它们与其他数据包区分开来。与刚刚进入OVN流水线的数据包一样，这些操作使用逻辑数据路径和逻辑入口端口元数据来注释这些数据包。另外，动作设置了逻辑输出端口字段，这是可用的，因为在进行ovn隧道封装前，逻辑输出端口是已知的。这三个信息是从隧道封装元数据中获取的（请参阅隧道封装了解编码细节）。然后操作重新提交到表33进入逻辑出口管道。
2. 
   OpenFlow流表8至31执行OVN Southbound数据库中的Logical_Flow表的逻辑入口管道。这些流表完全用于逻辑端口和逻辑数据路径等逻辑概念的表示。ovn-controller的一大部分工作就是把它们转换成等效的OpenFlow（尤其翻译数据库表：把Logical_Flow表0到表23翻译为成为OpenFlow流表8到31）。

   每个逻辑流都映射到一个或多个OpenFlow流。一个实际的数据包通常只匹配其中一个，尽管在某些情况下它可以匹配多个这样的数据流（这不是一个问题，因为它们都有相同的动作）。 ovn-controller使用逻辑流的UUID的前32位作为OpenFlow流的cookie。（这不一定是唯一的，因为逻辑流的UUID的前32位不一定是唯一的。）

   一些逻辑流程可以映射到Open vSwitch“连接匹配”扩展（参见ovs-field部分文档）。流连接操作使用的OpenFlow cookie为0，因为它们可以对应多个逻辑流。一个OpenFlow流连接匹配包含conj_id上的匹配。

   如果某个逻辑流无法在该hypervisor上使用，则某些逻辑流可能无法在给定的hypervisor中的OpenFlow流表中表示。例如，如果逻辑交换机中没有VIF驻留在给定的hypervisor上，并且该hypervisor上的其他逻辑交换机不可访问（例如，从hypervisor上的VIF开始的逻辑交换机和路由器的一系列跳跃），则逻辑流可能不能在那里所表示。

   大多数OVN操作在OpenFlow中具有相当明显的实现（具有OVS扩展），例如，next实现为resubmit，field =常量; 至于set_field，下面有一些更详细地描述：

   **output**
        
    通过将数据包重新提交给表32来实现。如果流水线执行多于一个的输出动作，则将每个单独地重新提交给表32.这可以用于将数据包的多个副本发送到多个端口。（如果数据包在输出操作之间未被修改，并且某些拷贝被指定给相同的hypervisor，那么使用逻辑多播输出端口将节省hypervisor之间的带宽。）

   **get_arp(P, A)**

   **get_nd(P, A)**

   通过将参数存储在OpenFlow字段中来实现，然后重新提交到表66，ovn-controller使用OVN南向数据库中的MAC_Binding表生成的此66流表。如果表66中存在匹配，则其动作将绑定的MAC存储在以太网目的地地址字段中。

   > OpenFlow操作保存并恢复用于上述参数的OpenFlow字段，OVN操作不必知道这个临时用途。

   **put_arp(P, A, E)**

   **put_nd(P, A, E)**

   通过将参数存储在OpenFlow字段中实现，通过ovn-controller来更新MAC_Binding表。

   > OpenFlow操作保存并恢复用于上述参数的OpenFlow字段，OVN操作不必知道这个临时用途。

3. 
   OpenFlow流表32至47在逻辑入口管道中执行输出动作。具体来说就是表32处理到远程hypervisors的数据包，表33处理到hypervisors的数据包，表34检查其逻辑入口和出口端口相同的数据包是否应该被丢弃。

   逻辑patch端口是一个特殊情况。逻辑patch端口没有物理位置，并且有效地驻留在每个hypervisor上。因此，用于输出到本地hypervisor上的端口的流表33也自然地将输出实现到单播逻辑patch端口。但是，将相同的逻辑应用到作为逻辑多播组一部分的逻辑patch端口会产生数据包重复，因为在多播组中包含逻辑端口的每个hypervisor也会将数据包输出到逻辑patch端口。因此，多播组执行表32中的逻辑patch端口的输出。

   表32中的每个流匹配逻辑输出端口上的单播或多播逻辑端口，其中包括远程hypervisor上的逻辑端口。每个流的操作实现就是发送一个数据包到它匹配的端口。对于远程hypervisor中的单播逻辑输出端口，操作会将隧道密钥设置为正确的值，然后将隧道端口上的数据包发送到正确的远端hypervisor。（当远端hypervisor收到数据包时，表0会将其识别为隧道数据包，并将其传递到表33中。）对于多播逻辑输出端口，操作会将数据包的一个副本发送到每个远程hypervisor，就像单播目的地一样。如果多播组包括本地hypervisor上的一个或多个逻辑端口，则其动作也重新提交给表33. 表32还包括：

   * 根据标志MLF_RCV_FROM_VXLAN匹配从VXLAN隧道接收到的数据包的高优先级的规则，然后将这些数据包重新提交给表33进行本地传递。    从VXLAN隧道收到的数据包由于缺少隧道密钥中的逻辑输出端口字段而到达此处，因此需要将这些数据包提交到表8以确定输出端口。
   * 根据逻辑输入端口匹配从localport类型的端口接收到的数据包并将这些数据包重新提交到表33以进行本地传送的较高优先级规则。每个    hypervisor都存在localport类型的端口，根据定义，它们的流量不应该通过隧道出去。
   * 如果没有其他匹配，则重新提交到流表33的备用流。

   
   对于驻留在本地而不是远程的逻辑端口，表33中的流表类似于表32中的流表。对于本地hypervisor中的单播逻辑输出端口，操作只是重新提交给表34.对于在本地hypervisor中包括一个或多个逻辑端口的多播输出端口，对于每个这样的逻辑端口P，操作将逻辑输出端口改变为P ，然后重新提交到表34。

   
   一个特殊情况是，当数据路径上存在localnet端口时，通过切换到localnet端口来连接远程端口。在这种情况下，不是在表32中增加一个到达远程端口的流，而是在表33中增加一个流以将逻辑输出端口切换到localnet端口，并重新提交到表33，好像它被单播到本地hypervisor的逻辑端口上。

   表34匹配并丢弃逻辑输入和输出端口相同并且MLF_ALLOW_LOOPBACK标志未被设置的数据包。它重新提交其他数据包到表40。

4. 
   OpenFlow表40至63执行OVN Southbound数据库中的Logical_Flow表的逻辑出口流程。出口管道可以在分组交付之前执行验证的最后阶段。 最终，它可以执行一个输出动作，ovn-controller通过重新提交到表64来实现。流水线从不执行输出的数据包被有效地丢弃（虽然它可能已经通过穿过物理网络的隧道传送）。

   > 出口管道不能改变逻辑输出端口或进行进一步的隧道封装。

5. 
   当设置MLF_ALLOW_LOOPBACK时，表64绕过OpenFlow环回。逻辑环回在表34中被处理，但是OpenFlow默认也防止环回到OpenFlow入口端口。因此，当MLF_ALLOW_LOOPBACK被设置时，OpenFlow表64保存OpenFlow入口端口，将其设置为0，重新提交到表65以获得逻辑到物理转换，然后恢复OpenFlow入口端口，有效地禁用OpenFlow回送防止。当MLF_ALLOW_LOOPBACK未被设置时，表64流程仅仅重新提交到表65。

6. 
   OpenFlow表65执行与表0相反的逻辑到物理翻译。它匹配分组的逻辑输出端口。 其操作将数据包输出到代表该逻辑端口的OVN集成网桥端口。如果逻辑出口端口是一个与VM嵌套的容器，那么在发送数据包之前，会加上具有适当VLAN ID的VLAN标头再进行发送。

## 逻辑路由器和逻辑patch端口

通常，逻辑路由器和逻辑patch端口不具有物理位置，并且实际上驻留在hypervisor上。逻辑路由器和这些逻辑路由器后面的逻辑交换机之间的逻辑patch端口就是这种情况，VM（和VIF）连接到这些逻辑交换机。

考虑从一个虚拟机或容器发送到位于不同子网上的另一个VM或容器的数据包。数据包将按照前面“数据包的物理生命周期”部分所述，使用表示发送者所连接的逻辑交换机的逻辑数据路径，遍历表0至65。在表32中，数据包本地重新提交到hypervisor上的表33的回退流。在这种情况下，从表0到表65的所有处理都在数据包发送者所在的hypervisor上进行。

当数据包到达表65时，逻辑出口端口是逻辑patch端口。表65中的实现取决于OVS版本，虽然观察到的行为意图是相同的：

  - 在OVS版本2.6和更低版本中，表65输出到代表逻辑patch端口的OVS的patch端口。在OVS patch端口的对等体中，分组重新进入OpenFlow流表，标识其逻辑数据路径和逻辑输入端口都是基于OVS patch端口的OpenFlow端口号。

  - 在OVS 2.7及更高版本中，数据包被克隆并直接重新提交到入口管道中的第一个OpenFlow流表，将逻辑入口端口设置为对等逻辑patch端口，并使用对等逻辑patch端口的逻辑数据路径（其实是表示逻辑路由器）。

分组重新进入入口流水线以便再次遍历表8至65，这次使用表示逻辑路由器的逻辑数据路径。处理过程继续，如前一部分“数据包的物理生命周期”部分所述。当数据包到达表65时，逻辑输出端口将再次成为逻辑patch端口。以与上述相同的方式，这个逻辑patch端口将使得数据包被重新提交给OpenFlow表8至65，这次使用目标VM或容器所连接的逻辑交换机的逻辑数据路径。

该分组第三次也是最后一次遍历表8至65。如果目标VM或容器驻留在远程hypervisor中，则表32将在隧道端口上将该分组从发送者的hypervisor发送到远程hypervisor。最后，表65将直接将数据包输出到目标VM或容器。

以下部分描述了两个例外，其中逻辑路由器或逻辑patch端口与物理位置相关联的情况。

### 网关路由器

网关路由器是绑定到物理位置的逻辑路由器。这包括逻辑路由器的所有逻辑patch端口以及逻辑交换机上的所有对等逻辑patch端口。在OVN Southbound数据库中，这些逻辑patch端口的Port_Binding条目使用l3gateway类型而不是patch类型，以便区分这些逻辑patch端口绑定到chassis。

当hypervisor在代表逻辑交换机的逻辑数据路径上处理数据包时，并且逻辑出口端口是表示与网关路由器连接的l3网关端口时，该暑假包将匹配表32中流表，通过隧道端口将数据包发送给网关路由器所在的chassis。表32中的处理过程与VIF相同。

网关路由器通常用于分布式逻辑路由器和物理网络之间。分布式逻辑路由器及其后面的逻辑交换机（虚拟机和容器附加到的逻辑交换机）实际上驻留在每个hypervisor中。分布式路由器和网关路由器通过另一个逻辑交换机连接，有时也称为连接逻辑交换机。另一方面，网关路由器连接到另一个具有连接到物理网络的localnet端口的逻辑交换机。

当使用网关路由器时，DNAT和SNAT规则与网关路由器相关联，网关路由器提供可处理一对多SNAT（又名IP伪装）的中央位置。

### 分布式网关端口

分布式网关端口是逻辑路由器patch端口，可以将分布式逻辑路由器与具有本地网络端口的逻辑交换机连接起来。

分布式网关端口的主要设计目标是尽可能多地在VM或容器所在的hypervisor上本地处理流量。只要有可能，应该在该虚拟机或容器的hypervisor上完全处理从该虚拟机或容器到外部世界的数据包，最终遍历该hypervisor上的所有localnet端口实例到物理网络。只要有可能，从外部到虚拟机或容器的数据包就应该通过物理网络直接导向虚拟机或容器的hypervisor，数据包将通过localnet端口进入集成网桥。

为了允许上面段落中描述的分组分布式处理，分布式网关端口需要是有效地驻留在每个hypervisor上的逻辑patch端口，而不是绑定到特定chassis的l3个网关端口。但是，与分布式网关端口相关的流量通常需要与物理位置相关联，原因如下：

  - 
     localnet端口所连接的物理网络通常使用L2学习。任何通过分布式网关端口使用的以太网地址都必须限制在一个物理位置，这样上游的L2学习就不会混淆了。从分布式网关端口向具有特定以太网地址的localnet端口发出的流量必须在特定chassis上发送出分布式网关端口的一个特定实例。具有特定以太网地址的localnet端口（或来自与localnet端口相同的逻辑交换机上的VIF）的流量必须定向到该特定chassis上的逻辑交换机的chassis端口实例。

     由于L2学习的含义，分布式网关端口的以太网地址和IP地址需要限制在一个物理位置。为此，用户必须指定一个与分布式网关端口关联的chassis。请注意，使用其他以太网地址和IP地址（例如，一对一NAT）穿过分布式网关端口的流量不限于此chassis。

     对ARP和ND请求的回复必须限制在一个单一的物理位置，在这个位置上应答的以太网地址。这包括分布式网关端口的IP地址的ARP和ND回复，这些回复仅限于与分布式网关端口关联的用户的chassis。

  -  
     为了支持一对多SNAT（又名IP伪装），其中跨多个chassis分布的多个逻辑IP地址被映射到单个外部IP地址，将有必要以集中的方式处理特定chassis上的某些逻辑路由器。由于SNAT外部IP地址通常是分布式网关端口IP地址，并且为了简单起见，使用与分布式网关端口相关联的相同chassis。

ovn-northd文档中描述了对特定chassis的流量限制的详细信息。

虽然分布式网关端口的大部分物理位置相关方面可以通过将某些流量限制到特定chassis来处理，但还需要一个额外的机制。当数据包离开入口管道，逻辑出口端口是分布式网关端口时，需要表32中两个不同的动作集中的一个：

   - 如果可以在发送者的hypervisor上本地处理该分组（例如，一对一的NAT通信量），则该分组应该以正常的方式重新提交到表33，用于分布式逻辑patch端口。
   - 但是，如果数据包需要在与分布式网关端口关联的chassis（例如，一对多SNAT流量或非NAT流量）上处理，则表32必须将该数据包在隧道端口上发送到该chassis。

为了触发第二组动作，已经添加了chassisredirect类型的南向数据库的Port_Binding表。将逻辑出口端口设置为类型为chassisredirect的逻辑端口只是一种方式，表明虽然该数据包是指向分布式网关端口的，但需要将其重定向到不同的chassis。在表32中，具有该逻辑输出端口的分组被发送到特定的chassis，与表32将逻辑输出端口是VIF或者类型为13的网关端口的分组指向不同的chassis的方式相同。一旦分组到达该chassis，表33将逻辑出口端口重置为代表分布式网关端口的值。对于每个分布式网关端口，除了表示分布式网关端口的分布式逻辑patch端口外，还有一个类型为chassisredirect的端口。

### 分布式网关端口的高可用性

OVN允许您为分布式网关端口指定chassis的优先级列表。这是通过将多个Gateway_Chassis行与OVN_Northbound数据库中的Logical_Router_Port关联来完成的。

当为网关指定了多个chassis时，所有可能向该网关发送数据包的chassis都将启用到所有配置的网关chassis的通道上的BFD。当前网关的主chassis是目前最高优先级的网关chassis，当前是根据BFD状态判断为主chassis。

有关L3网关高可用性的更多信息，请参阅
http://docs.openvswitch.org/en/latest/topics/high-availability

## VTEP网关的生命周期

网关其实是一种chassis，用于在逻辑网络的OVN管理部分和物理VLAN之间转发流量，将基于隧道的逻辑网络扩展到物理网络。

以下步骤通常涉及OVN和VTEP数据库模式的详细信息。请分别参阅ovn-sb，ovn-nb和vtep，了解关于这些数据库的全文。

1. VTEP网关的生命周期始于管理员将VTEP网关注册为VTEP数据库中的Physical_Switch表项。连接到此VTEP数据库的ovn-controller-vtep将识别新的VTEP网关，并在OVN_Southbound数据库中为其创建新的Chassis表条目。

2. 然后，管理员可以创建一个新的Logical_Switch表项，并将VTEP网关端口上的特定vlan绑定到任何VTEP逻辑交换机。一旦VTEP逻辑交换机绑定到VTEP网关，ovn-controller-vtep将检测到它，并将其名称添加到OVN_Southbound数据库中chassis表的vtep_logical_switches列。请注意，VTEP逻辑交换机的tunnel_key列在创建时未被填充。当相应的vtep逻辑交换机绑定到OVN逻辑网络时，ovn-controller-vtep将设置列。

3. 现在，管理员可以使用CMS将VTEP逻辑交换机添加到OVN逻辑网络。为此，CMS必须首先在OVN_Northbound数据库中创建一个新的Logical_Switch_Port表项。然后，该条目的类型列必须设置为“vtep”。 接下来，还必须指定options列中的vtep-logical-switch和vtep-physical-switch密钥，因为多个VTEP网关可以连接到同一个VTEP逻辑交换机。

4. OVN_Northbound数据库中新创建的逻辑端口及其配置将作为新的Port_Binding表项传递给OVN_Southbound数据库。 ovn-controller-vtep将识别更改并将逻辑端口绑定到相应的VTEP网关chassis。禁止将同一个VTEP逻辑交换机绑定到不同的OVN逻辑网络，则会在日志中产生警告。

5. 除绑定到VTEP网关chassis外，ovn-controller-vtep会将VTEP逻辑交换机的tunnel_key列更新为绑定OVN逻辑网络的相应Datapath_Binding表项的tunnel_key。

6. 接下来，ovn-controller-vtep将对OVN_Northbound数据库中的Port_Binding中的配置更改作出反应，并更新VTEP数据库中的Ucast_Macs_Remote表。这使得VTEP网关可以了解从哪里转发来自扩展外部网络的单播流量。

7. 最终，当管理员从VTEP数据库取消注册VTEP网关时，VTEP网关的生命周期结束。ovn-controller-vtep将识别该事件并删除OVN_Southbound数据库中的所有相关配置（chassis表条目和端口绑定）。

8. 当ovn-controller-vtep终止时，将清除OVN_Southbound数据库和VTEP数据库中的所有相关配置，包括所有注册的VTEP网关及其端口绑定的Chassis表条目以及所有Ucast_Macs_Remote表项和Logical_Switch隧道密钥。

## 安全

### 针对南向数据库的基于角色的访问控制

为了提供额外的安全性以防止OVNchassis受到攻击，从而防止流氓软件对南向数据库状态进行任意修改，从而中断OVN网络，从而有了针对南向数据库的基于角色的访问控制策略（请参阅ovsdb-server部分以获取更多详细信息）。

基于角色的访问控制（RBAC）的实现需要在OVSDB模式中添加两个表：RBAC_Role表，该表由角色名称进行索引，并将可以对给定角色修改的各个表的名称映射到权限表中的单个行，其中包含该角色的详细权限信息，权限表本身由包含以下信息的行组成：

**Table Name**

    关联表的名称。本栏主要作为帮助人们阅读本表内容。

**认证标准**

    一组包含列名称的字符串。至少一个列或列的内容：要修改，插入或删除的行中的键值必须等于尝试作用于该行的客户端的ID,以便授权检查通过。如果授权标准为空，则禁用授权检查，并且该角色的所有客户端将被视为授权。

**插入/删除**

    行插入/删除权限(布尔值)，指示相关表是否允许插入和删除行。 如果为true，则授权客户端允许插入和删除行。

**可更新的列**

    一组包含可能由授权客户端更新或变异的列或列名称的密钥对。只有在客户的授权检查通过并且所有要修改的列都包含在这组可修改的列中时，才允许修改行中的列。

OVN南行数据库的RBAC配置由ovn-northd维护。启用RBAC时，只允许对Chassis，Encap，Port_Binding和MAC_Binding表进行修改，并且重新分配如下：

**Chassis**

   - 授权：客户端ID必须与机箱名称匹配。
   - 插入/删除：允许授权的行插入和删除。
   - 更新：授权时可以修改列nb_cfg，external_ids，encaps和vtep_logical_switches。

**Encap授权**

   - 授权：禁用（所有客户端都被认为是授权的)。未来在这个表格中添加一个“创建chassis名称”列，并用它来进行授权检查。
   - 插入/删除：允许授权的行插入和删除。
   - 更新：列类型，选项和IP可以修改。

**Port_Binding**

   - 授权：禁用（所有客户端都被认为是授权的)。未来的增强可能会添加列（或关键字external_ids），以控制哪个chassis允许绑定每个端口。
   - 插入/删除：不允许行插入/删除（ovn-northd维护此表中的行）
   - 更新：只允许对机箱列进行修改。

**MAC_Binding**

   - 授权：禁用（所有客户被认为是授权
   - 插入/删除：允许行插入/删除。
   - 更新：logical_port，ip，mac和datapath列可以由ovn-controller修改。

为ovn-controller连接到南行数据库启用RBAC需要以下步骤：

1. 为证书CN字段设置为chassis名称的每个chassis创建SSL证书（例如，对于带有external-ids：system-id = chassis-1的chassis，通过命令“ovs-pki -B 1024 -u req + sign chassis-1 switch“）。

2. 当连接到南向数据库时配置每个ovn-controller使用SSL。（例如通过"ovs-vsctl  set  open . external-ids:ovn-remote=ssl:x.x.x.x:6642")。

3. 使用“ovn-controller”角色配置南向数据库SSL远程。(例如通过 "ovn-sbctl   set-connection role=ovn-controller pssl:6642")。


## 设计决策

### 隧道封装

OVN标注它从一个hypervisor发送的逻辑网络数据包到另一个hypervisor，包含以下三个元数据，这是在封装特定的方式编码：

1. 24位逻辑数据路径标识符，来自OVN Southbound数据库Datapath_Binding表中的隧道密钥列。
2. 15位逻辑入口端口标识符。ID 0保留在OVN内部供内部使用。可以将ID 1到32767（包括端点）分配给逻辑端口（请参阅OVN Southbound Port_Binding表中的tunnel_key列）。
3. 16位逻辑出口端口标识符。ID 0到32767与逻辑入口端口具有相同的含义。可以将包含32768到65535的ID分配给逻辑组播组（请参阅OVN Southbound Multicast_Group表中的tunnel_key列）。

对于hypervisor到hypervisor的流量，OVN仅支持Geneve和STT封装，原因如下：

1. 只有STT和Geneve支持OVN使用的大量元数据（每个数据包超过32位）（如上所述）。
2. STT和Geneve使用随机化的UDP或TCP源端口，允许在底层使用ECMP的环境中在多个路径之间进行高效分发。
3. NIC支持对STT和Geneve封装和解封装。

由于其灵活性，hypervisor之间的首选封装是Geneve。对于Geneve封装，OVN在Geneve VNI中传输逻辑数据路径标识符。OVN将类别为0x0102，类型为0x80的TLV中的逻辑入口端口和逻辑出口端口从MSB传输到LSB，其编码为32位，如下所示：

```
         1       15          16
       +---+------------+-----------+
       |rsv|ingress port|egress port|
       +---+------------+-----------+
         0
```

网卡缺少支持Geneve的环境可能更喜欢STT封装以达到性能的原因。对于STT封装，OVN将STT 64位隧道ID中的所有三个逻辑元数据编码如下，从MSB到LSB：

```
           9          15          16         24
       +--------+------------+-----------+--------+
       |reserved|ingress port|egress port|datapath|
       +--------+------------+-----------+--------+
           0
```

为了连接到网关，除了Geneve和STT之外，OVN还支持VXLAN，因为在机架交换机(ToR）上只支持VXLAN。目前，网关具有与由VTEP模式定义的能力匹配的特征集，因此元数据的比特数是必需的。 将来，不支持封装大量元数据的网关可能会继续减少功能集。