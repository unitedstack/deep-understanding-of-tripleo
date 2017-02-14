# undercloud物理环境部署

---
## 1. 修改主机名
```
$ hostname # 查看基础的主机名
$ hostname -f # 查看完成的主机名
```
如果当前的主机名不是你想要的，你可以自己定义你的undercloud的主机名:

```
[root@zhaozhilong ~]# hostnamectl set-hostname director.ustack.com
[root@zhaozhilong ~]# hostnamectl set-hostname --transient director.ustack.com
```

在`/etc/hosts` 中添加 当前主机名的解析,一定要有短域名解析，否则mq等服务会起不来。
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

127.0.0.1   undercloud.ustack.com undercloud 
```


## 2. 创建undercloud的部署用户

```
[root@director ~]# useradd stack
[root@director ~]# passwd stack # specify a password
```
上面创建了这个用户，然后我们就需要赋予这个用户sudo的权限
```
[root@director ~]# echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
[root@director ~]# chmod 0440 /etc/sudoers.d/stack
```
尝试使用stack这个用户进行登陆
```
[root@director ~]# su - stack
[stack@director ~]$
```

## 3. 配置yum源

我们这边使用ustack公司内部源:
```
[openstack-newton]
name = openstack-newton
baseurl = http://tripleO.ustack.com/repo/openstack-newton
gpgcheck = 0
priority=1
```
你也可以使用Centos社区的源:
```
[tripleO-centos]
name = tripleO-centos
http://mirror.centos.org/centos/7/cloud/x86_64/openstack-newton/
gpgcheck = 0
priority=1
```

**！！ 请务必确保openstack-newton源的priority=1**



## 4. 更新undercloud
安装 yum-plugin-priorities
```
sudo yum -y install yum-plugin-priorities
```

为了确保undercloud的kernel版本和上游版本一直，还有一些边缘组件也需要和上游同步，我们需要更新我们的系统：
```
[stack@director ~]$ sudo yum update -y
```
完成之后，就可以重启系统了
```
[stack@director ~]$ sudo reboot
```

## 5. 安装tripleO
上面的过程都是在准备我们的基础环境，现在我们需要安装我们的tripleO.
```
[stack@director ~]$ sudo yum install -y python-tripleoclient
```


## 6. 编写undercloud的配置文件
创建undercloud配置文件，并修改里面的配置。

```
$ cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf
```

然后去修改我们的配置

```
$ vim ~/undercloud.conf
[DEFAULT]
local_ip = 10.0.130.31/24
network_gateway = 10.0.130.31
undercloud_public_vip = 10.0.130.2
undercloud_admin_vip = 10.0.130.3
local_interface = em3 # pxe装机的网桥，必须和你的overcloud的pxe网卡在同一个vlan下面
network_cidr = 10.0.130.0/24
masquerade_network = 10.0.130.0/24
dhcp_start = 10.0.130.5
dhcp_end = 10.0.130.24
inspection_interface = br-ctlplane
inspection_iprange = 10.0.130.100,10.0.130.180
inspection_extras = true
undercloud_debug = true
[auth]
```

## 7. 开始部署undercloud
编写完我们的配置文件之后,我们就可以开始部署我们的undercloud了。
```
$ openstack undercloud install
```
大概15分钟，之后就可以安装成功了。




