#
# 自定义overcloud部署

在部署overcloud的过程中，我们一定有很多需要自定义的地方。那么TripleO是如何实现自定义参数部署的呢？我们通过下面几个章节一一说明。

# 1. 定义主机

## 1.1 分配节点

在默认部署的情况下，tripleO 会从 ironic 中拥有的同一tag的node中任意挑选一台，这会导致ironic node-list 显示的主机与nova list 显示的主机编号不一致。可以通过为每个ironic node分配特定的属性解决以上问题。

首先，为每个类型的node添加tag，例如第一台controller打上node:contoller-0 的tag，第二台controller 打上node:contoller-1的tag，以此类推。
```
ironic node-update <id> replace properties/capabilities='node:controller-0,boot_option:local'
ironic node-update <id> replace properties/capabilities='node:controller-1,boot_option:local'
...

ironic node-update <id> replace properties/capabilities='node:compute-0,boot_option:local'
ironic node-update <id> replace properties/capabilities='node:compute-1,boot_option:local'
...

ironic node-update <id> replace properties/capabilities='node:ceph-storage-0,boot_option:local'
ironic node-update <id> replace properties/capabilities='node:ceph-storage-1,boot_option:local'
...
```

创建Heat Environment file ：
**scheduler_hints_env.yaml**
```
parameter_defaults:
ControllerSchedulerHints:
'capabilities:node': 'controller-%index%'
NovaComputeSchedulerHints:
'capabilities:node': 'compute-%index%'
CephStorageSchedulerHints:
'capabilities:node': 'ceph-storage-%index%'
```

在部署时使用这个Environment File,同时每个节点的flavor要选择baremetal。
```
openstack overcloud deploy ....
  -e scheduler_hints_env.yaml \
  --control-flavor baremetal  \
  --compute-flavor baremetal \
  --ceph-storage-flavor baremetal
  ...
```


共有这些Hints可以选择。
```
ControllerSchedulerHints for Controller nodes.
NovaComputeSchedulerHints for Compute nodes.
BlockStorageSchedulerHints for Block Storage nodes.
ObjectStorageSchedulerHints for Object Storage nodes.
CephStorageSchedulerHints for Ceph Storage nodes.
[ROLE]SchedulerHints for custom roles. Replace [ROLE] with the role name.
```




## 1.2 分配主机名



## 1.3 分配主机IP

REF:[controlling-node-placement](https://access.redhat.com/documentation/en/red-hat-openstack-platform/10/paged/advanced-overcloud-customization/chapter-8-controlling-node-placement)

# 2.隔离网络

# 2. 部署代码自定义

# 3.


