# TripleO-Quickstart部署

TripleO-Quickstart的部署要求必须有一台物理服务器，在quickstart中称作VIRTHOST，因为要建多台虚拟机，所以配置至少是16G内存，32G更好，对网络没有特殊要求，能通公网就行。本次测试所使用的物理服务器为64G内存，CPU型号为Intel\(R\) Xeon\(R\) CPU E5-2620 v3 @ 2.40GHz，是在Softlayer上申请的一台物理机。[Softlayer](http://www.softlayer.com/)在2013年被IBM收购，被整合到IBM的[Bluemix](https://console.ng.bluemix.net/)中，Softlayer除了提供多种云服务外，也是唯一一家提供裸机服务的公有云，使用体验非常不错。

除了有一台物理服务器外，还需要有一个客户端机器，能够ssh到物理服务器，操作系统需要是RedHat系的，在quickstart中称为localhost，需要将quickstart的程序放到localhost中，然后ssh到物理服务器上执行相应的ansible程序。

本次测试将会部署一个HA模式的TripleO环境，有3个控制节点和1个计算节点，控制节点同时充当网络节点，quickstart目前默认部署Newton版本OpenStack，没有部署Ceph，存储使用本地存储。执行过程如下：

```
[root@localhost ~] export VIRTHOST='my_test_machine.example.com'
[root@localhost ~] wget https://raw.githubusercontent.com/openstack/tripleo-quickstart/master/quickstart.sh
[root@localhost ~] bash quickstart.sh --install-deps
[root@localhost ~] bash ./quickstart.sh --tags all --config ~/.quickstart/tripleo-quickstart/config/general_config/ha.yml $VIRTHOST
```

在quickstart中，基本上为每一个task都通过tag做了分类，可以通过--tags来选择执行某些task，上面使用all即执行所有的task，也就是要搭建一个完整的环境。使用--config可以指定部署模式，这里选择的是ha模式，此外还有多种模式可以选择，该配置文件还定义了一些其他参数，比如虚拟机的配置，是否跑tempest等。因为在安装过程中要去装各种包，而且要下载undercloud的镜像，国外的网络环境较好，会遇到比较少的坑。

等待一杯咖啡的时间，一个具备HA的OpenStack虚拟环境就部署好了，包含三个控制节点，一个计算节点。

由于在OpenStack中最复杂最难懂的就是网络了，尤其是在虚拟环境中，网络拓扑要比物理环境还要复杂，而且是在一台物理服务器上安装一个具备完整功能的OpenStack环境，因此，下面重点介绍下用tripleo-quickstart部署出来的集群的网络拓扑，如下图：

![](/assets/tripleo-undercloud-topology.png)

在物理服务器上，通过libvirt network建立了两个linux bridge，分别为brext和brovc，brext用来桥接undercloud，brovc用来桥接overcloud节点，undercloud虚拟机有两个网卡，分别桥接在brext和brovc上。brext提供了访问公网的能力，brovc将多个overcloud和undercloud节点连接在一起。

Undercloud是一个单机版的OpenStack，安装了Nova, Neutron, Ironic, Heat等组件，Ironic通过使用pxe_ssh driver来管理overcloud节点，Neutron为overcloud节点提供网络环境，Heat则在TripleO中被用来编排，创建整个overcloud的stack。默认的TripleO会在undercloud中创建下面几个网络：

```
[stack@undercloud ~]$ neutron net-list
+--------------------------------------+--------------+------------------------------------------------------+
| id                                   | name         | subnets                                              |
+--------------------------------------+--------------+------------------------------------------------------+
| 0a5f9e61-7f2a-4d1b-9b44-4f82a1412ef4 | internal_api | 0fe319dd-ec16-47c5-b039-707373b7875c 172.16.2.0/24   |
| 305526f7-d0bf-49b5-947a-d25778836cba | tenant       | a71b5b1c-5f52-4efd-af21-705192bf2a70 172.16.0.0/24   |
| 9e0930f3-8180-4fa3-a955-73aca0134795 | storage_mgmt | e596d585-7c12-471e-89de-eea48cabc0df 172.16.3.0/24   |
| be6f07e5-e212-4e9e-bc7b-0edd4eb11b13 | ctlplane     | ad8a6a4d-c5a3-4c44-a17d-1d1a9341f33b 192.168.24.0/24 |
| f303022b-907e-4ce6-a2c0-c1ce7fabe9ac | external     | 4a47bc1b-a7f2-45a4-bd40-2f3032198b34 10.0.0.0/24     |
| fcdff6b1-43d2-4787-b458-368b48b97a73 | storage      | 6de1e1b3-4972-49b1-ae19-20e163af010c 172.16.1.0/24   |
+--------------------------------------+--------------+------------------------------------------------------+
```
这几个网络都是Flat模式的网络，internal_api在overcloud中被用来作为内部API交互的网络，tenant是SDN网络，storage_mgmt是存储管理网络，ctlplane是管理网络，即ssh网络，external是外部网络，storage是存储网。

在上面的网络中，只有ctlplane的子网开启了DHCP功能，为overcloud的管理网络分配IP，因此在undercloud中有一个DHCP的namespace，通过veth tap桥接在br-int ovs网桥上。

external网络是用来开放API和面板的，跟internal_api对应，在overcloud中，API和面板分别绑定在了internal_api和external网络上，在undercloud的br-ctlplane ovs网桥上，创建了一个vlan10的port，并且在undercloud中加上了相应的路由信息，这样在undercloud中就可以直接通过这个网络访问overcloud中的API了：

```
[stack@undercloud ~]$ ip r
default via 192.168.23.1 dev eth0
10.0.0.0/24 dev vlan10  proto kernel  scope link  src 10.0.0.1
```

在overcloud中，通过在br-ex ovs网桥上绑定多个带tag的port，并且每个port分配了IP，来模拟多个网络，分别对应undercloud中创建的网络。在br-ex中的每个port都带着vlan tag，模拟交换机的access口，通过不同的vlan互相隔离。

控制节点和计算节点之间的SDN网络，即tenant网络，在上面图中使用黄色标注，即vlan50，通过vxlan建立隧道，使用vlan50上的ip作为对端IP。

其他就是标准的Neutron网络了。

