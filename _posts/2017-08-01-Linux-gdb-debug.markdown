---
layout: post
title:  "Linux GDB Debug"
date:   2017-08-01 13:00:05
categories: Linux
---
本文主要介绍如何使用gdb分析core文件。
二进制程序crash后，若系统有相关的配置，可以生成coredump文件，而core文件便是我们分析crash原因的重要线索。分析crash的backtrace就需要gdb工具。

### ulimit

1. ulimit -c     //可查看core文件的生成开关。若结果为0，则表示关闭了此功能，不会生成core文件。 
2. ulimit -c size   //可以限制core文件的大小（size的单位为kbyte），如果生成的信息超过此大小，将会被裁剪，如果生成被裁减的core文件，调试此core文件的时候，gdb也会提示错误.  
   ulimit -c unlimited   //core文件大小无限制  
   ulimit -c 0           //不生成core文件  

   注：这种设置仅对当前生效，如果想永久生效需将命令添加到/etc/profile
3. ulimit -a    //该命令将显示所有的用户定制，其中选项-a代表“all”。

### core文件配置

core文件存放路径在/proc/sys/kernel/core_pattern中配置

1. 通常我们将core文件生成在当前目录下：echo core > /proc/sys/kernel/core_pattern
2. echo "/corefile/core-%e-%p-%t" > core_pattern，可以将core文件统一生成到/corefile目录下，产生的文件名为core-命令名-pid-时间戳

   参数列表:

   * %p - insert pid into filename 添加pid
   * %u - insert current uid into filename 添加当前uid
   * %g - insert current gid into filename 添加当前gid
   * %s - insert signal that caused the coredump into the filename 添加导致产生core的信号
   * %t - insert UNIX time that the coredump occurred into filename 添加core文件生成时的unix时间
   * %h - insert hostname where the coredump happened into filename 添加主机名
   * %e - insert coredumping executable name into filename 添加命令名
3. /proc/sys/kernel/core_uses_pid可以控制core文件的文件名中是否添加pid作为扩展。文件内容为1，表示添加pid作为扩展名，生成的core文件格式为core.xxxx；为0则表示生成的core文件同一命名为core.   
   可通过以下命令修改此文件：echo "1" > /proc/sys/kernel/core_uses_pid

**注：/proc目录本身是动态加载的，每次系统重启都会重新加载，因此这种方法只能作为临时修改.**  
可以通过在/etc/sysctl.conf文件中，对sysctl变量kernel.core_pattern的设置。在sysctl.conf文件中添加下面两句话：

~~~
kernel.core_pattern = /var/core/core_%e_%p
kernel.core_uses_pid = 0
~~~

可以使用以下命令，使修改结果马上生效。

~~~
sysctl –p /etc/sysctl.conf
~~~

### gdb调试
* 已有core文件：
  * gdb 执行命令 core.pid
  * 进入gdb命令行后，执行bt，查看backtrace
* gdb直接调试：
  * gdb 执行命令   
  * 进入gdb命令行后，执行run ，程序运行
  * 出现core dump后，执行bt，查看backtrace
