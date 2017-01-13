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

## 7. 



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


