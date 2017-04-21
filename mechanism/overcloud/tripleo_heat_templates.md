## TripleO-Heat-Templates解析

TripleO的Heat Template是整个TripleO的核心，它定义了一系列复杂的模板，运用了Heat中一些高级的语法，抽象出了一个个嵌套的stack，来完成OverCloud中安装系统，网络配置，服务配置，高可用等一系列部署操作。本节重点解析下这些Template，讲解下TripleO中是如何使用Heat来安装部署OverCloud的，该项目的地址在[tripleo-heat-templates](https://github.com/openstack/tripleo-heat-templates)。如果对Heat的基本语法不熟悉的，请参考[Heat基础知识](/mechanism/dependencies/heat_basics.md)小节。



