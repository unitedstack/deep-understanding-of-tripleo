# TripleO-Quickstart解析

TripeleO-Quickstart是使用Ansible Playbook进行编排的，抽象出了一系列role，role中包含了各种具体的task，然后通过playbook对指定host进行编排，Ansible程序写的非常好，是一个很好的典范，这里对整个流程做下解析，方便加深对其的掌握，理解其原理。下图为执行quickstart过程中一些主要的步骤：  
![](/assets/tripleo-quickstart-analysis.png)

整个过程大体可以分为4个阶段，上图中分别用4种颜色标识：

* 第一个阶段，即浅红色部分，主要工作是清理上一次跑quickstart留下的环境，其中的non\_root\_user是在裸机上创建的一个用户，默认为stack，具有sudo权限，之后在裸机上执行的所有操作都是以这个用户的身份完成的。
* 第二个阶段，即浅黄色部分，是环境的准备阶段，因为是要在一台裸机上装虚拟机测试TripleO，所以要安装kvm，libvirt等包，而且要创建虚拟机使用的volume pool和libvirt network，下载镜像，然后定义undercloud和overcloud虚拟机，并且启动undercloud虚拟机等待下一个阶段使用。需要注意的是，这里仅仅下载了undercloud的镜像，因为undercloud镜像中已经包含了overcloud镜像，所以不需要额外下载。
* 第三个阶段，即浅绿色部分，是部署阶段，首先部署undercloud，然后部署overcloud，并且生成相应的rc文件，方便后面使用。因为本身tripleo本身就用了OpenStack中的很多组件，尤其是heat，已经进行了非常高度的抽象，因此部署步骤就很简单，quickstart将部署的命令都封装在了相应的脚本里，使用时，从ansible的模板生成，填上相应参数，传到undercloud的机器里执行。
* 第四个阶段，即浅蓝色部分，是验证阶段，验证OverCloud部署是否正常，验证HA是否正常，跑tempest进行功能验证等等。

这四个阶段，前两个阶段的步骤基本上都是在物理机上执行的，属于环境准备阶段，搭建起了TripleO的基本环境，这部分功能是在tripleo-quickstart中实现的，后两个阶段基本上都是在undercloud上执行的，是实际的部署验证阶段，这部分功能是在tripleo-quickstart-extras中实现的。

