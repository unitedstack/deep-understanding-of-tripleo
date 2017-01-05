## 什么是TripleO？

TripleO 是社区开发的一组基于Openstack 部署 Openstack的工具集。

![](/assets/overview.png)

先部署一个小型all in one 的openvstack，其中有keystone、horizon、glance、nova、neutron、heat、ironic。称为undercloud。然后通过undercloud 去部署一个openstack生产环境，称为overcloud。

![](/assets/logical_view.png)

TripleO 优点：

* 由社区维护，代码结构优良，文档资源丰富。TripleO基于openstack，学习成本低。
* 可以与puppet 或者 bash 脚本集成。
* 更容易与 CI & CD 系统集成。



