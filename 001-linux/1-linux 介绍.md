Linux 内核是由芬兰人林纳斯·托瓦兹（Linus Torvalds）在赫尔辛基大学上学时出于个人爱好而编写的(因为当时Unix太贵了)，免费使用和自由传播，类 Unix 操作系统，多用户、多任务、支持多线程和多 CPU，能运行主要的 UNIX 工具软件、应用程序和网络协议，继承了 Unix 以网络为核心的设计思想，是一个性能稳定的多用户网络操作系统。目前工作中主要使用的版本是RedHat、Ubuntu、Centos，延伸阅读请搜索 Linux发行版、GNU软件、GPL协议、Linus Torvalds、开源协议。

下面这张图概述了 计算机 -> Unix -> C语言 -> GNU/Linux、FreeBSD -> Linux、Next ->  Linux发行版、MacOS 的大概历史，欢迎观看本节末尾视频了解。

![](http://processon.com/chart_image/5fcf1a277d9c0830e8e237cb.png)

主要的linux发行版:

1.1 REHL - Red Hat Enterprise Linux 是红帽公司的商业版本  
![](https://pic1.zhimg.com/80/v2-b1f94ee89fc1b0921ba5ae78afc39f94_1440w.jpg)

1.2 CentOS - Community Enterprise Operating System 社区操作系统，基于 RHEL 重新编译的开源免费版  
![](https://pic3.zhimg.com/80/v2-5f4671f5ace7440d91a99cfefffd2dc6_1440w.jpg)

1.3 Ubuntu - 基于 Debian GNU/Linux  由 Canonical Ltd 打造  
![](https://pic2.zhimg.com/80/v2-6e9189f3b1e71f958a5120c7d5f25dcd_1440w.jpg)

1.4 SUSE - 德国 SuSE Linux AG 公司的发行版  
![](https://pic1.zhimg.com/80/v2-fe1d8107d0e8ae0f4e01b2c3efa4aeb4_1440w.jpg)

1.5 Debian - 致力于创建自由操作的合作组织及其作品  
![](https://pic4.zhimg.com/80/v2-4315498e27c1208cb80a10a3a543d5fb_1440w.jpg)

1.6 OEL - Oracle Enterprise Linux 对 Oracle 的软件支持较好  
![](https://pic1.zhimg.com/80/v2-f035ef84ef6a2db3580bbca5ee03af24_1440w.jpg)

学习的时候推荐使用的系统是 Ubuntu (https://ubuntu.com/download) 或者 Centos (https://www.centos.org/download/)，可以下载对应的 ISO 文件安装到虚拟机中，学过 Docker 的可以直接使用  docker-machine 命令。
