# undercloud虚拟环境部署

---



## 安装undercloud VM

**以下操作在物理机执行：**

### 1. 添加Newton的repo

我们默认使用UniedStack的内部源:

```
[openstack-newton]
name=openstack-newton
baseurl=http://tripleo.ustack.com/repo/openstack-newton/
enabled=1
gpgcheck=0
priority=1
```

添加内网解析

```
10.0.100.250 tripleo.ustack.com
```

如果通过公网安装：

```
sudo curl -L -o /etc/yum.repos.d/delorean-newton.repo https://trunk.rdoproject.org/centos7-newton/current/delorean.repo
sudo curl -L -o /etc/yum.repos.d/delorean-deps-newton.repo http://trunk.rdoproject.org/centos7-newton/delorean-deps.repo
sudo yum -y install --enablerepo=extras centos-release-ceph-jewel
sudo sed -i -e 's%gpgcheck=.*%gpgcheck=0%' /etc/yum.repos.d/CentOS-Ceph-Jewel.repo
```

如果通过内网安装，需要有这些repository

* CentOS Base
* CentOS EPEL
* RDO Trunk Delorean repository
* Newton Delorean Deps repository
* Ceph jewel repository

### 2. 安装yum-plugin-priorites

```
sudo yum -y install yum-plugin-priorities
```

### 3. 安装 TripleO CLI

```
sudo yum install -y python-tripleoclient
```

### 4. 通过环境变量定义安装参数

```vim
#指定undercloud vm 使用centos7
export NODE_DIST=centos7

#指定有多少个overcloud vm，以及他们的配置
export NODE_COUNT=2
export NODE_CPU=1
export NODE_MEM=6144
export NODE_DISK=40

#undercloud的配置
export UNDERCLOUD_NODE_CPU=4
export UNDERCLOUD_NODE_MEM=8192
export UNDERCLOUD_NODE_DISK=30

#指定undercloud 虚拟机的模板,这个是镜像的目录
export DIB_LOCAL_IMAGE=rhel-guest-image-7.1-20150224.0.x86_64.qcow2

# 需要创建哪几个网桥
export TESTENV_ARGS="--baremetal-bridge-names 'brbm brbm1 brbm2'"

# （可选）指定使用哪个virt pool 
export LIBVIRT_VOL_POOL=tripleo
export LIBVIRT_VOL_POOL_TARGET=/home/vm_storage_pool
```

### 5. 安装undercloud vm

这一步安装的并不是undercloud，这里安装的仅仅是运行undercloud的虚拟机。运行安装命令时，我们需要建立一个普通用户，用以运行安装undercloud。

```
# useradd zhao
# echo "zhao ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/zhao
```

切换到这个普通用户下面开始安装：

```
$ su - zhao
$ instack-virt-setup
```

## ssh 进入undercloud os

```
$ ssh root@instack
```

## 部署undercloud openstack

**以下步骤在undercloud vm 中执行**

### 1. 安装TripleOClient

```
$ su - stack
$ sudo yum -y install yum-plugin-prioritiessudo
$ sudo yum install -y python-tripleoclient
```

### 2. 编辑undercloud 配置文件

创建undercloud配置文件，并修改里面的配置。

```
$ cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf
```

然后去修改我们的配置

```
$ vim ~/undercloud.conf
[DEFAULT]
local_ip = 192.0.2.1/24
network_gateway = 192.0.2.1
undercloud_public_vip = 192.0.2.2
undercloud_admin_vip = 192.0.2.3
local_interface = eth0 # pxe装机的网桥，必须和你的overcloud的pxe网卡在同一个vlan或者说同一个桥下面
network_cidr = 192.0.2.0/24
masquerade_network = 192.0.2.0/24
dhcp_start = 192.0.2.5
dhcp_end = 192.0.2.24
inspection_interface = br-ctlplane
inspection_iprange = 192.0.2.100,192.0.2.120
inspection_extras = true
undercloud_debug = true
[auth]
```

### 3. 部署undercloud

```
$ openstack undercloud install
```



