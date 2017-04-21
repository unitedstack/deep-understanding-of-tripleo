## TripleO-Heat-Templates解析

TripleO的Heat Template是整个TripleO的核心，它定义了一系列复杂的模板，运用了Heat中一些高级的语法，抽象出了一个个嵌套的stack，来完成OverCloud中安装系统，网络配置，服务配置，高可用等一系列部署操作。本节重点解析下这些Template，讲解下TripleO中是如何使用Heat来安装部署OverCloud的，该项目的地址在[tripleo-heat-templates](https://github.com/openstack/tripleo-heat-templates)。如果对Heat的基本语法不熟悉的，请参考[Heat基础知识](/mechanism/dependencies/heat_basics.md)小节。

在[OverCloud部署解析](/mechanism/overcloud/overcloud_deploy.md)小节讲过，部署OverCloud只需要一条命令就可以：

```
openstack overcloud deploy --templates /usr/share/openstack-tripleo-heat-templates/ \
  -r /home/stack/templates/roles/roles_data.yaml \
  -e /home/stack/templates/environments/low-memory-usage.yaml \
  -e /home/stack/templates/environments/environment.yaml
```

`openstack overcloud deploy`命令最终是要创建一个stack，这个stack对应的上层模板文件为`overcloud.yaml`，该模板文件是由`overcloud.j2.yaml`这个模板文件渲染过来的，主要是根据

