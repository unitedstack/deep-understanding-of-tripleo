# TripleO-Quickstart介绍

[TripleO-Quckstart](https://github.com/openstack/tripleo-quickstart)是一个用来快速搭建TripleO测试环境的Ansible程序。在部署生产环境时，会用到[instack-undercloud](https://github.com/openstack/instack-undercloud)这个项目来部署undercloud，在该项目中有一个叫做[instack-virt-setup](https://github.com/openstack/instack-undercloud/blob/master/scripts/instack-virt-setup)的脚本，用来搭建测试tripleo用的虚拟环境，该脚本在Ocata版本将会被废弃，tripleo-quickstart就是用来取代它的，tripleo-quickstart通过一系列playbook，不仅可以用来搭建虚拟环境，还能在其上部署undercloud和overcloud，而且支持多种部署模式，部署完成之后，还能跑测试检验结果，已经形成了一个完整的体系，而且使用方便，现在已经被社区用来跑TripleO的CI程序。

和tripleo-quickstart相关的还有一个叫做[tripleo-quickstart-extras](https://github.com/openstack/tripleo-quickstart-extras)的项目，它是tripleo-quickstart功能的扩展，其实tripleo-quickstart本身只包含搭建虚拟测试环境的功能，包括环境检查，网络/存储/虚拟机的建立等等，而具体的搭建undercloud，overcloud的功能则是在tripleo-quickstart-extras中实现的，还包括测试检查，这些功能都被抽象成了ansible中的role。其实tripleo-quickstart-extras对tripleo-quickstart是没有依赖关系的，只要有环境，tripleo-quickstart-extras就用来部署undercloud/overcloud，不论是否是虚拟环境，因此在部署生产环境时也可以使用tripleo-quickstart-extras。其实本身tripleo的部署步骤就不复杂，tripleo-quickstart-extras只是将其中一些需要手动执行的命令编排了一下，让部署变得更加简单高效了。

TripleO-Quickstart

