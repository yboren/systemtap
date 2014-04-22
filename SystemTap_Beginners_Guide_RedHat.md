引言
========

SystemTap是一个跟踪和探测内核运行的工具，它可以让用户清楚地了解操作系统（尤其是内核）运行的细节。SystemTap也能提供像工具netstat，ps，top，iostat那样的类似输出信息，但是它拥有更强的过滤信息和分析信息的功能，被赋予了更强大的能力。

文档目标
--------

SystemTap提供一些底层的支持，可以监控正在运行的内核的细节情况，从而帮助系统管理员或者开发者发现内核的bug或者性能问题。

如果没有SystemTap，要监控正在运行的内核的某些细节的话，需要修改内核代码，重新编译内核，安装内核，并重新启动机器，非常繁琐。有了SystemTap，用户只需编写SystemTap脚本，然后运行，就可以达到相同的目的。

然而，SystemTap一开始就是给那些拥有丰富内核知识的用户而设计的，这样就使得SystemTap对于系统管理员或对内核了解有限的开发者而言，并没有太大用处。并且，现有的SystemTap文档同样只适合那些内核知识和经验丰富的用户。这样就使得依靠现在的文档学习SystemTap比较困难。

因此有了这篇SystemTap初学者指南，主要期望达到以下目标：

*给用户介绍SystemTap，让他们熟悉架构，并提供相关版本的内核安装说明。
*提供预先写好的SystemTap脚本，这些脚本涵盖到了监测系统各个组件，并分析这些脚本的功能以及介绍如何编写这些脚本。

SystemTap的能力
--------------

*灵活性：SystemTap可以让用户只需要开发简单的脚本，就可以做到查看和监控内核空间的各种事件，包括内核函数，系统调用等。从这点上来说，SystemTap其实已经超越了工具的范畴，它更像一个系统，来帮助用户开发内核相关的分析监控工具。
*易用性：正如前面提到的，SystemTap中允许用户编写脚本就能做到探测内核空间的事件，而不必做那些繁琐的修改内核代码，编译，安装，并重新启动内核的事情。

使用SystemTap
============

安装SystemTap
------------

SystemTap需要安装以下RPM：

*systemtap
*systemtap-runtime
如果系统包含yum工具，可以这样 yum install systemtap systemtap-runtime。

接下来安装SystemTap依赖的内核的RPM。

SystemTap依赖的内核包包括以-devel,，-debuginfo和 -debuginfo-common-arch结尾的RPM。例如对于vanilla内核，需要安装的包如下：

*kernel-debuginfo
*kernel-debuginfo-common-arch
*kernel-devel
如果内核支持PAE，需要安装kernel-PAE-debuginfo，kernel-PAE-debuginfo-common-arch和kernel-PAE-devel。

初始测试
-------

如果SystemTap依赖的内核已经启用，那么就可以做一些测试确保SystemTap已经安装正确。否则就要使用正确的内核重新启动机器。

测试可以运行命令：stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'只要检测到有一个VFS的读操作，SystemTap就打印一个read performed，然后正常退出。如果SystemTap安装成功，你应该得到类似下面的输出：

>Pass 1: parsed user script and 45 library script(s) in 340usr/0sys/358real ms.

>Pass 2: analyzed script: 1 probe(s), 1 function(s), 0 embed(s), 0 global(s) in 290usr/260sys/568real ms.

>Pass 3: translated to C into "/tmp/stapiArgLX/stap_e5886fa50499994e6a87aacdc43cd392_399.c" in 490usr/430sys/938real ms.

>Pass 4: compiled C into "stap_e5886fa50499994e6a87aacdc43cd392_399.ko" in 3310usr/430sys/3714real ms.

>Pass 5: starting run.

>read performed

>Pass 5: run completed in 10usr/40sys/73real ms.

最后三行的输出（即以Pass 5开始的行 ）表明，SystemTap能够成功地创建内核探针，运行探测，检测到被探测的事件（这里是VFS的读取动作），并执行一个事先定义的处理程序（打印文本，然后将其关闭，并且没有任何错误）。

在其它电脑上运行
---------------

运行一个SystemTap脚本的时候，实际上是生成了一个内核模块，然后加载这个内核模块运行，从而得到内核相关的数据。

一般来说，SystemTap脚本只能运行在安装了SystemTap的机器上，也就是说，如果你需要在十台机器上运行，就需要在这些机器上都安装上SystemTap，这样的方式有很大的局限性，比如企业中可能会禁止安装有调试信息的RPM包，这样就自然无法安装SystemTap了。

要解决这样的问题，就需要用SystemTap脚本生成内核模块，然后把内核模块直接拿到其它机器上运行。这样的话有下面的优点

*一个主机可以安装适合各种机器的调试信息包
*其它机器只需要安装systemtap-runtime，就可以运行主机生成的SystemTap的内核模块

运行SystemTap脚本
----------------

可以使用命令stap运行SystemTap脚本，stap命令可以从标准输入或者文件读入脚本运行。

运行stap和staprun需要系统的root权限。然而，并不是所有的用户都有root权限的，有时一个非特权用户可能需要到他们的机器上运行SystemTap模块。

为了让普通用户在没有root权限的情况下能够运行SystemTap模块，可以将它们添加到stapdev用户组，该组的成员可以使用stap运行SystemTap脚本或staprun运行SystemTap模块。

这里说的运行stap包括编译SystemTap脚本成内核模块，然后加载到内核。这个操作需要root权限，stapdev组的成员拥有这个权限，相当于拥有root权限，因此只把可以信任的用户加为stapdev组的成员。

另外还有个stapusr组，该组的成员只能使用staprun运行SystemTap模块。而且模块必须位于这个目录/lib/modules/kernel_version/systemtap/（注意，此目录必须只能由root用户拥有，而且必须只能由root用户可写）。

了解SystemTap的工作原理
=======================

编写SystemTap脚本可以实现多种多样的功能，比如提取和过滤数据，分析诊断复杂的问题等等。

SystemTap脚本有两个关键概念：事件和事件处理程序。当SystemTap脚本运行的时候，SystemTap监控特定事件的发生，当事件发生时，linux内核就会执行事先定义好的事件处理程序，执行完后再恢复内核的正常运行。

常见的事件有这么种：函数的进入和退出，定时器超时，会话结束等。而事件处理程序就是一系列脚本语句组成的代码块，在事件发生时执行。事件处理程序通常要干的事情包括从事件上下文中提取数据，存储在内部变量，做一些统计计算，打印处理结果。

SystemTap架构
-------------

当运行一个SystemTap脚本的时候，实际上是开始了一个SystemTap会话，这个会话包括以下过程：

1.SystemTap检查现有的tapset库，用从库中找到的定义替换脚本中的调用。
1.把SystemTap脚本翻译成C语言，并用编译器把C代码编译成一个内核模块。
1.SystemTap加载生成的内核模块，然后激活脚本中用到的探针。
1.监控的事件发生时，SystemTap脚本中的相应处理程序就会运行。
1.SystemTap会话终止时，探针被禁用，内核模块卸载。

用stap命令就可以完成上面整个过程，这个工具是SystemTap最主要的前端工具。

SystemTap脚本
------------

SystemTap脚本是整个SystemTap会话的基础，SystemTap脚本决定SystemTap会话过程需要收集什么信息，如何收集这些信息。

就程序开发而言，使用事件和事件处理程序，和在程序中加入调试用的打印语句类似，可以帮助事后查看程序的运行情况。

SystemTap脚本可以在无需重新编译程序代码的情况下插入探测语句，具有更高的灵活性。探测语句监测到特定的事件发生，相应的事件处理程序就运行记录相关的数据，并把数据用指定的格式打印出来。

脚本格式
-------

SystemTap脚本使用扩展名.stp，一般包含类似这样的探测语句：

>probe event {statements}



