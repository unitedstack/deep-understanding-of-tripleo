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



