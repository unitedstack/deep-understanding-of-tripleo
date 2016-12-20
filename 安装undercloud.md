undercloud 虚拟环境部署（公网）
---
以下操作在物理机执行：

## 1. 添加Newton的repo
```
sudo curl -L -o /etc/yum.repos.d/delorean-newton.repo https://trunk.rdoproject.org/centos7-newton/current/delorean.repo

sudo curl -L -o /etc/yum.repos.d/delorean-deps-newton.repo http://trunk.rdoproject.org/centos7-newton/delorean-deps.repo

sudo yum -y install --enablerepo=extras centos-release-ceph-jewel
sudo sed -i -e 's%gpgcheck=.*%gpgcheck=0%' /etc/yum.repos.d/CentOS-Ceph-Jewel.repo
```

## 2. 安装yum-plugin-priorites
```
sudo yum -y install yum-plugin-priorities

```


## 3. 安装 TripleO CLI
```
sudo yum install -y python-tripleoclient

```

## 4. 通过环境变量定义安装参数
```
export NODE_DIST=centos7


export NODE_COUNT=2
export NODE_CPU=1
export NODE_MEM=6144
export NODE_DISK=40


export UNDERCLOUD_NODE_CPU=4
export UNDERCLOUD_NODE_MEM=8192
export UNDERCLOUD_NODE_DISK=30


export DIB_LOCAL_IMAGE=rhel-guest-image-7.1-20150224.0.x86_64.qcow2


export TESTENV_ARGS="--baremetal-bridge-names 'brbm brbm1 brbm2'"

# If you wish to specify an alternative pool name:
export LIBVIRT_VOL_POOL=tripleo
# If you want to specify an alternative target
export LIBVIRT_VOL_POOL_TARGET=/home/vm_storage_pool


```


## 5. 安装undercloud vm 
（并不是undercloud，这里安装的仅仅是运行undercloud的虚拟机。）
```
instack-virt-setup
```



