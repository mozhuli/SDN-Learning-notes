# 4.2 ovn-controller

> 本文翻译自[ovs官方手册](http://openvswitch.org/support/dist-docs/ovn-controller.8.html)，有删减

## 概要

ovn-controller [options] [ovs-database]

## 描述

ovn-controller是OVN（开放虚拟网络）的本地控制器守护进程。向上它通过OVSDB协议连接到OVN Southbound数据库（请参阅ovn-sb部分），向下通过OVSDB协议连接到Open vSwitch数据库（请参阅ovs-vswitchd.conf.db部分）以及通过OpenFlow连接到ovs-vswitchd. OVN部署中的每个hypervisor和软件网关都运行自己独立的ovn-controller副本; 因此，ovn-controller的下行连接的是本地机器，而不会在物理网络上运行。

## ACL注册

ACL日志消息通过ovn-controller的日志记录机制进行记录的。ACL日志条目在日志级信息中具有模块acl_log。配置日志记录在“日志记录选项”部分中进行了介绍。

## 参数选项

### 守护进程选项

- **--pidfile[=pidfile]**
    
    创建一个文件（默认情况下，program.pid）以指示正在运行的进程的PID。如果没有指定pidfile参数，或者它不以/开头，那么它将在 **/usr/local/var/run/openvswitch**中创建。如果未指定--pidfile，则不会创建pidfile。

- **--overwrite-pidfile**

    默认情况下，当指定--pidfile并且指定的pidfile已经存在并被正在运行的进程锁定时，守护进程会拒绝启动。当指定--overwrite-pidfile会覆盖pidfile。当没有指定--pidfile时，这个选项不起作用。

- **--detach**

    运行该程序作为后台进程。进程分叉，并在子进程中启动一个新的会话，关闭标准文件描述符（这有副作用，禁止登录到控制台），并将其当前目录更改为根目录（除非指定了--no-chdir）。子进程完成初始化后，父进程退出。

- **--monitor**

    创建一个额外的进程来监视这个程序。如果由于指示程序错误的信号（SIGABRT，SIGALRM，SIGBUS，SIGFPE，SIGILL，SIGPIPE，SIGSEGV，SIGXCPU或SIGXFSZ）而死亡，则监视器进程启动它的新副本。如果守护进程因其他原因死亡或退出，监视进程将退出。

    这个选项通常和--detach一起使用，但是没有它也能起作用。

- **--no-chdir**

    默认情况下，当指定--detach时，守护进程在分离之后将其当前工作目录更改为根目录。否则，从粗心选择的目录中调用守护程序将会阻止管理员卸载保存该目录的文件系统。

    指定--no-chdir会禁止此行为，从而防止守护程序更改其当前工作目录。这对于收集核心文件可能是有用的，因为将核心文件转储写入到当前工作目录是一个常见的行为，而且根目录不是一个很好的选择。

    未指定--detach时，此选项不起作用。

- **--no-self-confinement**

    默认情况下，这个守护进程会自我限制自己在编译时使用白名单中众所周知的目录下的文件。坚持这种默认行为并不需要使用这个标志，除非使用其他一些访问控制来限制守护进程。请注意，与通常从内核空间（例如DAC或MAC）实施的其他访问控制实现相比，自限制是从用户空间守护进程本身施加的，因此不应被视为完全限制策略，而应该是被视为一个额外的安全层。

- **--user**=user:group

    此程序以user：group中指定的其他用户身份运行，从而删除了大部分root权限。简短形式的user和：group也被允许，分别假定当前的用户或组。只有root用户启动的守护进程接受这个参数。

    在Linux上，在删除root权限之前，守护进程将被授予CAP_IPC_LOCK和CAP_NET_BIND_SERVICES。与数据路径交互的守护进程（例如ovs-vswitchd）将被授予两个额外的功能，即CAP_NET_ADMIN和CAP_NET_RAW。 即使新用户是root用户，功能更改也将适用。

    在Windows上，此选项目前不受支持。出于安全原因，指定此选项将导致守护进程不启动。

### 日志选项

- **-v**[spec]

  **--verbose**=[spec]

    设置日志级别。没有任何spec将每个模块和目标的日志级别设置为dbg。否则，spec是由空格或逗号或冒号分隔的单词列表，最多可以从下面的每个类别中选择一个：

    ·  ovs-appctl上的vlog/list命令显示的有效模块名称将日志级别更改限制为指定的模块。

    ·  系统日志，控制台或文件，以将日志级别更改分别限制为系统日志，控制台或文件。（如果指定了--detach，守护进程将关闭其标准文件描述符，因此登录到控制台将不起作用）。在Windows平台上，syslog被接受为一个单词，并且只与--syslog-target选项一起使用（否则该选项无效）

    ·  off，emer，err，warn，info或dbg来控制日志级别。给定严重级别或更高级别的消息将被记录，严重性较低的消息将被过滤掉。off过滤出所有消息。请参阅ovs-appctl部分了解每个日志级别的定义。

    无论为文件设置的日志级别如何，除非指定了--log-file（参见下文），否则不会记录到文件。

- **-v**
       
  **--verbose**

    设置最大日志记录详细级别，相当于--verbose=dbg

- **-vPATTERN**:destination:pattern

  **--verbose**=PATTERN:destination:pattern

    将目标的日志模式设置为pattern。请参阅ovs-appctl了解有关pattern的有效语法的说明。

- **-vFACILITY**:facility
       
  **--verbose**=FACILITY:facility

    设置日志消息的RFC5424功能，facility 可以是kern, user, mail, daemon, auth, syslog, lpr, news, uucp, clock,ftp, ntp, audit, alert, clock2, local0,  local1,  local2,  local3, local4, local5, local6, local7中的一个。如果未指定此选项，则将daemon用作本地系统日志的默认值，并在向通过--syslog-target选项提供的目标发送消息时使用local0。

- **--log-file**[=file]

    启用日志记录到文件。如果指定了文件，则将其用作日志文件的确切名称。如果省略文件，则使用缺省日志文件名称/usr/local/var/log/openvswitch/program.log。

- **--syslog-target**=host:port

    除系统syslog之外，还要将系统日志消息发送到主机上的UDP端口。主机必须是数字IP地址，而不是主机名。

- **--syslog-method**=method

    指定方法作为如何将系统日志消息发送到syslog守护进程。支持以下形式：

    · libc，使用libc syslog（）函数，这是默认的行为。使用这个选项的不利之处在于，libc在每个消息中添加固定的前缀，然后通过/dev/log UNIX域套接字实际发送到syslog守护进程。

    · unix：file，直接使用UNIX域套接字。使用此选项可以指定任意消息格式。但是，rsyslogd 8.9及更早版本无论如何都使用硬编码的解析器函数，这限制了UNIX域套接字的使用。如果您想使用较老的rsyslogd版本使用任意的消息格式，请改为使用UDP套接字到本地主机IP地址。

    · udp：ip：port，使用UDP套接字。使用这种方法，也可以使用较老的rsyslogd来使用任意的消息格式。在通过UDP套接字发送系统日志消息时，需要考虑额外的预防措施，例如，指定的UDP端口，意外的iptables规则可能干扰本地系统日志通信，并且存在一些适用于UDP套接字的安全性考虑因素，但不适用于UNIX域套接字。

### PKI 选项

为了使用SSL连接到北向和南向数据库，需要进行PKI配置。

- **-p** privkey.pem
              
  **--private-key**=privkey.pem

    指定包含用作出SSL连接标识的私钥的PEM文件

- **-c** cert.pem

  **--certificate**=cert.pem

    指定包含证书的PEM文件，该证书证明-p或--private-key上指定的私钥是可信的。证书必须由证书颁发机构（CA）签署，SSL连接中的对等方将使用该证书对其进行验证。

- **-C** cacert.pem
              
  **--ca-cert**=cacert.pem

    指定一个包含CA证书的PEM文件，用于验证由SSL对等体提供给该程序的证书（这可能与SSL对等体用于验证在-c或--certificate上指定的证书相同的证书，也可能是不同的证书 ，这取决于正在使用的PKI设计。）

- **-C** none
              
  **--ca-cert**=none

    禁用SSL对等体提供的证书的验证。但这会带来安全风险，因为这意味着证书不能被验证为已知可信主机的证书。

## 配置

ovn-controller从本地Open vSwitch的ovsdb-server实例中检索大部分配置信息。默认位置是本地Open vSwitch的“run”目录中的db.sock。可以通过以下列形式之一指定ovs-database参数来覆盖它：

   - **ssl**:ip:port

        指定的ip上的主机上指定的SSL端口，必须表示为IPv4或IPv6地址格式的IP地址（而不是DNS名称）。如果ip是一个IPv6地址，那么用方括号包住ip，例如：ssl:[::1]:6640。当指定ssl协议时--private-key, --certificate, 和--ca-cert三个参数不可省略。

    - **tcp**:ip:port

        连接到给定ip的TCP端口，其中ip可以是IPv4或IPv6地址。如果ip是一个IPv6地址，那么用方括号把ip包起来。例如：ssl:[::1]:6640

    - **unix**:file

        在POSIX上，连接到名为file的Unix域服务器套接字。

        在Windows上，连接到由路径文件中创建的文件表示的本地命名管道，以模拟Unix域套接字的行为。

    - **pssl**:port:ip

        在给定的SSL端口上监听连接。默认情况下，该连接没有绑定到特定的本地IP地址，它只侦听IPv4（而不是IPv6）地址，但可以只侦听给定的ip（IPv4或IPv6地址）的连接。如果ip是一个IPv6地址，然后用方括号括住ip。例如：pssl:6640:[::1]，当指定pssl协议时--private-key, --certificate, 和--ca-cert三个参数不可省略。
    
    - **ptcp**:port:ip

        在给定的TCP端口上监听连接。默认情况下，该连接没有绑定到特定的本地IP地址，它只侦听IPv4（而不是IPv6）地址，但可以只侦听给定的ip（IPv4或IPv6地址）的连接。如果ip是一个IPv6地址，则用方括号括住ip，例如：ptcp:6640:[::1]

    - **punix**:file

        在POSIX上，侦听Unix域服务器套接字名为file的连接。

        在Windows上，在本地命名管道上侦听。 在路径文件中创建一个文件来模仿Unix域套接字的行为。


**ovn-controller**假定它从本地OVS实例的Open_vSwitch表中的以下键获取配置信息：

   - **external_ids:system-id**

        Chassis表中使用的chassis名称

    - **external_ids:hostname**

        在Chassis表中使用的主机名

    - **external_ids:ovn-bridge**

        逻辑端口所连接的集成网桥，缺省值是br-int。如果ovn-controller启动时这个网桥不存在，将会使用ovn-architecture中建议的默认配置自动创建。

    - **external_ids:ovn-remote**

        连接到OVN数据库的配置，与上面记录的ovs-database具有相同的形式

    - **external_ids:ovn-remote-probe-interval**

        与OVN数据库连接的不活动探测间隔，以毫秒为单位。如果该值为零，则会禁用连接保持活动功能。

        如果该值不是零，那么它将被强制为至少1000毫秒的值。

    - **external_ids:ovn-encap-type**

        chassis应该用来连接到这个节点的封装类型。多个封装类型可以用逗号分隔的列表来指定。每个列出的封装类型将与ovn-encap-ip配对。

        用于连接hypervisor的支持的隧道类型是geneve和stt。网关可以使用geneve，vxlan或stt。

        由于vxlan中的元数据量有限，相对于其他隧道格式，连接网关的功能和性能将会降低。

    - **external_ids:ovn-encap-ip**

        chassis应使用external_ids指定的封装类型连接到此节点的IP地址，与ovn-encap-type配合使用。

    - **external_ids:ovn-bridge-mappings**

        将物理网络名称映射到提供到该网络的连接的本地ovs网桥的键值对的列表。将两个物理网络名称映射到两个ovs桥接的示例值是：physnet1：br-eth0，physnet2：br-eth1。

    - **external_ids:ovn-encap-csum**

        ovn-encap-csum表示可以以合理的性能发送和接收封装校验和。这意味着发送者将数据传输到这个chassis，他们应该使用校验和来保护OVN元数据。设置为true以启用或禁用，这取决于网络接口卡的功能，启用封装校验和可能会导致性能损失。在这种情况下，可以禁用封装校验和。

ovn-controller从本地OVS实例的Open_vSwitch数据库中读取以下值：

   - **datapath-type from Bridge表**

        该值从Bridge表的本地OVS集成网桥行中读取，并填充到OVN_Southbound数据库中的Chassis表的external_ids：datapath-type中。

    - **iface-types from Open_vSwitch表**

        该值从Bridge表的本地OVS集成网桥行中读取，并填充到OVN_Southbound数据库中的chassis表的external_ids：iface-types中。

    - **private_key, certificate, ca_cert,  and  bootstrap_ca_cert  from SSL表**

        这些值提供SSL配置，用于在通过external_ids：ovn-remote配置SSL连接类型时连接到OVN南行数据库服务器。 请注意，此SSL配置也可以通过命令行选项提供，如果两者都存在，则数据库中的配置优先。

##  OVS数据库的使用

ovn-controller在Open vSwitch数据库中使用多个external_ids键来跟踪端口和接口。 为了正确操作，用户不应该改变或清除这些键：

   - **Port表中的external_ids:ovn-chassis-id**

        此值的存在标识集成网桥内的隧道端口，由ovn-controller创建，以便到达远程chassis，它的值是远程chassis的chassis标识。

    - **Bridge表中的external_ids:ct-zone-*.**

        逻辑端口和网关路由器由ovn-controller分配一个连接跟踪区域用于有状态的服务。为了保持ovn-controller重新启动的状态，这些密钥存储在集成电桥的Bridge表中。该名称由ct-zone的前缀以及逻辑端口或网关路由器区域值组成，此键的值标识用于此端口的区域。

    - **Port表中的external_ids:ovn-localnet-port**

        此值的存在标识一个patch端口是由ovn-controller创建的，用来连接集成网桥和另一个网桥来实现localnet逻辑端口。有关更多信息，请参阅上面的external_ids：ovn-bridge-mappings。

        每个localnet逻辑端口被实现为一对patch端口，一个在集成网桥中，一个在不同网桥中，具有相同的external_id：ovn-localnet-port值。

    - **Port表中的external_ids:ovn-l2gateway-port**

        此值的存在标识一个patch端口是由ovn-controller创建的，用来连接集成网桥和另一个网桥来实现一个l2gateway逻辑端口。有关更多信息，请参阅上面的external_ids：ovn-bridge-mappings。

        每个l2gateway逻辑端口被实现为一对patch端口，一个在集成网桥中，一个在不同网桥中，具有相同的external_ids:ovn-l2gateway-port值。

    - **Port表中的external-ids:ovn-l3gateway-port**

        此值标识一个patch端口，由ovn-controller创建，用于实现一个l3网关逻辑端口。该patch端口与OVN逻辑patch端口类似，只是l3网关端口只能绑定到特定chassis。

    - **Port表中的external-ids:ovn-logical-patch-port**

        此值标识一个patch端口，由ovn-controller创建，用于在集成网桥内实现一个OVN逻辑patch端口。它的值是它实现的OVN逻辑patch端口的名称。

## 运行时管理命令

ovs-appctl可以发送命令到正在运行的ovn-controller进程。当前支持的命令如下所述：

   - **exit**

        使ovn-controller正常终止

    - **ct-zone-list**

        列出每个本地逻辑端口及其连接跟踪区域

    - **inject-pkt microflow**

        将microflow注入连接的Open vSwitch实例。microflow必须包含Open vSwitch实例上存在的入口逻辑端口（inport参数）。

        microflow参数描述了在ovn-sb中描述的OVN逻辑表达式语法中要转发的数据包，主要用于调试模拟。解析器可以理解先决条件; 例如，如果表达式引用了ip4.src，则不需要显式指明ip4或eth.type == 0x800。
