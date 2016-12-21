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

附件：

