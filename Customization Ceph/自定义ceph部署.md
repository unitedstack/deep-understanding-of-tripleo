# 自定义Ceph存储部署

---


# 4. 配置 ceph osd

在stack用户目录下建立`templates`目录

```
mkdir ~/templates
```

复制ceph 配置文件到templates目录中

```
cp /usr/share/openstack-tripleo-heat-templates/environments/storage-environment.yaml ~/templates/
```

在storage-environment.yaml中添加一段ExtraConfig 的section，格式大致如下：

```
parameter_defaults: #已存在的section
ExtraConfig: #这是我们要添加的section
ceph::profile::params::osds:
```

分别存储journal 和 osd

```
ceph::profile::params::osds:
'/dev/sdc':
journal: '/dev/sdb'
'/dev/sdd':
journal: '/dev/sdb'
```

将journal 和osd 数据放在一个硬盘内

```
ceph::profile::params::osds:
'/dev/sdb': {}
'/dev/sdc': {}
'/dev/sdd': {}
```