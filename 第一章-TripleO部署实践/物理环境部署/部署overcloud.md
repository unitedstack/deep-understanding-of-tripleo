# 部署Overcloud

---

在这一步，我们已经完成了undercloud的部署。现在要开始部署overcloud。

## 1. 准备overcloud 镜像

Overcloud 镜像可以自己制作也可以下载现成的, 我们这里使用了ustack内部的镜像:

```
$ lftp tripleo.ustack.com:/pub/images/newton/
cd ok, cwd=/pub/images/newton
lftp tripleo.ustack.com:/pub/images/newton> ls
-rw-r--r--    1 0        0        1670748160 Jan 08 10:49 overcloud_image.tar
lftp tripleo.ustack.com:/pub/images/newton> get overcloud_image.tar 
```

这里的演示使用下载的镜像。[Overcloud 镜像下载地址](http://buildlogs.centos.org/centos/7/cloud/x86_64/tripleo_images/)

下载后解压，将这些文件放到stack用户的根目录底下：

```
$ ls ~/images/
ironic-python-agent.initramfs
ironic-python-agent.kernel
overcloud-full.initrd
overcloud-full.qcow2
overcloud-full.vmlinuz
```

> 如果要修改镜像的root密码： `$ virt-customize -a overcloud-full.qcow2 --root-password password:<my_root_password>`


## 2. 上传镜像

```
$ . stackrc
$ openstack overcloud image upload
```
> 默认情况下，我们的glance使用的是swift的存储后端，所以在进行这个步骤时必须保证你的swift的服务是可以使用的。

上传完成之后，查看镜像：
```
$ openstack image list
+--------------------------------------+------------------------+
| ID | Name |
+--------------------------------------+------------------------+
| 765a46af-4417-4592-91e5-a300ead3faf6 | bm-deploy-ramdisk      |
| 09b40e3d-0382-4925-a356-3a4b4f36b514 | bm-deploy-kernel       |
| ef793cd0-e65c-456a-a675-63cd57610bd5 | overcloud-full         |
| 9a51a6cb-4670-40de-b64b-b70f4dd44152 | overcloud-full-initrd  |
| 4f7e33f4-d617-47c1-b36f-cbe90f132e5d | overcloud-full-vmlinuz |
+--------------------------------------+------------------------+
```

这个会显示在收集物理机信息的时候使用的PXE镜像，上传的时候会把这些镜像都拷贝到/httpboot这个目录下面


## 3. 定义物理机列表

我们现在已经有了镜像，紧接着就是定义我们overcloud主机了。我们将overcloud vm的信息写入instackenv.json。参照以下格式：
```
$ vim instackenv.json:
{
"nodes": [
    {
      "pm_user": "root",
      "arch": "x86_64",
      "name": "10.0.10.126_compute",
      "pm_addr": "10.0.10.126",
      "pm_password": "XXXX",
      "pm_type": "pxe_ipmitool",
      "mac": [
        "24:6E:96:06:2E:04"
      ],
      "cpu": "32",
      "memory": "8192",
      "disk": "599"
    },
    {
      "pm_user": "root",
      "arch": "x86_64",
      "name": "10.0.10.106_control",
      "pm_addr": "10.0.10.106",
      "pm_password": "XXXX",
      "pm_type": "pxe_ipmitool",
      "mac": [
        "EC:F4:BB:D9:88:3C"
      ],
      "cpu": "32",
      "memory": "8192",
      "disk": "599"
    },
    {
      "pm_user": "root",
      "arch": "x86_64",
      "name": "10.0.108.122_control",
      "pm_addr": "10.0.108.122",
      "pm_password": "XXXX",
      "pm_type": "pxe_ipmitool",
      "mac": [
        "B8:CA:3A:6B:2D:24"
      ],
      "cpu": "32",
      "memory": "8192",
      "disk": "599"
    },
    {
      "pm_user": "root",
      "arch": "x86_64",
      "name": "10.0.108.171_control",
      "pm_addr": "10.0.108.171",
      "pm_password": "XXXX",
      "pm_type": "pxe_ipmitool",
      "mac": [
	"EC:F4:BB:D9:7F:1C"
      ],
      "cpu": "32",
      "memory": "8192",
      "disk": "599"
    },
    {
      "pm_user": "root",
      "arch": "x86_64",
      "name": "10.0.108.123_compute",
      "pm_addr": "10.0.108.123",
      "pm_password": "XXXX",
      "pm_type": "pxe_ipmitool",
      "mac": [
        "B8:CA:3A:6C:46:6C"
      ],
      "cpu": "32",
      "memory": "8192",
      "disk": "599"
    },
    {
      "pm_user": "root",
      "arch": "x86_64",
      "name": "10.0.10.113_compute",
      "pm_addr": "10.0.10.113",
      "pm_password": "XXXX",
      "pm_type": "pxe_ipmitool",
      "mac": [
        "EC:F4:BB:D9:7A:6C"
      ],
      "cpu": "32",
      "memory": "8192",
      "disk": "599"
    }
  ]
}

```

## 4. 收集物理机信息
我们已经把物理机的列表全部写在我们的instackenv.json文件中了，那么下一步就是要让ironic去收集这些物理机的信息了。首先我们需要先注册虚拟机：

```
$ . stackrc
$ openstack baremetal import instackenv.json
```
这一步，只是注册了我们的物理机，我们可以在ironic中看到,我们注册的物理机：
```
$ ironic node-list
+--------------------------------------+----------------------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name                 | Instance UUID                        | Power State | Provisioffing State| Maintenance |
+--------------------------------------+----------------------+--------------------------------------+-------------+--------------------+-------------+
| 7a14b906-5810-4c50-b06d-3f6a532ff05b | 10.0.108.122_control | None                                 | power off   | available          | False       |
| b6101189-44bb-4133-8186-1dc9758c1d0e | 10.0.10.113_compute  | None                                 | power off   | available          | False       |
| 689fbaf6-186b-4ee0-921e-02ea611f02d5 | 10.0.108.171_control | None                                 | power off   | available          | False       |
| 147d03d4-1a98-4898-9aca-5fd6a46f6a2c | 10.0.10.126_compute  | None                                 | power off   | available          | False       |
| 8279e339-11d5-4117-afd7-581293215d61 | 10.0.10.106_control  | None                                 | power off   | available          | False       |
| 2c7205f1-7002-4678-843f-eaa1743d85b4 | 10.0.108.123_compute | None                                 | power off   | available          | False       |
+--------------------------------------+----------------------+--------------------------------------+-------------+--------------------+-------------+

```
接着我们需要去收集我们物理机的信息，我们把这个过程固化成脚本：
```
#!/bin/bash

node=`ironic node-list | grep "power" | awk -F\| '{print $2}'`
for i in ${node[*]}
do
   openstack baremetal node manage $i
   nohup openstack overcloud node introspect $i --provide  &
done
```
我们使用把收集信息的进程打到后台，你可以使用`ps ax | grep openstack`来查看任务的情况。在收集物理机信息的时候，会尝试开启物理机，然后进入PXE引导，之会安装一个带有ironic的镜像，然后由ironic去收集物理机的信息。所以，在这时候必须保证你的PXE网络是OK的。在收集完我们的物理机的信息之后，我们可以通过下面的命令来检查收集到的物理机信息：

```
$ openstack  baremetal introspection data save <UUID> > ironic_data.save
```
正常情况下，我们可以在导出的文件里看到很多的物理机定制信息。

## 5. 定义根磁盘
我们必须为机器定义一个根磁盘，以供后续装机过程的使用。很多时候，/dev/sda是我们的根磁盘，但也有根磁盘不是它的时候。所谓的根磁盘就是你的系统盘，BIOS中可以设置开机从哪一个磁盘启动，你选了哪一个盘，哪一个盘就是你的根磁盘。我们这里默认的根磁盘就是/dev/sda。

```bash
ironic node-update <UUID> add properties/root_device='{"name": "/dev/sda"}'
```

## 7. 定义主机角色
首先为我们的控制节点定义我们的节点类型：
```
list=(`ironic node-list|grep control|awk '{print $2}'`);for i in {0..2};do ironic node-update ${list[$i]} replace properties/capabilities="node:controller-$i,boot_option:local";done
```
接下来是我们的计算节点
```
list=(`ironic node-list|grep compute|awk '{print $2}'`);for i in {0..2};do ironic node-update ${list[$i]} replace properties/capabilities="node:compute-$i,boot_option:local";done
```

## 8. 定义tripleo-heat-template配置文件
我们这边下载ustack的内部配置文件
```
$ wget http://tripleo.ustack.com/template/tripleo/newton/deploy_template.tar
$ tar xvf deploy_template.tar
```
替换当前tripleo所使用的配置文件：
```
$ rm -fr /usr/share/openstack-tripleo-heat-templates/
$ cp -r deploy_template/openstack-tripleo-heat-templates  /usr/share/
```

## 8. 划分网络
接下来我们根据前面定义的网络架构进行网络规划,我们需要修改我们的配置文件network-environment.yaml：
```
$ vim deploy_template/templates/environments/network-environment.yaml

resource_registry:
  OS::TripleO::Compute::Net::SoftwareConfig:
    ../nic-configs/compute.yaml
  OS::TripleO::Controller::Net::SoftwareConfig:
    ../nic-configs/controller.yaml

parameter_defaults:
  ControlPlaneSubnetCidr: '24'
  ControlPlaneDefaultRoute: 10.0.130.1
  EC2MetadataIp: 10.0.130.31  # Generally the IP of the Undercloud
  StorageNetCidr: 10.0.132.0/24
  StorageMgmtNetCidr: 10.0.133.0/24
  TenantNetCidr: 10.0.134.0/24
  ExternalNetCidr: 10.0.135.0/24
  InternalApiNetCidr: 10.0.136.0/24
  StorageNetworkVlanID: 4002
  StorageMgmtNetworkVlanID: 4003
  TenantNetworkVlanID: 4004
  ExternalNetworkVlanID: 4005
  InternalApiNetworkVlanID: 4006
  StorageAllocationPools: [{'start': '10.0.132.40', 'end': '10.0.132.80'}]
  StorageMgmtAllocationPools: [{'start': '10.0.133.40', 'end': '10.0.133.80'}]
  TenantAllocationPools: [{'start': '10.0.134.40', 'end': '10.0.134.80'}]
  ExternalAllocationPools: [{'start': '10.0.135.40', 'end': '10.0.135.80'}]
  InternalApiAllocationPools: [{'start': '10.0.136.40', 'end': '10.0.136.80'}]
  ExternalInterfaceDefaultRoute: 10.0.135.1
  DnsServers: ["119.29.29.29","8.8.4.4"]
  NeutronExternalNetworkBridge: "''"
  NeutronTunnelTypes: 'vxlan'
  NeutronNetworkType: 'vxlan,vlan'

  BondInterfaceOvsOptions: "bond_mode=active-backup"
  NeutronNetworkVLANRanges: "datacentre:4008:4015"
  NeutronVniRanges: "1:1000"
  NeutronBridgeMappings: "datacentre:br-ex"
```

## 9. 定义网卡
这里我们需要指定每一个角色的网卡的对接顺序：
```
$ vim templates/nic-configs/controller.yaml

...
resources:
  OsNetConfigImpl:
    type: OS::Heat::StructuredConfig
    properties:
      group: os-apply-config
      config:
        os_net_config:
          network_config:
            -
              type: interface
              name: em3 # 这个是第一个千兆网的地址
              
              
              
              use_dhcp: false
              addresses:
                -
                  ip_netmask:
                    list_join:
                      - '/'
                      - - {get_param: ControlPlaneIp}
                        - {get_param: ControlPlaneSubnetCidr}
              routes:
                -
                  ip_netmask: 169.254.169.254/32
                  next_hop: {get_param: EC2MetadataIp}
            -
              type: ovs_bridge # 起一个ovs的桥
              name: {get_input: bridge_name}
              dns_servers: {get_param: DnsServers}
              members:
                -
                  type: ovs_bond
                  name: bond1 # 这个桥下面绑定的网卡，这里做了bond
                  ovs_options: {get_param: BondInterfaceOvsOptions}
                  members:
                    -
                      type: interface
                      name: em1
                      primary: true
                    -
                      type: interface
                      name: em2
                -
                  type: vlan # 这个桥下的vlan
                  device: bond1
                  vlan_id: {get_param: ExternalNetworkVlanID}
                  addresses:
                    -
                      ip_netmask: {get_param: ExternalIpSubnet}
                  routes:
                    -
                      default: true
                      next_hop: {get_param: ExternalInterfaceDefaultRoute}
                -
                  type: vlan
                  device: bond1
                  vlan_id: {get_param: InternalApiNetworkVlanID}
                  addresses:
                    -
                      ip_netmask: {get_param: InternalApiIpSubnet}
                -
                  type: vlan
                  device: bond1
                  vlan_id: {get_param: StorageNetworkVlanID}
                  addresses:
                    -
                      ip_netmask: {get_param: StorageIpSubnet}
                -
                  type: vlan
                  device: bond1
                  vlan_id: {get_param: StorageMgmtNetworkVlanID}
                  addresses:
                    -
                      ip_netmask: {get_param: StorageMgmtIpSubnet}
                -
                  type: vlan
                  device: bond1
                  vlan_id: {get_param: TenantNetworkVlanID}
                  addresses:
                    -
                      ip_netmask: {get_param: TenantIpSubnet}

...

```


## 10. 指定主机名
我们这里也需要指定我们的主机的名字：
```
$ vim templates/environments/scheduler_hints_env.yaml

parameter_defaults:
  ControllerSchedulerHints:
    'capabilities:node': 'controller-%index%'
  NovaComputeSchedulerHints:
    'capabilities:node': 'compute-%index%'
  CephStorageSchedulerHints:
    'capabilities:node': 'ceph-storage-%index%'
  HostnameMap:
    overcloud-controller-0: overcloud-controller-1-1
    overcloud-controller-1: overcloud-controller-1-2
    overcloud-controller-2: overcloud-controller-1-3
    overcloud-novacompute-0: overcloud-compute-1-1
    overcloud-novacompute-1: overcloud-compute-1-2
    overcloud-novacompute-2: overcloud-compute-1-3
```
这里有必要解释一下，`'capabilities:node': 'controller-%index%'`中` %index% `是一直从0往上递增的。也就是匹配到了`ironic node-show <uuid> `中我们定义的角色内，如:`properties/capabilities="node:controller-0,boot_option:local" `。


## 11. 指定主机IP
我们在上面定义了主机名字，紧接着就是我们的主机IP了：
```
resource_registry:
  OS::TripleO::Controller::Ports::ExternalPort: /usr/share/openstack-tripleo-heat-templates/network/ports/external_from_pool.yaml
  OS::TripleO::Controller::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat-templates/network/ports/internal_api_from_pool.yaml
  OS::TripleO::Controller::Ports::StoragePort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage_from_pool.yaml
  OS::TripleO::Controller::Ports::StorageMgmtPort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage_mgmt_from_pool.yaml
  OS::TripleO::Controller::Ports::TenantPort: /usr/share/openstack-tripleo-heat-templates/network/ports/tenant_from_pool.yaml
  # Management network is optional and disabled by default
  #OS::TripleO::Controller::Ports::ManagementPort: /usr/share/openstack-tripleo-heat-templates/network/ports/management_from_pool.yaml

  OS::TripleO::Compute::Ports::ExternalPort: /usr/share/openstack-tripleo-heat-templates/network/ports/noop.yaml
  OS::TripleO::Compute::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat-templates/network/ports/internal_api_from_pool.yaml
  OS::TripleO::Compute::Ports::StoragePort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage_from_pool.yaml
  OS::TripleO::Compute::Ports::StorageMgmtPort: /usr/share/openstack-tripleo-heat-templates/network/ports/noop.yaml
  OS::TripleO::Compute::Ports::TenantPort: /usr/share/openstack-tripleo-heat-templates/network/ports/tenant_from_pool.yaml
  #OS::TripleO::Compute::Ports::ManagementPort: /usr/share/openstack-tripleo-heat-templates/network/ports/management_from_pool.yaml

  OS::TripleO::Network::Ports::NetVipMap: /usr/share/openstack-tripleo-heat-templates/network/ports/net_vip_map_external.yaml
  OS::TripleO::Network::Ports::ExternalVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/noop.yaml
  OS::TripleO::Network::Ports::InternalApiVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/noop.yaml
  OS::TripleO::Network::Ports::StorageVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/noop.yaml
  OS::TripleO::Network::Ports::StorageMgmtVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/noop.yaml
  OS::TripleO::Network::Ports::RedisVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/from_service.yaml

parameter_defaults:
  ControllerIPs:
    # Each controller will get an IP from the lists below, first controller, first IP
    external:
    - 10.0.135.31
    - 10.0.135.32
    - 10.0.135.33
    internal_api:
    - 10.0.136.31
    - 10.0.136.32
    - 10.0.136.33
    storage:
    - 10.0.132.31
    - 10.0.132.32
    - 10.0.132.33
    storage_mgmt:
    - 10.0.133.31
    - 10.0.133.32
    - 10.0.133.33
    tenant:
    - 10.0.134.31
    - 10.0.134.32
    - 10.0.134.33
    #management:
    #- 172.16.4.251
  NovaComputeIPs:
    # Each compute will get an IP from the lists below, first compute, first IP
    internal_api:
    - 10.0.136.34
    - 10.0.136.35
    - 10.0.136.36
    storage:
    - 10.0.132.34
    - 10.0.132.35
    - 10.0.132.36
    storage_mgmt:
    - 10.0.133.34
    - 10.0.133.35
    - 10.0.133.36
    tenant:
    - 10.0.134.34
    - 10.0.134.35
    - 10.0.134.36
    #management:
    #- 172.16.4.252
    #
  ExternalNetworkVip: 10.0.135.70
  InternalApiNetworkVip: 10.0.136.70
  StorageNetworkVip: 10.0.132.70
  StorageMgmtNetworkVip: 10.0.133.70
  ServiceVips:
    redis: 10.0.136.71

```
所有的定义都是按着节点数目来排序的。

## 12. 定义节点role

我们这边需要做计算节点和存储结点的融合，所以需要小小的调整：
```
- name: Controller
  CountDefault: 1
  ServicesDefault:
    - OS::TripleO::Services::CACerts
    - OS::TripleO::Services::CephMon
    - OS::TripleO::Services::CephExternal
    - OS::TripleO::Services::CephRgw
    - OS::TripleO::Services::CinderApi
    - OS::TripleO::Services::CinderBackup
    - OS::TripleO::Services::CinderScheduler
    - OS::TripleO::Services::CinderVolume
    - OS::TripleO::Services::Core
    - OS::TripleO::Services::Kernel
    - OS::TripleO::Services::Keystone
    - OS::TripleO::Services::GlanceApi
    - OS::TripleO::Services::GlanceRegistry
    - OS::TripleO::Services::HeatApi
    - OS::TripleO::Services::HeatApiCfn
    - OS::TripleO::Services::HeatApiCloudwatch
    - OS::TripleO::Services::HeatEngine
    - OS::TripleO::Services::MySQL
    - OS::TripleO::Services::NeutronDhcpAgent
    - OS::TripleO::Services::NeutronL3Agent
    - OS::TripleO::Services::NeutronMetadataAgent
    - OS::TripleO::Services::NeutronApi
    - OS::TripleO::Services::NeutronCorePlugin
    - OS::TripleO::Services::NeutronOvsAgent
    - OS::TripleO::Services::RabbitMQ
    - OS::TripleO::Services::HAproxy
    - OS::TripleO::Services::Keepalived
    - OS::TripleO::Services::Memcached
    - OS::TripleO::Services::Pacemaker
    - OS::TripleO::Services::Redis
    - OS::TripleO::Services::NovaConductor
    - OS::TripleO::Services::MongoDb
    - OS::TripleO::Services::NovaApi
    - OS::TripleO::Services::NovaMetadata
    - OS::TripleO::Services::NovaScheduler
    - OS::TripleO::Services::NovaConsoleauth
    - OS::TripleO::Services::NovaVncProxy
    - OS::TripleO::Services::Ntp
    - OS::TripleO::Services::SwiftProxy
    - OS::TripleO::Services::SwiftStorage
    - OS::TripleO::Services::SwiftRingBuilder
    - OS::TripleO::Services::Snmp
    - OS::TripleO::Services::Timezone
    - OS::TripleO::Services::CeilometerApi
    - OS::TripleO::Services::CeilometerCollector
    - OS::TripleO::Services::CeilometerExpirer
    - OS::TripleO::Services::CeilometerAgentCentral
    - OS::TripleO::Services::CeilometerAgentNotification
    - OS::TripleO::Services::Horizon
    - OS::TripleO::Services::GnocchiApi
    - OS::TripleO::Services::GnocchiMetricd
    - OS::TripleO::Services::GnocchiStatsd
    - OS::TripleO::Services::ManilaApi
    - OS::TripleO::Services::ManilaScheduler
    - OS::TripleO::Services::ManilaBackendGeneric
    - OS::TripleO::Services::ManilaBackendNetapp
    - OS::TripleO::Services::ManilaBackendCephFs
    - OS::TripleO::Services::ManilaShare
    - OS::TripleO::Services::AodhApi
    - OS::TripleO::Services::AodhEvaluator
    - OS::TripleO::Services::AodhNotifier
    - OS::TripleO::Services::AodhListener
    - OS::TripleO::Services::SaharaApi
    - OS::TripleO::Services::SaharaEngine
    - OS::TripleO::Services::IronicApi
    - OS::TripleO::Services::IronicConductor
    - OS::TripleO::Services::NovaIronic
    - OS::TripleO::Services::TripleoPackages
    - OS::TripleO::Services::TripleoFirewall
    - OS::TripleO::Services::OpenDaylightApi
    - OS::TripleO::Services::OpenDaylightOvs
    - OS::TripleO::Services::SensuClient
    - OS::TripleO::Services::FluentdClient
    - OS::TripleO::Services::VipHosts

- name: Compute
  CountDefault: 1
  HostnameFormatDefault: '%stackname%-novacompute-%index%'
  ServicesDefault:
    - OS::TripleO::Services::CACerts
    - OS::TripleO::Services::CephOSD # 把ceph的role加入到我们的compute节
    - OS::TripleO::Services::CephClient
    - OS::TripleO::Services::CephExternal
    - OS::TripleO::Services::Timezone
    - OS::TripleO::Services::Ntp
    - OS::TripleO::Services::Snmp
    - OS::TripleO::Services::NovaCompute
    - OS::TripleO::Services::NovaLibvirt
    - OS::TripleO::Services::Kernel
    - OS::TripleO::Services::ComputeNeutronCorePlugin
    - OS::TripleO::Services::ComputeNeutronOvsAgent
    - OS::TripleO::Services::ComputeCeilometerAgent
    - OS::TripleO::Services::ComputeNeutronL3Agent
    - OS::TripleO::Services::ComputeNeutronMetadataAgent
    - OS::TripleO::Services::TripleoPackages
    - OS::TripleO::Services::TripleoFirewall
    - OS::TripleO::Services::NeutronSriovAgent
    - OS::TripleO::Services::OpenDaylightOvs
    - OS::TripleO::Services::SensuClient
    - OS::TripleO::Services::FluentdClient
    - OS::TripleO::Services::VipHosts
```

## 开始部署overcloud
```
openstack overcloud deploy --templates \
  -r /home/stack/templates/roles/roles_data.yaml \
  -e /home/stack/templates/environments/network-environment.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml \
  -e /home/stack/templates/environments/storage-environment.yaml \
  -e /home/stack/templates/environments/ips-from-pool-all.yaml \
  --control-flavor baremetal\
  --compute-flavor baremetal\
  --control-scale 3 \
  --compute-scale 3 \
  --ntp-server 0.pool.ntp.org 
```


