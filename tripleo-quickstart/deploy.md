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

### 附录

* undercloud br-ctlplane流表

 ```
[stack@undercloud ~]$ sudo ovs-ofctl dump-flows br-ctlplane
NXST_FLOW reply (xid=0x4):
 cookie=0xa382b4ab161c4f5a, duration=287114.883s, table=0, n_packets=640, n_bytes=63449, idle_age=65534, hard_age=65534, priority=4,in_port=2,dl_vlan=1 actions=strip_vlan,NORMAL
 cookie=0xa382b4ab161c4f5a, duration=287154.063s, table=0, n_packets=3, n_bytes=258, idle_age=65534, hard_age=65534, priority=2,in_port=2 actions=drop
 cookie=0xa382b4ab161c4f5a, duration=287154.285s, table=0, n_packets=9441009, n_bytes=29995798300, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
 ```
* undercloud br-int流表

 ```
 [stack@undercloud ~]$ sudo ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0xbca7bd29346f9fa9, duration=287162.954s, table=0, n_packets=581299, n_bytes=31344498, idle_age=0, hard_age=65534, priority=3,in_port=1,vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:1,NORMAL
 cookie=0xbca7bd29346f9fa9, duration=287202.139s, table=0, n_packets=94718, n_bytes=7492743, idle_age=707, hard_age=65534, priority=2,in_port=1 actions=drop
 cookie=0xbca7bd29346f9fa9, duration=287202.890s, table=0, n_packets=643, n_bytes=63707, idle_age=65534, hard_age=65534, priority=0 actions=NORMAL
 cookie=0xbca7bd29346f9fa9, duration=287202.892s, table=23, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xbca7bd29346f9fa9, duration=287202.889s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 ``` 
* controll_0 br-tun流表

 ```
 [root@overcloud-controller-1 ~]# ovs-ofctl dump-flows br-tun
NXST_FLOW reply (xid=0x4):
 cookie=0x8513f3cebfb2fa84, duration=276881.519s, table=0, n_packets=583649, n_bytes=31994786, idle_age=0, hard_age=65534, priority=1,in_port=1 actions=resubmit(,2)
 cookie=0x8513f3cebfb2fa84, duration=276869.341s, table=0, n_packets=9, n_bytes=670, idle_age=65534, hard_age=65534, priority=1,in_port=2 actions=resubmit(,4)
 cookie=0x8513f3cebfb2fa84, duration=276869.338s, table=0, n_packets=4, n_bytes=280, idle_age=65534, hard_age=65534, priority=1,in_port=4 actions=resubmit(,4)
 cookie=0x8513f3cebfb2fa84, duration=276869.335s, table=0, n_packets=40902, n_bytes=3189452, idle_age=3, hard_age=65534, priority=1,in_port=3 actions=resubmit(,4)
 cookie=0x8513f3cebfb2fa84, duration=276881.518s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x8513f3cebfb2fa84, duration=276881.516s, table=2, n_packets=40074, n_bytes=2719568, idle_age=3, hard_age=65534, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x8513f3cebfb2fa84, duration=276881.515s, table=2, n_packets=543575, n_bytes=29275218, idle_age=0, hard_age=65534, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
 cookie=0x8513f3cebfb2fa84, duration=276881.514s, table=3, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x8513f3cebfb2fa84, duration=239847.949s, table=4, n_packets=40889, n_bytes=3188008, idle_age=3, hard_age=65534, priority=1,tun_id=0x5c actions=mod_vlan_vid:4,resubmit(,10)
 cookie=0x8513f3cebfb2fa84, duration=276881.513s, table=4, n_packets=6, n_bytes=460, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x8513f3cebfb2fa84, duration=276881.401s, table=6, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x8513f3cebfb2fa84, duration=276881.399s, table=10, n_packets=40909, n_bytes=3189942, idle_age=3, hard_age=65534, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0x8513f3cebfb2fa84,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:OXM_OF_IN_PORT[]),output:1
 cookie=0x8513f3cebfb2fa84, duration=35533.393s, table=20, n_packets=17409, n_bytes=1251681, hard_timeout=300, idle_age=3, hard_age=2, priority=1,vlan_tci=0x0004/0x0fff,dl_dst=fa:16:3e:6b:d6:95 actions=load:0->NXM_OF_VLAN_TCI[],load:0x5c->NXM_NX_TUN_ID[],output:3
 cookie=0x8513f3cebfb2fa84, duration=276881.398s, table=20, n_packets=53, n_bytes=3850, idle_age=65534, hard_age=65534, priority=0 actions=resubmit(,22)
 cookie=0x8513f3cebfb2fa84, duration=239847.950s, table=22, n_packets=30, n_bytes=1316, idle_age=65534, hard_age=65534, priority=1,dl_vlan=4 actions=strip_vlan,load:0x5c->NXM_NX_TUN_ID[],output:3,output:4,output:2
 cookie=0x8513f3cebfb2fa84, duration=276881.397s, table=22, n_packets=543596, n_bytes=29277612, idle_age=0, hard_age=65534, priority=0 actions=drop
 ```
* contorll_0 br-int流表

 ```
 [root@overcloud-controller-1 ~]# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0xbb0786bfe2f662c0, duration=265938.161s, table=0, n_packets=577089, n_bytes=31597998, idle_age=0, hard_age=65534, priority=3,in_port=1,vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:3,NORMAL
 cookie=0xbb0786bfe2f662c0, duration=276928.381s, table=0, n_packets=106165, n_bytes=7751821, idle_age=885, hard_age=65534, priority=2,in_port=1 actions=drop
 cookie=0xbb0786bfe2f662c0, duration=276928.545s, table=0, n_packets=120741, n_bytes=9002846, idle_age=4, hard_age=65534, priority=0 actions=NORMAL
 cookie=0xbb0786bfe2f662c0, duration=276928.547s, table=23, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xbb0786bfe2f662c0, duration=276928.544s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 ```
* controll_0 br-ex流表

 ```
 [root@overcloud-controller-1 ~]# ovs-ofctl dump-flows br-ex
NXST_FLOW reply (xid=0x4):
 cookie=0xafa8d606ada81d76, duration=265988.920s, table=0, n_packets=39286, n_bytes=3071764, idle_age=5, hard_age=65534, priority=4,in_port=7,dl_vlan=3 actions=strip_vlan,NORMAL
 cookie=0xafa8d606ada81d76, duration=276979.138s, table=0, n_packets=1265, n_bytes=66286, idle_age=21267, hard_age=65534, priority=2,in_port=7 actions=drop
 cookie=0xafa8d606ada81d76, duration=276979.143s, table=0, n_packets=489065150, n_bytes=98446193899, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
 ```
* compute_0 br-tun流表

 ```
 [root@overcloud-novacompute-0 ~]# ovs-ofctl dump-flows br-tun
NXST_FLOW reply (xid=0x4):
 cookie=0x85eb64bc3e0f6ac1, duration=277743.277s, table=0, n_packets=46825, n_bytes=3485282, idle_age=1, hard_age=65534, priority=1,in_port=1 actions=resubmit(,2)
 cookie=0x85eb64bc3e0f6ac1, duration=277059.640s, table=0, n_packets=142, n_bytes=23696, idle_age=21319, hard_age=65534, priority=1,in_port=3 actions=resubmit(,4)
 cookie=0x85eb64bc3e0f6ac1, duration=277023.273s, table=0, n_packets=40094, n_bytes=2719688, idle_age=1, hard_age=65534, priority=1,in_port=4 actions=resubmit(,4)
 cookie=0x85eb64bc3e0f6ac1, duration=233719.067s, table=0, n_packets=62, n_bytes=5192, idle_age=65534, hard_age=65534, priority=1,in_port=2 actions=resubmit(,4)
 cookie=0x85eb64bc3e0f6ac1, duration=277743.276s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x85eb64bc3e0f6ac1, duration=277743.274s, table=2, n_packets=40113, n_bytes=3161002, idle_age=1, hard_age=65534, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x85eb64bc3e0f6ac1, duration=277743.273s, table=2, n_packets=6712, n_bytes=324280, idle_age=65534, hard_age=65534, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
 cookie=0x85eb64bc3e0f6ac1, duration=277743.272s, table=3, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x85eb64bc3e0f6ac1, duration=197945.765s, table=4, n_packets=39836, n_bytes=2683592, idle_age=1, hard_age=65534, priority=1,tun_id=0x5c actions=mod_vlan_vid:6,resubmit(,10)
 cookie=0x85eb64bc3e0f6ac1, duration=277743.271s, table=4, n_packets=15, n_bytes=1090, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x85eb64bc3e0f6ac1, duration=277743.270s, table=6, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0x85eb64bc3e0f6ac1, duration=277743.269s, table=10, n_packets=40283, n_bytes=2747486, idle_age=1, hard_age=65534, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0x85eb64bc3e0f6ac1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:OXM_OF_IN_PORT[]),output:1
 cookie=0x85eb64bc3e0f6ac1, duration=35687.268s, table=20, n_packets=17405, n_bytes=1695793, hard_timeout=300, idle_age=1, hard_age=1, priority=1,vlan_tci=0x0006/0x0fff,dl_dst=fa:16:3e:ea:0e:8f actions=load:0->NXM_OF_VLAN_TCI[],load:0x5c->NXM_NX_TUN_ID[],output:4
 cookie=0x85eb64bc3e0f6ac1, duration=277743.268s, table=20, n_packets=112, n_bytes=11204, idle_age=21324, hard_age=65534, priority=0 actions=resubmit(,22)
 cookie=0x85eb64bc3e0f6ac1, duration=197945.766s, table=22, n_packets=122, n_bytes=12480, idle_age=21324, hard_age=65534, priority=1,dl_vlan=6 actions=strip_vlan,load:0x5c->NXM_NX_TUN_ID[],output:4,output:3,output:2
 cookie=0x85eb64bc3e0f6ac1, duration=277743.267s, table=22, n_packets=5678, n_bytes=276452, idle_age=65534, hard_age=65534, priority=0 actions=drop
 ```
* compute_0 br-int流表

 ```
 [root@overcloud-novacompute-0 ~]# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0xa765c80babd2ebcb, duration=197996.437s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=10,icmp6,in_port=8,icmp_type=136 actions=resubmit(,24)
 cookie=0xa765c80babd2ebcb, duration=197996.435s, table=0, n_packets=7167, n_bytes=301014, idle_age=8, hard_age=65534, priority=10,arp,in_port=8 actions=resubmit(,24)
 cookie=0xa765c80babd2ebcb, duration=277794.862s, table=0, n_packets=648145, n_bytes=37251497, idle_age=1, hard_age=65534, priority=2,in_port=1 actions=drop
 cookie=0xa765c80babd2ebcb, duration=197996.441s, table=0, n_packets=32551, n_bytes=2813463, idle_age=3, hard_age=65534, priority=9,in_port=8 actions=resubmit(,25)
 cookie=0xa765c80babd2ebcb, duration=277794.916s, table=0, n_packets=40319, n_bytes=2750270, idle_age=3, hard_age=65534, priority=0 actions=NORMAL
 cookie=0xa765c80babd2ebcb, duration=277794.918s, table=23, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xa765c80babd2ebcb, duration=197996.439s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=2,icmp6,in_port=8,icmp_type=136,nd_target=fe80::f816:3eff:fe6b:d695 actions=NORMAL
 cookie=0xa765c80babd2ebcb, duration=197996.436s, table=24, n_packets=7155, n_bytes=300510, idle_age=8, hard_age=65534, priority=2,arp,in_port=8,arp_spa=192.168.100.104 actions=resubmit(,25)
 cookie=0xa765c80babd2ebcb, duration=277794.915s, table=24, n_packets=18, n_bytes=756, idle_age=21373, hard_age=65534, priority=0 actions=drop
 cookie=0xa765c80babd2ebcb, duration=197996.443s, table=25, n_packets=39706, n_bytes=3113973, idle_age=3, hard_age=65534, priority=2,in_port=8,dl_src=fa:16:3e:6b:d6:95 actions=NORMAL
 ```
* compute_0 br-ex流表

 ```
 [root@overcloud-novacompute-0 ~]# ovs-ofctl dump-flows br-ex
NXST_FLOW reply (xid=0x4):
 cookie=0xaff6936771f75f35, duration=277836.775s, table=0, n_packets=1199, n_bytes=62346, idle_age=21417, hard_age=65534, priority=2,in_port=5 actions=drop
 cookie=0xaff6936771f75f35, duration=277836.795s, table=0, n_packets=6286307, n_bytes=3591027442, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
 ```