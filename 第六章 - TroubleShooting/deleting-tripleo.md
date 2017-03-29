# Deleting TripleO

---

在某些情况下，我们可能会需要删除undercloud openstack 或者删除overcloud openstack 环境。  
那么，How？

## 删除undercloud

1. 在undercloud上删除所有openstack包
2. 删除mariadb 数据库文件
3. 删除所有ovs port 以及ovs
4. 删除/etc/swift 目录

### 删除openstack 包

```bash
$ sudo yum remove openstack*
```

### 删除mariadb 数据库文件

```bash
$ sudo rm -rf /var/lib/mysql/
```

### 删除ovs-port 、ovs bridge

列出ovs所有的port 和 bridge。
```
sudo ovs-vsctl show
```

先删除ovs-port后才能删除ovs bridge
```bash

ovs-vsctl del-port <port-name>
···

ovs-vsctl del-br <br-name>
···
```

### 删除`/etc/swift` 目录

```
rm -rf /etc/swift
```

## 删除overcloud

相比删除undercloud，overcloud的删除过程简单的多。一条命令即可
```bash
openstack stack delete overcloud --yes
```

但是，有时可能会出现无法删除stack的情况，请具体问题具体分析。通过abandon stack可以在不删除任何resource的情况下删除整个stack。
1.在`/etc/heat/heat.conf`[default]中将`enable_stack_abandon = true`这个参数设置为true
2.重启openstack-heat-engine 服务。
3.`openstack stack abandon overcloud`
4. 删除stack后手动执行nova delete、neutron port-delete、neutron net-delete。注意不要删除ctlplane的网络以及ctlplane的metadata地址。通常以.5结尾。



