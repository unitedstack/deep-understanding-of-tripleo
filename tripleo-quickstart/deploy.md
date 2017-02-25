# TripleO-Quickstart部署

TripleO-Quickstart的部署要求必须有一台物理服务器，在quickstart中称作VIRTHOST，因为要建多台虚拟机，所以配置至少是16G内存，32G更好，对网络没有特殊要求，能通公网就行。本次测试所使用的物理服务器为64G内存，CPU型号为Intel\(R\) Xeon\(R\) CPU E5-2620 v3 @ 2.40GHz，是在Softlayer上申请的一台物理机。[Softlayer](http://www.softlayer.com/)在2013年被IBM收购，被整合到IBM的[Bluemix](https://console.ng.bluemix.net/)中，Softlayer除了提供多种云服务外，也是唯一一家提供裸机服务的公有云，使用体验非常不错。

除了有一台物理服务器外，还需要有一个客户端机器，能够ssh到物理服务器，操作系统需要是RedHat系的，在quickstart中称为localhost，需要将quickstart的程序放到localhost中，然后ssh到物理服务器上执行相应的ansible程序。

本次测试将会部署一个HA模式的TripleO环境，有3个控制节点和1个计算节点，控制节点同时充当网络节点，没有部署Ceph，存储使用本地存储。执行过程如下：

```
[root@localhost ~] export VIRTHOST='my_test_machine.example.com'
[root@localhost ~] wget https://raw.githubusercontent.com/openstack/tripleo-quickstart/master/quickstart.sh
[root@localhost ~] bash quickstart.sh --install-deps
[root@localhost ~] bash ./quickstart.sh --tags all --config ~/.quickstart/tripleo-quickstart/config/general_config/ha.yml $VIRTHOST
```

在quickstart中，基本上为每一个task都通过tag做了分类，可以通过--tags来选择执行某些task，上面使用all即执行所有的task，也就是要搭建一个完整的环境。使用--config可以指定部署模式，这里选择的是ha模式，此外还有多种模式可以选择。因为在安装过程中要去装各种包，而且要下载undercloud的镜像，国外的网络环境较好，会遇到比较少的坑。

等待一杯咖啡的时间，一个具备HA的OpenStack虚拟环境就部署好了，包含三个控制节点，一个计算节点。由于在OpenStack中最复杂最难懂的就是网络了，尤其是在虚拟环境中，网络拓扑要比物理环境还要复杂，而且是在一台物理服务器上安装一个具备完整功能的OpenStack环境，因此，下面重点介绍下用tripleo-quickstart部署出来的集群的网络拓扑，如下图：

![](/assets/tripleo-undercloud-topology.png)

