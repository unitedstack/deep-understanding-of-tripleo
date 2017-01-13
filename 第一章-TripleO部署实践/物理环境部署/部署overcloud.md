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


