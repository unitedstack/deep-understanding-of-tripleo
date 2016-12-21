部署overcloud
---
## 1. 准备overcloud 镜像
Overcloud 镜像可以自己制作也可以下载现成的。这里的演示使用下载的镜像。

[Overcloud 镜像下载地址](http://buildlogs.centos.org/centos/7/cloud/x86_64/tripleo_images/)

将这些文件放到stack用户的根目录底下
```
ironic-python-agent.initramfs
ironic-python-agent.kernel
overcloud-full.initrd
overcloud-full.qcow2
overcloud-full.vmlinuz
```

## 2. 上传镜像
```
$ . stackrc
$ openstack overcloud image upload
```


## 3. 定义根磁盘
查看introspection得到的磁盘信息，确认sda是不是我们想要的根磁盘。
```
list=(`ironic node-list|grep power|awk '{print $2}'`);for i in  ${list[*]} ;do openstack baremetal introspection data save $i | jq '.inventory.disks' ;done
```
### 如果sda是我们想要的根磁盘:
```
list=(`ironic node-list|grep power|awk '{print $2}'`);for i in ${list[*]};do ironic node-update $i add properties/root_device='{"name": "/dev/sda"}';done
```


### 如果sda不是我们想要的根磁盘
那就需要使用serial或者wwn来定义根磁盘，手动对每一个node依次执行定义:
```

#wwn
ironic node-update $i add properties/root_device=properties/root_device='{"wwn": "xxx"}'

#serial
ironic node-update $i add properties/root_device=properties/root_device='{"serial": "xxx"}'

```

修正logic_gb
```
a
```


## 4. 定义ceph
在stack用户目录下建立`templates`目录
```
mkdir ~/templates
```
复制ceph 配置文件到templates目录中
```
cp /usr/share/openstack-tripleo-heat-templates/environments/storage-environment.yaml ~/templates/
```






