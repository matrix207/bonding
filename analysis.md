###1.  目的

   本文档结合相关内核代码和对Linux 2.6.9内核中Bonding模块的三种主要工作模式的工作  
   原理和流程。在配置Bond模块时，除了资料[2]，本文档也有一定的参考价值。

###2. 内容

本文档包含下列内容：

* Bonding模块工作流程综述。（第3节）
* Bonding链路状态监控机制（mii模式、arp模式）描述。（第4节）
* Bonding模块三种主要工作模式：balance-rr、active- backup和broadcast相关代码分析。（第5节）
* Bonding模块关键数据结构和函数的代码分析。（第5节）

如果想了解bonding模块的原理和工作流程，请阅读3、4节，如果想进一步分析bonding模块的代码，请阅读5节。

###3. Bonding模块工作流程综述

   Bonding模块本质上是一个虚拟的网卡驱动（network device driver），只不过并没有  
   真实的物理网卡与之对应，而是由这个虚拟网卡去“管辖”一系列的真实的物理网卡，所以  
   它的代码结构和一般网卡驱动的代码结构非常类似，这是共性；除此之外，它还有自己的  
   一些特性功能，例如特别的链路状态监控机制，绑定/解除绑定等。

####3.1 物理网卡的活动状态和链路状态：

   在bonding模块中为每一个被绑定的物理网卡定义了两种活动状态和四种链路状态：注意，  
   这里的链路状态和实际网卡真实的链路状态（是否故障、是否有网线连接）没有直接的关系，  
   虽然bonding模块通过MII或者ARP侦测到实际网卡故障时也会改变自定义的链路状态值  
   （例如从BOND_LINK_UP切换到BOND_LINK_FAIL随后切换到 BOND_LINK_DOWN状态），但是  
   概念上应该把这两类链路状态区分开。在本文档随后的内容中，除非特别指出，“链路状态”  
   都指bonding模块自定义的链路状态。

   活动状态：

    * BOND_STATE_ACTIVE：处于该状态的网卡是潜在的发送数据包的候选者
    * BOND_STATE_BACKUP：处于该状态的网卡在选择发送数据的网卡时被排除

   链路状态：

    * BOND_LINK_UP：  上线状态（处于该状态的网卡是是潜在的发送数据包的候选者）
    * BOND_LINK_DOWN：故障状态
    * BOND_LINK_FAIL：网卡出现故障，向状态BOND_LINK_DOWN 切换中
    * BOND_LINK_BACK：网卡恢复，向状态BOND_LINK_UP切换中

   一个网卡必须活动状态为BOND_STATE_ACTIVE并且链路状态为 BOND_LINK_UP，才有可能作  
   为发送数据包的候选者，注意，这里所说的数据包并不包含ARP请求，在使用ARP链路状态  
   监控时，一个处于BOND_LINK_BACK状态的网卡也可能发送ARP请求。

   bonding模块的所有工作模式可以分为两类：多主型工作模式和主备型工作模式，balance-rr  
   和broadcast属于多主型工作模式而active-backup属于主备型工作模式。（balance-xor、  
   自适应传输负载均衡模式（balance-tlb）和自适应负载均衡模式（balance-alb）也属于  
   多主型工作模式，IEEE 802.3ad动态链路聚合模式（802.3ad）属于主备型工作模式，在本文档中不加以讨论）

   在多主型工作模式中，如果物理网卡不出现故障，所有的物理网卡都处于 BOND_STATE_ACTIVE  
   和BOND_LINK_UP的状态下，参与数据的收发，此时：如果工作在balance-rr 模式中轮流  
   向各个网卡发送数据，curr_active_slave字段（见5.1.3）指向前次发送数据（不包含  
   ARP请求）的物理网卡，该指针每次发送过数据后都会切换到下一个物理网卡；在broadcast  
   模式中向所有网卡发送数据，curr_active_slave字段除非网卡有故障发生不会切换。

   在主备型工作模式中，如果物理网卡不出现故障，只有一块网卡（活动网卡）处于   
   BOND_STATE_ACTIVE和BOND_LINK_UP的状态下，负责数据的收发，而其他网卡（后备网卡）  
   处于BOND_STATE_BACKUP 和BOND_LINK_DOWN状态下，此时curr_active_slave字段指向当前的活动网卡。

   如果工作在active-backup模式下，可以指定一个物理网卡作为主网卡（primitive interface）  
   ,如果主网卡可用，就把主网卡作为当前活动网卡，否则在其他的可用网卡中选取一块网  
   卡作为当前活动网卡，而一旦主网卡从故障中恢复，不管当前活动网卡是否故障都切换到  
   主网卡。在balance-tlb和balance-alb模式中也可以指定主网卡，但是其意义和active-backup模式中并不相同。

####3.2 数据收发流程

   如果一个物理网卡被虚拟网卡绑定，则代表该网卡的数据结构struct net_device中下列字段将发生变化：

    * flags字段（unsigned short）将包含IFF_SLAVE标志。
    * master字段（struct net_device *）将指向虚拟网卡。

   在主备型工作模式下，所有的非活动物理网卡的flags字段还将设置IFF_NOARP标志位表示对ARP请求不做出回应。

   而代表虚拟网卡的struct net_device数据结构的flags字段将包含IFF_MASTER标志。

   所有被绑定的物理网卡都将被设置相同的MAC地址和MTU值（和虚拟网卡也相同），这些值  
   将和第一块被绑定的物理网卡保持一致（“第一块网卡”并不是一个强制条件，这是由bonding  
   模块的启动流程造成的，我们也可以手工设置虚拟网卡的MAC地址和MTU值，这个设定同时  
   也将用于所有被绑定的物理网卡）。另外，所有被绑定的物理网卡没有IP地址，所以不参  
   与发送IP数据包的路由选择。

   在下面的三节中，只描述数据发送和接收过程中和bonding相关的一些特殊处理，关于Linux  
   内核的一般数据包收发流程请参考资料[3][4]，本文档不再赘述。

#####3.2.1 接收数据

   无论在何种模式下，只要物理网卡的实际链路状态正常，任何被绑定的物理网卡都可以接  
   收数据（虽然没有IP地址，但是仍然有MAC地址），即使是处于BOND_STATE_BACKUP和BOND_LINK_DOWN  
   状态时，这是由于BOND_STATE_BACKUP和BOND_LINK_DOWN是bonding模块自己定义的管理物  
   理网卡所用的状态，和内核的TCP/IP栈没有任何关系，bonding模块最多在主备模式下给  
   备用物理网卡设置IFF_NOARP标志，使它对ARP数据包不做出回应，仅此而已。

   收取数据包时，物理网卡驱动的中断处理函数把数据包放入接收队列中，随后软中断  
   NET_RX_SOFTIRQ的处理函数net_rx_action被调用，该函数将调用接收数据包的物理网卡网卡的poll函数。

   无论一个物理网卡是否支持NAPI，函数netif_receive_skb都将在某个阶段被调用。  
   （如果物理网卡不支持NAPI，内核使用函数process_backlog代替真实的poll调用，  
   而process_backlog调用netif_receive_skb）。

   在netif_receive_skb函数中将调用函数skb_bond，该函数本质上作如下操作：

   if(dev->master) skb->dev = dev->master;

   即把数据包skb的物理网卡字段替换为虚拟网卡，使得该数据包看起来像是从虚拟网卡接  
   收的一样，随后的处理和其他数据包没有任何差别，不再赘述。

#####3.2.2 发送数据

   发送数据包时，内核根据路由选择某一个虚拟网卡作为发送接口（注意被绑定的物理网卡  
   没有IP地址），最后调用该虚拟网卡的数据包传输接口net_device-> hard_start_xmit，  
   注意此时该数据包中的dev字段指向虚拟网卡。net_device-> hard_start_xmit接口根据  
   不同的工作模式指向不同的传输函数，但是无论是何种工作模式，最后bond_dev_queue_xmit  
   函数都将被调用（以一个选定的物理网卡作为参数调用）。

   bond_dev_queue_xmit函数将作如下操作：

   skb->dev = slave_dev;

   即替换skb的dev字段为选定的物理网卡。

   随后，bond_dev_queue_xmit将调用标准的内核接口dev_queue_xmit把数据包放入选定物理网卡的发送队列中。

#####3.2.3 ARP请求和回应

   既然被绑定物理网卡没有IP地址，那么如果接收到ARP请求之后，根据何IP地址决定是否产生应答？  
   如果产生应答，在应答中，源IP地址应该是什么？

   答案是：被绑定物理网卡接收到ARP请求后，由于函数arp_rcv在netif_receive_skb之后被调用，  
   而skb->dev经过netif_receive_skb的处理将指向虚拟网卡，所以是否产生应答由该物理网卡所  
   属的虚拟网卡的IP地址决定（当然前提是物理网卡没有设置IFF_NOARP标志），并且最终的ARP  
   回应将包含虚拟网卡的IP地址（细节请参考arp_rcv和arp_process函数）。

###4. 链路状态监控

链路状态监控有如下两个作用：

* 根据被绑定物理网卡的实际链路状态（是否故障、网线是否连接）更新bonding模块自定义  
  的物理网卡链路状态和活动状态。
* 在主备模式下，根据自定义的物理网卡链路状态切换活动状态和当前的活动网卡。

Bonging模块支持两种模式的链路状态监控：通过MII ioctl调用直接进行或是通过发送ARP  
请求间接进行。前者通过调用物理网卡驱动程序的MII ioctl接口直接获得物理网卡是否  
故障的信息，后者通过向预定义的一组IP地址发送ARP请求，并观察被监控物理网卡是否  
有收到数据包（可能是ARP回答或者是一般数据包）间接进行。

这两种链路状态监控模式不能并存。参数arp_interval和miimon表示以毫秒为单位的检测  
间隔，在加载bonding模块时如果指定了非0的arp_interval参数并且miimon参数等于0，  
则使用ARP链路状态监控；如果指定了非0 miimon参数则使用MII链路状态监控  
（强制arp_interval = 0从而忽略arp_interval参数）。如果arp_interval和miimon都  
等于0则使用参数值为100的MII链路状态监控（强制miimon等于100 ms）。

如果使用ARP链路状态监控，还*必须*指定 arp_ip_target参数，该参数设定ARP监控时发送ARP  
请求的目标IP列表，各个IP之间使用逗号分隔。

如果使用MII链路状态监控，还*可以*指定参数updelay和downdelay作为从BOND_LINK_DOWN到  
BOND_LINK_UP或者从BOND_LINK_UP 到BOND_LINK_DOWN切换的时间间隔，这两个参数默认是0值。  
（在ARP链路状态监控中这两个参数没有用）

####4.1 MII链路状态监控

MII链路状态监控可以用下列流程图表示

您的浏览器可能不支持显示此图像。

初始时，如果虚拟网卡工作在多主型工作模式下，则所有物理网卡的链路状态为BOND_LINK_UP，  
并且活动状态处于BOND_STATE_ACTIVE，IFF_NOARP标志都没有设置；否则所有物理网卡的链路  
状态为 BOND_LINK_UP，但是只有当前活动网卡的活动状态处于BOND_STATE_ACTIVE并且没有  
设置IFF_NOARP 标志，而其余网卡的活动状态为BOND_STATE_BACKUP并且IFF_NOARP标志被设置。

MII检测机制每miimon毫秒检测一遍所有被绑定物理网卡的状态。

1. 在某时刻，如果通过MII调用侦测到某一个物理网卡发生故障，则该物理网卡的链路状态立  
   即被设置为BOND_LINK_FAILED。
2. 如果在downdelay毫秒内物理网卡恢复正常，则重新把网卡的链路状态设置为BOND_LINK_UP。
3. 如果在downdelay毫秒内物理网卡始终没有恢复正常，则该物理网卡的链路状态被设置为  
   BOND_LINK_DOWN。如果虚拟网卡工作于主备型工作模式下，则活动状态被设置为BOND_STATE_BACKUP  
   同时设置物理网卡的IFF_NOARP标志，并且如果出故障的是当前活动网卡，则通过一个重  
   选择过程选择新的活动网卡（一般是第一块可用物理网卡）。
4. 如果一个链路状态为BOND_LINK_DOWN的物理网卡在MII检测过程中恢复正常，则立即把网卡  
   的链路状态设置为BOND_LINK_BACK。
5. 如果在updelay毫秒内物理网卡又发生故障，就把链路状态重新设置为BOND_LINK_DOWN。
6. 如果在updelay毫秒内物理网卡始终保持可用状态，就把链路状态重新设置为BOND_LINK_UP。  
   如果虚拟网卡工作于主备型工作模式下，则同时设置活动状态为BOND_STATE_ACTIVE并且  
   清除物理网卡的IFF_NOARP标志，并且如果是主网卡恢复到BOND_STATE_ACTIVE状态，则会  
   把当前活动网卡切换到主网卡。

####4.2 ARP链路状态监控

#####4.2.1 active-backup工作模式下的ARP链路状态监控

该模式下的ARP链路状态监控可以分为两个阶段：

1. 如果当前活动网卡（curr_active_slave不为NULL）存在，则以间隔 arp_interval毫秒从当  
   前活动网卡向arp_targets表示的各个IP地址发送ARP请求，如果当前活动网卡在过去的  
   2*arp_interval毫秒内没有数据包发送*和*接收并且已经作为活动网卡至少 2*arp_interval  
   毫秒，则把当前活动网卡的链路状态设置为BOND_LINK_DOWN并且试图在链路状态为BOND_LINK_UP  
   或者BOND_LINK_BACK的网卡中选取一个网卡作为当前活动网卡。如果有这样一个网卡存在，  
   则原来的活动网卡的活动状态被设置为BOND_STATE_BACKUP，并且IFF_NOARP标志被设置，  
   新的活动网卡链路状态被设置为BOND_STATE_UP, 活动状态被设置为BOND_STATE_ACTIVE，  
   IFF_NOARP标志被清除。

2. 如果上述过程没有选出新的活动网卡（正常情况下active-backup 模式下除当前活动网卡  
   外所有网卡的链路状态都是BOND_LINK_DOWN，所以可能没有链路状态为BOND_LINK_UP或者  
   BOND_LINK_BACK的后备网卡），则开始一个下述的选取活动网卡的过程：

从第一块可用（即IFF_UP标志被设置，netif_running(dev)和netif_carrier_ok(dev)都返回非0值）  
物理网卡开始，向arp_targets表示的各个IP地址发送ARP请求，然后观察所有的物理网卡，  
如果有物理网卡在arp_interval毫秒内有数据发送*和*接收，就把它设置为当前活动网卡，结束这个选取过程。否则换下一个可用物理网卡，重复这个过程。
注意即使物理网卡被设置IFF_NOARP标志，仍旧可以收到ARP应答数据包。

#####4.2.2 其他工作模式下的ARP链路状态监控

   虚拟网卡每arp_interval遍历一遍所有被绑定物理网卡，如果在网卡在过去的2 * arp_interval毫秒内没有任何数据的发送*或者*接收，就把网卡的链路状态设置为 BOND_LINK_DOWN，活动状态设置为BOND_STATE_BACKUP，如果在过去的arp_interval毫秒有数据包发送*和*接收，则把网卡的链路状态设置为BOND_LINK_UP，活动状态设置为 BOND_STATE_ACTIVE。

   在遍历过程中，对每一个可用的物理网卡（IFF_UP标志被设置，netif_running(dev)和netif_carrier_ok(dev)都返回非0值），都试图从该网卡向arp_targets表示的各个IP地址发送ARP请求，保证其他的被绑定的物理网卡可以收到ARP应答包。

###5. 代码分析

####5.1 关键数据结构

* 1.struct bond_params

文件：driver/net/bonding/bonding.h

该结构是全局结构（每系统一个），对应于加载bonding模块时传入的各个参数

	主要成员：
	名称  类型  含义
	mode  int  Bonding模块工作模式
			   BOND_MODE_ROUNDROBIN     balance-rr模式
			   BOND_MODE_ACTIVEBACKUP   active-backup模式
			   BOND_MODE_XOR            balance-xor模式
			   BOND_MODE_BROADCAST      broadcast模式
			   BOND_MODE_8023AD         IEEE 802.3ad动态链路聚合模式
			   BOND_MODE_TLB            自适应传输负载均衡模式
			   BOND_MODE_ALB            自适应负载均衡模式
	miimon  int  使用MII链路状态监控时的时间间隔（ms）
	arp_interval  int  使用arp链路状态监控时的时间间隔（ms）
	use_carrier  int  使用MII链路状态监控时是否使用更新的carrier调用
	updelay  int  使用MII链路状态监控时从BACK状态切换到UP状态的时延(ms)
	downdelay  int  使用MII链路状态监控时从FAIL状态切换到DOWN状态的时延(ms)
	primary  char[]  用来在active-backup、balance-tlb和balance-alb模式中指定主网卡
	arp_targets  u32[]  在ARP链路状态监控中将向这些IP地址发送ARP请求。

* 2.struct slave

文件：driver/net/bonding/Bonding.h

每一个被管辖的物理网卡对应一个该数据结构的实例

	主要成员：
	名称  类型  含义
	dev  struct net_device*  指向被绑定的物理网卡
	next，prev  struct slave *  所有的slave数据结构通过这两个指针双向链接到一起形成*循环*链表
	delay  s16  用于保存MII链路状态监控和ARP链路状态监控的时延值。
	jiffies  u32  用于active-backup模式下的ARP状态监控
	link  s8  表示对应网卡的链路状态，取下列四个值之一：
				BOND_LINK_UP     上线状态
				BOND_LINK_DOWN   故障状态
				BOND_LINK_FAIL   网卡出现故障，状态BOND_LINK_DOWN切换中
				BOND_LINK_BACK   网卡恢复，状态BOND_LINK_UP切换中
	state  s8  表示对应网卡活动状态，取下列两个值之一：
				BOND_STATE_ACTIVE            活动状态
				BOND_STATE_BACKUP            后备状态
	original_flags  u32  保存被绑定物理网卡原来的flags
	perm_hwaddr  u8[]  保存被绑定物理网卡原来的MAC地址
	ad_info  struct ad_slave_info  记录IEEE 802.3ad动态链路聚合模式下的“每网卡”相关状态信息
	tlb_info  struct tlb_slave_info  记录自适应传输负载均衡模式下的“每网卡”相关状态信息
	link_failure_count  u32  网卡从BOND_LINK_FAIL切换到BOND_LINK_DOWN的次数
	speed  u16  记录网卡的传输速度（10M/100M/1000G）
	duplex  u8  网卡工作模式（全双工？）

* 3.struct bonding

文件：driver/net/bonding/Bonding.h

每一个虚拟网卡对应一个该数据结构的实例。

	主要成员：
	名称  类型  含义
	dev  struct net_device*  指向虚拟网卡（例如bond0、bond1等等）
	first_slave  struct   slave *  指向被绑定的第一个物理网卡对应的slave结构。
	curr_active_slave  struct   slave *  指向当前活动的网卡对应的slave结构，在不同的工作模式下有不同的含义。
	current_arp_slave  struct   slave *  用于ARP状态监控（只用于bond_activebackup_arp_mon）
	primary_slave  struct   slave *  如果使用BOND_MODE_ACTIVEBACKUP、BOND_MODE_TLB或者BOND_MODE_ALB模式，则指向主物理网卡对应的slave结构（primary_slave）
	slave_cnt  s32  虚拟网卡所管辖的物理网络接口的个数
	lock  rwlock_t  每一个虚拟网卡管辖一系列物理网卡，每一个物理网卡对应一个slave结构，所有的slave被放在一个链表中，这个读写锁用来保护该链表。
	curr_slave_lock  rwlock_t  用来保护curr_active_slave的读写锁。
	mii_timer  struct   timer_list  用于MII链路状态监控的定时器
	arp_timer  struct   timer_list  用于ARP链路状态监控的定时器
	kill_timers  s8  如果该标志置位，所有的计时器超时后就不再重新设置，从而可以被安全删除
	bond_list  struct   list_head  通过该结构，所有的bonding数据结构被连接为双向链表，链表头是全局变量bond_dev_list。

####5.2 关键函数

本节描述了bonding模块关键函数的操作流程，这些函数是基本的原子模块，其他没有被列举的函数仅仅是对这些函数的简单包装。

#####5.2.1 模块初始化/释放

* 1.初始化

bonding_init

原型： static int __init bonding_init(void)

   bonding_init作为bonging模块的初始化函数，在模块被加载时被调用。它主要做如下工作：

   1. 调用函数bond_check_params解析传入模块的参数并检查其合法性，结果放入数据结构params中。其中params是一个类型为bond_params的全局变量。
   2. 如果内核支持proc文件系统，调用bond_create_proc_dir在/proc/net下创建目录/proc/net/bonding。
   3. 如果传入参数指定了bond设备的个数（通过参数max_bonds），则通过下列步骤创建max_bonds个bond设备（从bond0到bondN）
         1. 调用alloc_netdev和dev_alloc_name创建网络设备，指定每一个设备的私有数据是一个bonding结构（dev->priv）。
         2. 为每一个新创建的虚拟网络设备调用bond_init函数。
         3. 调用register_netdevice注册这个新创建的网络设备。
   4. 调用register_netdevice_notifier，注册函数bond_netdev_notifier为网络事件处理函数。

bond_init

原型： static int __init bond_init(struct net_device *bond_dev, struct bond_params *params)

该函数对每一个新创建的虚拟网络设备调用一次。它主要做下列工作。

   1. 取出虚拟网络设备bond_dev的私有数据块，用bond指向它（struct bonding *bond）。
   2. 初始化两个读写锁bond->lock和bond->curr_slave_lock。
   3. 把bond->first_slave、bond->curr_active_slave、bond->current_arp_slave、bond->primary_slave全部置为NULL。bond->dev指向属主网络设备bond_dev。
   4. 设置bond_dev的一系列通用接口函数，例如open、close、get_stats、do_ioctl、set_multicast_list、change_mtu和set_mac_address。
   5. 根据不同的工作模式，设置bond_dev的通用接口函数hard_start_xmit 指向不同的目的函数。例如如果工作模式是BOND_MODE_ROUNDROBIN，则 hard_start_xmit指向函数bond_xmit_roundrobin。
   6. 设置bond_dev->tx_queue_len为0，表示发送队列大小没有限制。
   7. 设置bond_dev->flags为IFF_MASTER|IFF_MULTICAST表示该设备支持Muticase并且是一个流量均衡组中的master（被它管辖的其他物理网卡将被设置IFF_SLAVE标志）。
   8. 其他和VLAN相关的标志设置和初始化。
   9. 如果内核支持proc文件系统，调用bond_create_proc_entry在目录/proc/net/bonding下创建对应的proc文件。
  10. 调用list_add_tail把该bonding数据结构添加到bond_dev_list中。

* 2.释放

bonding_exit

原型： static void __exit bonding_exit(void)

该函数在模块被卸载的时候被调用，它主要做如下工作：

1. 调用unregister_netdevice_notifier注销网络事件处理函数。
2. 调用bond_free_all注销所有形如bondN的虚拟网络接口。bond_free_all遍历bond_dev_list链表，并且对其中的每一个类型为 struct bonding*的数据结构bond做如下操作：
1. 调用unregister_netdevice注销bond->dev
2. 调用bond_deinit(bond->dev)
3. 调用bond_destroy_proc_dir删除/proc/net/bonding目录

bond_deinit

原型： static inline void bond_deinit(struct net_device *bond_dev)

该函数对每一个虚拟网卡的实例调用一次，它主要做如下操作：

1. 调用list_del把虚拟网卡对应的bonding数据结构从bond_dev_list链表中摘除。
2. 调用bond_remove_proc_entry删除/proc/net/bonding目录中的对应文件。

#####5.2.2 物理网卡的绑定/解除绑定

   在下面的讨论中，假定ifenslave使用新的ABI接口，即：

    * 在绑定物理网卡时，如果虚拟的网卡还没有MAC地址，则ifenslave通过IOCTL把该虚拟网卡的MAC地址设置为该物理网卡的MAC地址（保证bond_enslave被调用时虚拟网卡已经有了MAC地址）。
    * 如果被绑定网卡处于UP状态，则ifenslave首先把它设置为DOWN状态（保证bond_enslave被调用时被绑定物理网卡处于DOWN状态）。

   如果使用旧版本的ABI接口，则虚拟的网卡的MAC地址由bonding模块在bond_enslave函数中自行设置，并且被绑定网卡在bond_enslave被调用时可能处于UP状态，需要由bond_enslave函数自行处理。

* 1.绑定

bond_enslave

原型： static int bond_enslave(struct net_device *bond_dev, struct net_device *slave_dev)

该函数在试图把一个物理网卡绑定到一个虚拟的网卡时被调用，其中bond_dev表示虚拟的网卡，slave_dev表示真实的物理网卡。该函数主要做如下操作：

1. 取出bond_dev的私有数据，用bond指向它（struct bonding *）。
2. 一系列的合法性检查，包括：
	1. bond_dev的flags是否已经设置了IFF_UP（虚拟的网卡必须已经处于UP状态）
	2. slave_dev的flags是否没有设置IFF_SLAVE（防止同一个物理网卡被绑定两次）
	3. bond_dev的flags如果设置了NETIF_F_VLAN_CHALLENGED，则bond->vlan_list不能为空。
	4. slave_dev->flags是否没有设置IFF_UP（物理网卡应该处于DOWN状态）
	5. slave_dev->set_mac_address不能为NULL（物理网卡应该支持设置MAC地址）
3. 调用kmalloc分配一个新的slave结构。
4. 把slave_dev->flags保存在slave->original_flags中。
5. 把slave_dev原有的MAC地址保存在slave-> perm_hwaddr中。
6. 设置slave_dev新的MAC地址为虚拟网卡的MAC地址。
7. 调用netdev_set_master设置slave_dev，该函数主要作如下操作：
	1. 设置slave_dev->flags的IFF_SLAVE标志。
	2. 设置slave_dev->master指向虚拟网卡bond_dev。
8. 设置slave->dev指向slave_dev。
9. 如果bond_dev工作在模式BOND_MODE_TLB或者BOND_MODE_ALB，对slave调用bond_alb_init_slave函数。
10. 维护和Multicast以及VLAN相关的一系列数据结构。
11. 调用bond_attach_slave把slave加入bond的链表（通过维护bond-> first_slave和slave结构的next，prev指针）
12. 把slave的delay和link_failure_count都清零。
13. 监测slave_dev的链路状态：
	1. 如果使用MII链路，并且bond_check_dev_link返回BMSR_LSTATUS（表示链路正常），或者不使用MII链路监控，则根据updelay是否为0把slave->link设置为BOND_LINK_BACK或者BOND_LINK_UP。
	2. 如果使用MII链路，并且bond_check_dev_link返回非BMSR_LSTATUS值，则设置slave->link为BOND_LINK_DOWN。
14. 调用bond_update_speed_duplex更新slave_dev的链路速率，如果失败则设置slave_dev的链路速率为100M全双工。
15. 如果虚拟网卡工作在BOND_MODE_ACTIVEBACKUP、BOND_MODE_TLB或者BOND_MODE_ALB模式下，并且slave_dev是用户指定的主网卡，则设置bond->primary_slave为slave_dev。
16. 设置bond->curr_active_slave和slave的活动状态，维护VLAN和Multicast相关数据结构：
	1. 如果虚拟网卡工作在BOND_MODE_ACTIVEBACKUP：如果bond->curr_active_slave没有被设置或者bond->curr_active_slave不响应ARP（设置了IFF_NOARP标志），并且slave_dev不处于BOND_LINK_DOWN状态，则设置slave_dev为活动网卡（设置BOND_STATE_ACTIVE标志，清除IFF_NOARP标志），否则设置slave_dev为后备网卡（设置BOND_STATE_BACKUP标志，设置IFF_NOARP标志）。
	2. 如果虚拟网卡工作在BOND_MODE_ROUNDROBIN或者 BOND_MODE_BROADCAST：直接设置slave_dev的活动状态为BOND_STATE_ACTIVE（在BOND_MODE_ROUNDROBIN和BOND_MODE_BROADCAST 模式下，IFF_NOARP标志始终被清除），如果没有设置bond->curr_active_slave，则设置bond->curr_active_slave指向slave。
	3. 其他工作模式：（还没有加以分析）

* 2.解除绑定

bond_release

原型 static int bond_release(struct net_device *bond_dev, struct net_device *slave_dev)

该函数在试图解除一个物理网卡的绑定状态时被调用，其中bond_dev表示虚拟的网卡，slave_dev表示真实的物理网卡。该函数主要做如下操作：

 1. 取出bond_dev的私有数据，用bond指向它（struct bonding *）。
 2. 寻找对应的slave结构。
 3. 一系列的合法性检查，包括：
	 1. slave_dev->flags是否设置了IFF_SLAVE。
	 2. slave结构是否存在。
	 3. slave_dev原来的MAC地址是否和bond_dev相同，如果相同给出警告（防止MAC冲突）
 4. 如果虚拟网卡工作在BOND_MODE_8023AD模式，调用bond_3ad_unbind_slave
 5. 调用bond_detach_slave把slave从bond的链表中摘除（通过维护bond-> first_slave和slave结构的next，prev指针）。
 6. 如果slave_dev是虚拟网卡以前的主物理网卡，则设置bond->primary_slave为NULL。
 7. 如果slave_dev是虚拟网卡以前的活动网卡，则设置bond->active_slave为NULL（通过调用bond_change_active_slave函数）。
 8. 如果虚拟网卡工作在模式BOND_MODE_TLB或者BOND_MODE_ALB则调用bond_alb_deinit_slave。
 9. 如果slave_dev是虚拟网卡以前的活动网卡，则调用bond_select_active_slave寻找一个新的活动网卡。
10. 如果虚拟网卡再也没有管辖的物理网卡，清除虚拟网卡的MAC地址（如果新调用ifenslave绑定物理网卡，则重新设置这个MAC地址）。
11. 维护VLAN和Multicast相关的数据结构。
12. 调用netdev_set_master解除master和slave的绑定关系并且调用dev_close关闭slave_dev。
13. 恢复slave_dev的MAC地址（根据slave->perm_hwaddr）和flags（根据slave->original_flags）。
14. 调用kfree释放slave结构。

#####5.2.3 网卡驱动通用接口（interface service routines）

既然bonding模块本质上是一个虚拟网卡的驱动模块，所以必须提供一组所有网卡驱动模块都遵守的通用接口函数。

* 1.open/close

bond_open（net_device->open接口）

原型： static int bond_open(struct net_device *bond_dev)

该函数在对应的虚拟网卡被打开时调用（即使用ifup/ifconfig工具启动网卡的时候），主要做如下操作（只分析三种主要模式）：

1. 设置bond->kill_timers为0。
2. 如果使用MII链路状态监控：
	1. 初始化mii_timer。
	2. 设置超时时间mii_timer->expires为当前jiffies+1（立即调用bond_mii_monitor函数）
	3. 设置bond_mii_monitor为定时器的超时处理函数。
3. 如果使用ARP链路状态监控：
	1. 初始化arp_timer。
	2. 设置超时时间arp_timer->expires为当前jiffies+1（立即调用定时器的超时处理函数）
	3. 如果工作在BOND_MODE_ACTIVEBACKUP，设置bond_activebackup_arp_mon为超时处理函数。
	4. 如果工作在其他模式，设置bond_loadbalance_arp_mon为超时处理函数。

 
bond_close（net_device->stop接口）

原型： static int bond_close(struct net_device *bond_dev)

该函数在对应的虚拟网卡被关闭时调用（即使用ifdown/ifconfig工具关闭网卡的时候），主要做如下操作（只分析三种主要模式）：

1. 调用bond_mc_list_destroy维护Multicast相关数据结构。
2. 设置bond->kill_timers为1，所有的计时器超时后就不再重新设置，从而可以被安全删除。
3. 删除所有的定时器，包括mii_timer和arp_timer。
4. 调用bond_release_all释放所有被绑定的物理网卡，本质上该函数只是遍历slave链表并且对每一个元素调用bond_release。
5. 如果虚拟网卡工作在BOND_MODE_TLB或者BOND_MODE_ALB模式下，调用bond_alb_deinitialize。

* 2.ioctl接口

bond_do_ioctl（net_device->do_ioctl 接口）

原型： static int bond_do_ioctl(struct net_device *bond_dev, struct ifreq *ifr, int cmd)

该函数是虚拟网卡的IOCTRL接口，仅仅根据不同的IOCTRL命令调用其他函数执行相应的功能，
所以不再列出操作流程而仅仅列举出这些被调用的函数和相应的功能：

    1.链路状态设置和查询（bond_ethtool_ioctl或者if_mii）
    2.Bonding模块状态查询（bond_info_query）
    3.被绑定的物理网卡状态查询（bond_slave_info_query）
    4.物理网卡的绑定和解除绑定（bond_enslave/bond_release）
    5.虚拟网卡的MAC地址设置（bond_sethwaddr）
    6.切换当前活动的物理网卡（bond_ioctl_change_active）

* 3.统计值查询

bond_get_stats（net_device-> get_stats 接口）

原型： static struct net_device_stats *bond_get_stats(struct net_device *bond_dev)

该函数枚举所有被管辖的物理网卡，并且对每一个物理网卡调用get_stats接口，然后把对应的统计值加起来并作为最终的返回值，这些统计值包括。
名称  含义
rx_packets  接收包总数
rx_bytes  接收字节总数
rx_errors  接收过程中错误数据包数
rx_dropped  接受过程中丢弃包数
tx_packets  发送包总数
tx_bytes  发送字节总数
tx_errors  发送过程中错误数据包数
tx_dropped  发送过程中丢弃包数
multicast  Multicast数据包总数
collisions  MAC地址冲突次数
rx_length_errors  接收数据包长度错误总数
rx_over_errors  ring buff溢出次数
rx_crc_errors  接收数据包CRC校验错误总数
rx_frame_errors  接收数据包frame对齐错误总数
rx_fifo_errors  接收队列溢出次数
rx_missed_errors  接收时丢失的包数（仅仅对某些媒体有效）
tx_aborted_errors  发送取消次数（例如发送超时）
tx_carrier_errors  链路错误总数
tx_fifo_errors  发送队列溢出次数
tx_heartbeat_errors  心跳信号丢失（仅仅对某些媒体有效）
tx_window_errors  接收窗口错误（不明，需要进一步确认）
 

    * bond_set_multicast_list（net_device-> set_multicast_list 接口）

原型：

static void bond_set_multicast_list(struct net_device *bond_dev)

   该函数设置和Multicast和混杂模式相关的一组数据结构，由于三种主要工作模式并不过多地涉及这个函数，所以本文档不给出详细的说明。 

    * bond_change_mtu（net_device-> change_mtu 接口）

   原型：

static int bond_change_mtu(struct net_device *bond_dev, int new_mtu)

   该函数把被虚拟网卡的MTU和被它管辖的所有物理网卡的MTU设置为同一值，主要做如下操作：

         1. 枚举所有被管辖的物理网卡，对每一个物理网卡调用change_mtu设置新的MTU值，如果物理网卡没有change_mtu接口函数，则直接设置slave->dev->mtu等于new_mtu。
         2. 设置bond_dev->mtu的值等于new_mtu。

 

    * bond_set_mac_address（net_device-> set_mac_address 接口）

      原型：

      static int bond_set_mac_address(struct net_device *bond_dev, void *addr)

   该函数设置虚拟网卡的MAC地址和被管辖的物理网卡的MAC地址为同一值，主要做如下操作：

         1. 枚举所有被管辖的物理网卡，对每一个物理网卡调用set_mac_address设置新的MAC地址，如果物理网卡没有set_mac_address函数，则错误返回。
         2. 设置bond_dev->dev_addr的值等于新的MAC地址。

                     4. 数据包传输（接收/发送）

     Bonding模块仅仅负责把发送数据包按照特定的工作模式转给被管辖的物理网卡发送，而每一个物理网卡负责自己的数据包接收，即虚拟网卡不管理各个物理网卡的数据接收过程，它能做的仅仅是设置它们的IFF_NOARP标志，使某一个物理网卡对ARP请求不做出回应。

     在模块初始化时， bond_init函数根据工作模式把net_device-> hard_start_xmit接口设置为不同的函数，对于 BOND_MODE_ROUNDROBIN、BOND_MODE_ACTIVEBACKUP和BOND_MODE_BROADCAST 模式，该接口分别被设置为下列三个函数之一 

    * bond_xmit_roundrobin（net_device-> hard_start_xmit 接口）

原型：

static int bond_xmit_roundrobin(struct sk_buff *skb, struct net_device *bond_dev)

该函数用来在BOND_MODE_ROUNDROBIN模式中发送数据包，主要做如下操作：

         1. 合法性检查，包括：

         1. 检查bond_dev ->flags中IFF_UP标志是否设置
         2. netif_running(bond_dev)是否返回非0值
         3. 对应的虚拟网卡是否至少有一个管辖的物理网卡

         2. 从bond->curr_active_slave开始遍历slave链表，找到第一个链路状态为BOND_LINK_UP，活动状态为BOND_STATE_ACTIVE的物理网卡并且调用bond_dev_queue_xmit向这个物理网卡发送数据，然后设置bond->curr_active_slave为slave链表中的下一个物理网卡。

         3. 如果没有找到这样的网卡或者bond_dev_queue_xmit返回非0值，则调用dev_kfree_skb丢弃数据包。

 

    * bond_xmit_activebackup（net_device-> hard_start_xmit 接口）

原型：

static int bond_xmit_activebackup(struct sk_buff *skb, struct net_device *bond_dev)

该函数用来在BOND_MODE_ACTIVEBACKUP模式中发送数据包，主要做如下操作：

         1. 如果是试图发送ARP请求，则把全局变量my_ip设置为ARP请求中的发送方IP地址（skb->data+以太网头长度+ARP头长度+6），这个全局变量在ARP链路状态监控中被使用。
         2. 合法性检查，包括：

         1. 检查bond_dev ->flags中IFF_UP标志是否设置
         2. netif_running(bond_dev)是否返回非0值
         3. 对应的虚拟网卡是否至少有一个管辖的物理网卡

         3. 如果bond->curr_active_slave不为空，则调用bond_dev_queue_xmit向这个物理网卡发送数据。

         4. 否则，调用dev_kfree_skb丢弃数据包。

 

    * bond_xmit_broadcast（net_device-> hard_start_xmit接口）

原型：

static int bond_xmit_broadcast(struct sk_buff *skb, struct net_device *bond_dev)

该函数用来在BOND_MODE_BROADCAST模式中发送数据包，主要做如下操作：

         1. 合法性检查，包括：

         1. 检查bond_dev ->flags中IFF_UP标志是否设置
         2. netif_running(bond_dev)是否返回非0值
         3. 对应的虚拟网卡是否至少有一个管辖的物理网卡

         2. 从bond->curr_active_slave开始遍历slave链表，找到所有状态为BOND_LINK_UP，活动状态为BOND_STATE_ACTIVE的物理网卡，包括bond->curr_active_slave，调用bond_dev_queue_xmit向这些物理网卡发送数据（其中需要通过skb_clone复制skb结构）。

         3. 如果发送失败，调用dev_kfree_skb丢弃数据包

 

    * bond_dev_queue_xmit

      原型：

      int bond_dev_queue_xmit(struct bonding *bond, struct sk_buff *skb, struct net_device *slave_dev)

   该函数被bond_xmit_roundrobin，bond_xmit_activebackup 和bond_xmit_broadcast 调用，向实际的物理网卡发送数据包，主要做如下操作：

         1. 设置skb->dev为slave_dev（在此之前skb->dev指向虚拟网卡，现在指向真实的物理网卡）
         2. 维护和VLAN相关的数据结构。
         3. 调用dev_queue_xmit发送数据包。

 

    * dev_queue_xmit

      原型：

      该函数不是bonding模块的一部分而是内核的一个标准接口，为了清楚起见也把它列出来，请参考net/core/dev.c文件。

      int dev_queue_xmit(struct sk_buff *skb)

         1. 如果底层的物理网卡不支持Scatter/Gather IO，而skb包含了分片（注意不是IP分片，而是和DMA相关的一个概念，见skbbuff.h），则调用__skb_linearize合并分片。
         2. 如果底层的设备不支持计算校验和，则计算一系列校验和。
         3. 如果底层的设备有发送队列（qdisc），则把数据包放入发送队列中，退出。
         4. 如果底层的设备没有发送队列（例如loopback或者其他没有真实物理网卡对应的设备，bonding模块自然也算一个），则直接调用底层设备的hard_start_xmit发送数据包。
         5. 如果发送失败，调用dev_kfree_skb丢弃数据包

#####5.2.4. 链路状态监控

* 1.MII链路状态监控

bond_mii_monitor

原型： static void bond_mii_monitor(struct net_device *bond_dev)

   如果使用MII链路状态监控，则该函数被周期调用以检测每一个被绑定物理网卡的链路状态, 主要做如下操作：

         1. 计算局部变量delta_in_ticks = (bond->params.miimon * HZ) / 1000，即miimon参数的jiffies表示。
         2. 如果kill_timers被设置，直接退出。
         3. 如果没有任何物理网卡被绑定，重新设置定时器，退出。
         4. 根据bond_check_dev_link的结果，按照第5节描述的MII链路状态监控模型设置网卡的链路状态。
         5. 如果原来物理网卡的链路状态为BOND_LINK_FAIL，而 bond_check_dev_link返回非BMSR_LSTATUS值，则除了把链路状态设置为BOND_LINK_DOWN之外，还做如下操作：

         1. 如果虚拟网卡工作在模式BOND_MODE_8023AD，调用bond_3ad_handle_link_change
         2. 如果虚拟网卡工作在模式BOND_MODE_TLB或者BOND_MODE_ALB模式下，调用bond_alb_handle_link_change。
         3. 如果当前被检查的slave不是curr_active_slave，设置标志do_failover表明可能会发生slave切换。

         6. 如果原来物理网卡的链路状态为BOND_LINK_BACK而bond_check_dev_link 返回BMSR_LSTATUS，则除了把链路状态设置为BOND_LINK_UP之外，还做如下操作：

         1. 如果虚拟网卡工作在模式BOND_MODE_8023AD或者被监测网卡不是primary_slave，则设置物理网卡的活动状态为BOND_STATE_BACKUP
         2. 如果虚拟网卡*不是*工作在模式BOND_MODE_ACTIVEBACKUP，则设置物理网卡的活动状态为BOND_STATE_ACTIVE
         3. 如果虚拟网卡工作在模式BOND_MODE_8023AD，调用bond_3ad_handle_link_change
         4. 如果虚拟网卡工作在模式BOND_MODE_TLB或者BOND_MODE_ALB模式下，调用bond_alb_handle_link_change。
         5. 如果当前被检查的slave不是curr_active_slave，设置标志do_failover表明可能会发生slave切换。

         7. 调用bond_update_speed_duplex更新物理网卡的速率。

         8. 如果do_failover被设置，调用bond_select_active_slave。
         9. 设置定时器的超时值为jiffies+delta_in_ticks。 					

bond_check_dev_link

原型： static int bond_check_dev_link(struct bonding *bond, struct net_device *slave_dev, int reporting)

该函数调用MII/ETHTOOL IOCTL或者使用netif_carrier_ok()检查链路是否正常工作（如果用  
户指定了use_carrier）参数，如果该函数返回BMSR_LSTATUS表明链路是正常的，否则表示链  
路故障（例如掉网线等等）。

* 2.ARP链路状态监控

bond_loadbalance_arp_mon

原型： static void bond_loadbalance_arp_mon(struct net_device *bond_dev)

如果虚拟网卡工作在*非*BOND_MODE_ACTIVEBACKUP 模式下，而用户指定了使用ARP状态监控，  
则周期性地对每一个被绑定物理网卡调用该函数，注意该函数不使用 downdelay和updelay参数。

由于*非*BOND_MODE_ACTIVEBACKUP模式下所有的被绑定网卡都是处于活动状态（BOND_STATE_ACTIVE），  
所以该函数的功能是轮流从每一个被绑定物理网卡发送ARP请求，并且在一段时间间隔内是否  
有数据包接收，如果没有就设置被检查物理网卡的链路状态为BOND_LINK_DOWN，活动状态设置  
为BOND_STATE_BACKUP 表示不参与发送数据（但是只要IFF_UP被设置、netif_running和  
netif_carrier_ok都返回非0（真）值，即本地网卡检查通过，仍然周期性地发送ARP请求出去），  
请参考5.2节中的描述。

   该函数主要做如下操作：

         1. 计算局部变量delta_in_ticks = (bond->params.arp_interval * HZ) / 1000，即arp_interval参数的jiffies表示。
         2. 如果kill_timers被设置，直接退出。
         3. 如果没有任何物理网卡被绑定，重新设置定时器，退出。
         4. 枚举所有被绑定的物理网卡，做如下操作：
               1. 假如物理网卡的链路状态不是BOND_LINK_UP并且在delta_in_ticks时间间隔内发送过*并且*接受过数据包，则把链路状态设置为BOND_LINK_UP，活动状态设置为BOND_STATE_ACTIVE，并且如果curr_active_slave为空则设置do_failover局部变量。
               2. 假如物理网卡的链路状态是BOND_LINK_UP并且在2*delta_in_ticks时间间隔内没有发送过或者没有接受过数据包，则把链路状态设置为BOND_LINK_DOWN，活动状态设置为BOND_STATE_BACKUP，如果当前slave是curr_active_slave则设置do_failover局部变量。
               3. 如果dev->flags中IFF_UP被设置，netif_running和netif_carrier_ok都返回非0（真）值，则尝试调用bond_arp_send_all从该网卡发送ARP请求（参考bond_arp_send_all的描述）。
         5. 如果do_failover被设置，调用bond_select_active_slave。
         6. 设置定时器的超时值为jiffies+delta_in_ticks。

bond_activebackup_arp_mon

原型： static void bond_activebackup_arp_mon(struct net_device *bond_dev)

如果虚拟网卡工作在BOND_MODE_ACTIVEBACKUP模式下，而用户指定了使用ARP状态监控，则周期性地对每一个被绑定物理网卡调用该函数。

   该函数主要做如下操作：

   1. 计算局部变量delta_in_ticks = (bond->params.arp_interval * HZ) / 1000，即arp_interval参数的jiffies表示。
   2. 如果kill_timers被设置，直接退出。
   3. 如果没有任何物理网卡被绑定，重新设置定时器，退出。
   4. 枚举所有被绑定的物理网卡，做如下操作：
               1. 如果物理网卡在时间间隔delta_in_ticks内接收过数据包，就把网卡的链路状态设置为BOND_LINK_UP（网卡的活动状态保持不变），设置curr_active_slave为NULL。
               2. 如果物理网卡在时间间隔3*delta_in_ticks内没有接收过数据包并且该网卡不是curr_active_slave，就把网卡的链路状态设置为BOND_LINK_DOWN并且调用bond_set_slave_inactive_flags设置网卡的活动状态为BOND_STATE_BACKUP，并且设置IFF_NOARP标志位，设置curr_active_slave为NULL。
   5. 检查curr_active_slave，如果curr_active_slave不为NULL：
         1. 如果curr_active_slave在2*delta_in_ticks内没有发送也没有接收过数据包，就把curr_active_slave的链路状态设置为BOND_LINK_DOWN并且调用bond_select_active_slave寻找一个新的网卡作为新的curr_active_slave，设置current_arp_slave为curr_active_slave。
         2. 如果使用bond->primary_slave并且bond->primary_slave的链路状态是BOND_LINK_UP且bond->primary_slave不是curr_active_slave，就把bond->primary_slave作为新的curr_active_slave。
         3. 否则设置current_arp_slave为NULL；
         4. 调用bond_arp_send_all通过curr_active_slave发送ARP请求。
   6. 检查curr_active_slave，如果curr_active_slave为NULL，则从current_arp_slave 开始或者从first_slave开始选出一个网卡并且把链路状态设置为BOND_LINK_BACK的作为 curr_active_slave的候选者（包存在current_arp_slave中），在下一次lbond_activebackup_arp_mon 被调用的时候将把这个网卡设置为curr_active_slave。
   7. 设置定时器的超时值为jiffies+delta_in_ticks。

* 3.slave切换

bond_find_best_slave

原型： static struct slave *bond_find_best_slave(struct bonding *bond)

该函数从被绑定网卡中选出最佳者作为curr_active_slave的候选，主要做如下操作：

   1. 如果没有物理网卡被绑定，返回NULL。
   2. 如果没有设置primary_slave或者primary_slave不可用，从first_slave开始，否则从primary_slave开始遍历被绑定物理网卡列表，如果有网卡的链路状态为BOND_LINK_UP，则返回这个物理网卡。如果没有链路状态为BOND_LINK_UP的网卡，返回处于BOND_LINK_BACK状态最久者（delay值最小）。

bond_change_active_slave

原型： static void bond_change_active_slave(struct bonding *bond, struct slave *new_active)

该函数切换new_active为新的curr_active_slave，主要做如下操作：

   1. 如果curr_active_slave和new_active相同，不做任何操作。
   2. 如果new_active的链路状态是BOND_LINK_BACK，把链路状态设为BOND_LINK_UP。
   3. 如果当前工作在BOND_MODE_ACTIVEBACKUP状态，把curr_active_slave的活动状态设置为BOND_STATE_BACKUP，并且设置IFF_NOARP标志位；把new_active的活动状态设置为BOND_STATE_ACTIVE，清除IFF_NOARP标志位。
   4. 设置curr_active_slave为new_active。

###6. 参考
* [1]《Linux 多网卡绑定/负载均衡调研报告》
* [2]《Linux Ethernet Bonding Driver mini-howto》/src/net/Documentation/networking/bonding.txt
* [3]《The Linux® Networking Architecture: Design and Implementation of Network Protocols in the Linux Kernel》Klaus Wehrle
* [4]《Understanding Linux Network_Internals》Christian Benvenuti 
