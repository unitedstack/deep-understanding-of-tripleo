# 理解TripleO

---


## 什么是TripleO?

TripleO is a project aimed at installing, upgrading and operating OpenStack clouds using OpenStack’s own cloud facilities as the foundation - building on Nova, Ironic, Neutron and Heat to automate cloud management at datacenter scale.

TriplO 是使用openstack(undercloud)来部署、升级、运维openstack(overcloud)的一个项目。

tripleO是社区推出的OpenStack部署工具，目前RedHat主推的OpenStack部署工具，已经发了4个版本。相对来说，tripleO比起其他社区的部署工具更加的灵活方便。TripelO部署时需要先准备一个OpenStack控制器的镜像，然后用Ironic再去部署裸机，再通过heat在裸机上部署OpenStack。架构上分为undercloud和overcloud，基本的部署概念如下：

![](/assets/ARCHITECTURE-1.png)

我们在部署的时候，先安装一个种子节点，也就是我们的undercloud，然后再让undercloud去部署overcloud。

## TripleO的核心组件

我们来梳理一下大概的tripleO用到的组件

### Puppet

在做线上变更，或者部署服务的时候，都需要用到Puppet，所以说这个老牌的DevOps工具还是需要我们仔细的去研究的。具体的Puppet技术细节和使用方法，可以参考：[《DevOps运维基础-Puppet》](/(technology/puppet/README.md)。

### Ironic

在部署裸机的时候，我们需要用到Ironic，在Ironic里面注册的主机，在nova装机的时候会把裸机当成虚拟机来进行装机，这个步骤是通过逻辑的IPMI进行控制的。

### Heat

在实际部署的时候，我们往往需要一次性部署好几个节点。这时候就需要一个编排工具，包括线上变更和集群部署都需要Heat的参与，所以说Heat在TripleO中非常重要。

## 小结

TripleO的实际部署细节文档可参考官方文档，官方文档的一些细节都讲的非常的到位：

* [《TripleO的官方部署》](http://docs.openstack.org/developer/tripleo-docs/)

当然，TripleO还有quickstart模块，专门用来安装undercloud的：

* [《TripleO的Quickstart》](http://docs.openstack.org/developer/tripleo-quickstart/)



