# 部署Overcloud

---

在这一步，我们已经完成了undercloud的部署。现在要开始部署overcloud。

## 1. 准备overcloud 镜像

Overcloud 镜像可以自己制作也可以下载现成的, 如果要使用ustack内部的镜像，请使用lftp 下载。tripleo.ustack.com 对应的 IP 是10.0.100.250.

```bash
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
aaovercloud-full.qcow2
overcloud-full.vmlinux
```

> 如果要修改镜像的root密码： `$ virt-customize -a overcloud-full.qcow2 --root-password password:<my_root_password>`

## 2. 上传镜像

第一次部署推荐使用社区镜像，PoC 环境和生产环境一定要自己制作镜像。

```
$ . stackrc
$ openstack overcloud image upload --image-path /home/stack/images
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

## 3. 收集物理机信息

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

## 4. 导入overcloud信息


```
openstack baremetal import instackenv.json
```

进行introspection收集overcloud node 的信息

```
openstack baremetal introspection bulk start
```


## 5. 为物理机定义节点类型

在规划节点时，希望特定的物理机作为特定的角色。比如有一台物理机，我们在ironic 配置文件里将它定义为overcloud 中ceph节点，不要任性的变成计算节点或者控制节点。这样，我们就需要为这些节点定义类型。  
定义类型有两种方法，比如我只需要ceph节点，安装在ceph host 上\(在ironic 配置文件中叫做ceph\)，nova 节点安装在nova host 上，控制节点安装在controller host 上。

正常写法：

```
openstack baremetal node set --property capabilities='profile:compute,boot_option:local <compute node uuid>'
openstack baremetal node set --property capabilities='profile:control,boot_option:local <control node uuid>'
openstack baremetal node set --property capabilities='profile:ceph-storage,boot_option:local <ceph node uuid>'
```

一句话写法：

```
ironic node-list|grep 'controller'|awk '{print $2}'|xargs -I{} ironic node-update {} add properties/capabilities='profile:control,boot_option:local'

ironic node-list|grep 'compute'|awk '{print $2}'|xargs -I{} ironic node-update {} add properties/capabilities='profile:compute,boot_option:local'

ironic node-list|grep 'ceph'|awk '{print $2}'|xargs -I{} ironic node-update {} add properties/capabilities='profile:ceph-storage,boot_option:local'
```


## 4. 定义根磁盘

在执行完`openstack baremetal introspection bulk start`之后，根据得到的信息来定义overcloud节点的根磁盘。
根磁盘可以通过以下参数来指定。

```
model (String): Device identifier.
vendor (String): Device vendor.
serial (String): Disk serial number.
wwn (String): Unique storage identifier.
size (Integer): Size of the device in GB.
```

查看introspection得到的磁盘信息，确认sda是不是我们想要的根磁盘。

```bash
list=(`ironic node-list|grep power|awk '{print $2}'`);for i in ${list[*]} ;do openstack baremetal introspection data save $i | jq ".inventory.disks" ;done
```

### 如果sda是我们想要的根磁盘:

```bash
list=(`ironic node-list|grep power|awk '{print $2}'`);for i in ${list[*]};do ironic node-update $i add properties/root_device='{"name": "/dev/sda"}';done
```

### 如果sda不是我们想要的根磁盘

那就需要使用wwn来定义根磁盘，手动对每一个node依次执行定义:

```
#wwn
ironic node-update $i add properties/root_device=properties/root_device='{"wwn": "xxx"}'
```

然后需要修正logic\_gb，可以重新执行introspection或者手动指定logic\_gb

* 重新执行introspection
```
openstack baremetal introspection bulk start
```
* 或者更新logic\_gb
```
ironic node-update <UUID> add properties/local_gb=<NEW VALUE>
```

## 4. 部署

在进行自定义部署之前，应该使用简单部署验证以上步骤都没有问题，确保主机可以被正常部署出来。
现在开始我们的部署之旅 -- 最简单的部署，等待一杯咖啡的时间。

```
openstack overcloud deploy --template
```



部署步骤

1. 写overcloud 节点配置文件,定义overcloud所有节点的ipmi地址、用户名密码。pxe mac 
2. inspection node
3. 定义根磁盘
4. 定义网卡顺序
5. 定义capabilities hint
6. 定义 network-environment
7. 定义 各节点nic配置文件
8. 定义 Network-isolation 
9. 指定各节点IP地址
10. 自定义role
11. 


