Linux内核bonding模块代码分析
============================

####源码信息
1. 代码来源: linux-2.6.32.63/drivers/net/bonding 


####优缺点
* 不足
  - Bond更改了网口的驱动，其网口不能被用作其他用途。
  - Bond默认只能做网口MII监测不能做链路监测（链路是指本机到网关的路径），也就是  
    只能监测网口是否连接(网口是否亮)；当然Bond也支持ARP协议的链路监测，但是ARP链  
    路监测在一些场景下，太消耗资源，得不偿失。

####参考链接
* [Bonding模块代码及主要工作模式分析1](http://bbs.csdn.net/topics/340055267)
* [Bonding模块代码及主要工作模式分析2](http://bbs.csdn.net/topics/340055270)
* [Bonding模块代码及主要工作模式分析3](http://bbs.csdn.net/topics/340055274)
* [Bonding模块代码及主要工作模式分析4](http://bbs.csdn.net/topics/340055280)
* [Bonding模块工作流程综述](http://bbs.csdn.net/topics/340144549)
* [Bonding模块主要工作模式相关代码分析一](http://bbs.csdn.net/topics/340144635)
* [Bonding模块主要工作模式相关代码分析二](http://bbs.csdn.net/topics/340144662)
* [bonding的源代码分析](http://www.docin.com/p-311297097.html)
* [Linux Bond的原理及其不足](http://www.tektea.com/archives/1969.html)
