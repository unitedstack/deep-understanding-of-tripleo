什么是TripleO？
---

TripleO 是社区开发的一组基于Openstack 部署 Openstack的工具集。

![](/assets/overview.png)

先部署一个小型all in one 的openstack，其中有keystone、horizon、glance、nova、neutron、heat、ironic。称为undercloud。然后通过undercloud 去部署一个真正的openstack环境，称为overcloud。
![](/assets/logical_view.png)

