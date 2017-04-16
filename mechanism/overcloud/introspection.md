## Introspection原理


### 概述

Introspection是指能够通过程序自动的收集服务器的物理信息，比如CPU型号，网卡个数，Mac地址，磁盘大小以及个数等等，这个功能非常有用，试想一下，在部署几十台或者更多节点时，要手动收集每一台服务器的物理信息，这该是多么痛苦的一件事，而且还很容易出错，如果能将这个过程自动化，不仅可以提高效率，减少错误，还能够实现一些高级功能，比如服务器自动发现，自动注册等等。

本节主要来介绍下TripleO是如何实现Introspection的，主要涉及到4个项目：

* ironic
* ironic_python_agent
* ironic_inspector
* swift

这4个项目的逻辑关系如下图：

![](/assets/introspection.png)

* Ironic是对裸机进行管理的项目，可以控制裸机的整个生命周期，开机，关机，重启等等；
* Ironic-Python-Agent(IPA)是通过ramdisk的方式运行在裸机里的Python程序，通过让服务器从PXE启动，加载ironic_python_agent镜像到ramdisk，从而启动IPA的，因为它是直接在物理服务器上，所以这为收集物理机信息提供了前提条件；Ramdisk是一种将内存当做磁盘使用的技术，使得系统运行在物理机的内存里，并没有写入磁盘；IPA内部实现了插件式的架构，可以指定多个collector进行收集，满足各种收集需求；
* Ironic-Inspector是整个introspection的主体，它提供了API去触发introspection操作，并且提供回调接口给IPA用，然后将收集来的数据进行处理，并且将部分信息主动更新Ironic中的node信息；对收集来的数据进行处理，Ironic-Inspector也采取了插件式的架构，指定了一系列Hook进行处理，比如检查一些数据的有效性，重组数据，更新Ironic中的node属性等等；
* Ironic-Inspector将收集来的数据存储到Swift中；

### 
将裸机注册到Ironic中后，就可以进行Introspection了，非常简单。

首先先要将node置为manageable状态：

```
openstack baremetal node manage 5a58fe7c-f04a-43dc-a9ba-48c6da1abced
```
然后执行：

```
openstack baremetal introspection start 5a58fe7c-f04a-43dc-a9ba-48c6da1abced
```



```
[stack@pre4-undercloud templates]$ openstack baremetal node show 5a58fe7c-f04a-43dc-a9ba-48c6da1abced
+------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                  | Value                                                                                                                                                                                                                   |
+------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| console_enabled        | False                                                                                                                                                                                                                   |
| created_at             | 2017-04-16T07:17:40+00:00                                                                                                                                                                                               |
| driver                 | pxe_ipmitool                                                                                                                                                                                                            |
| driver_info            | {u'deploy-ramdisk': u'0a601990-89eb-4678-b460-ebbae1be54d1', u'ipmi_address': u'10.0.108.120', u'ipmi_username': u'root', u'deploy_kernel': u'38b34314-e3a9-410b-b435-8495b110ab41', u'ipmi_password': u'******'}       |
| driver_internal_info   | {}                                                                                                                                                                                                                      |
| extra                  | {}                                                                                                                                                                                                                      |
| inspection_finished_at | None                                                                                                                                                                                                                    |
| inspection_started_at  | None                                                                                                                                                                                                                    |
| instance_info          | {}                                                                                                                                                                                                                      |
| instance_uuid          | None                                                                                                                                                                                                                    |
| last_error             | None                                                                                                                                                                                                                    |
| maintenance            | False                                                                                                                                                                                                                   |
| maintenance_reason     | None                                                                                                                                                                                                                    |
| name                   | rack2-4-compute                                                                                                                                                                                                         |
| ports                  | [{u'href': u'http://10.0.161.2:6385/v1/nodes/5a58fe7c-f04a-43dc-a9ba-48c6da1abced/ports', u'rel': u'self'}, {u'href': u'http://10.0.161.2:6385/nodes/5a58fe7c-f04a-43dc-a9ba-48c6da1abced/ports', u'rel': u'bookmark'}] |
| power_state            | power off                                                                                                                                                                                                               |
| properties             | {}                                                                                                                                                                                                                      |
| provision_state        | manageable                                                                                                                                                                                                              |
| provision_updated_at   | 2017-04-16T07:32:02+00:00                                                                                                                                                                                               |
| reservation            | None                                                                                                                                                                                                                    |
| target_power_state     | None                                                                                                                                                                                                                    |
| target_provision_state | None                                                                                                                                                                                                                    |
| updated_at             | 2017-04-16T07:32:02+00:00                                                                                                                                                                                               |
| uuid                   | 5a58fe7c-f04a-43dc-a9ba-48c6da1abced                                                                                                                                                                                    |
+------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```


```
[stack@pre4-undercloud templates]$ openstack baremetal node show 5a58fe7c-f04a-43dc-a9ba-48c6da1abced
+------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                  | Value                                                                                                                                                                                                                   |
+------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| console_enabled        | False                                                                                                                                                                                                                   |
| created_at             | 2017-04-16T07:17:40+00:00                                                                                                                                                                                               |
| driver                 | pxe_ipmitool                                                                                                                                                                                                            |
| driver_info            | {u'deploy-ramdisk': u'0a601990-89eb-4678-b460-ebbae1be54d1', u'ipmi_address': u'10.0.108.120', u'ipmi_username': u'root', u'deploy_kernel': u'38b34314-e3a9-410b-b435-8495b110ab41', u'ipmi_password': u'******'}       |
| driver_internal_info   | {}                                                                                                                                                                                                                      |
| extra                  | {u'hardware_swift_object': u'extra_hardware-5a58fe7c-f04a-43dc-a9ba-48c6da1abced'}                                                                                                                                      |
| inspection_finished_at | None                                                                                                                                                                                                                    |
| inspection_started_at  | None                                                                                                                                                                                                                    |
| instance_info          | {}                                                                                                                                                                                                                      |
| instance_uuid          | None                                                                                                                                                                                                                    |
| last_error             | None                                                                                                                                                                                                                    |
| maintenance            | False                                                                                                                                                                                                                   |
| maintenance_reason     | None                                                                                                                                                                                                                    |
| name                   | rack2-4-compute                                                                                                                                                                                                         |
| ports                  | [{u'href': u'http://10.0.161.2:6385/v1/nodes/5a58fe7c-f04a-43dc-a9ba-48c6da1abced/ports', u'rel': u'self'}, {u'href': u'http://10.0.161.2:6385/nodes/5a58fe7c-f04a-43dc-a9ba-48c6da1abced/ports', u'rel': u'bookmark'}] |
| power_state            | power off                                                                                                                                                                                                               |
| properties             | {u'memory_mb': u'65536', u'cpu_arch': u'x86_64', u'local_gb': u'277', u'cpus': u'40', u'capabilities': u'cpu_txt:true,cpu_aes:true,cpu_hugepages_1g:true,cpu_hugepages:true,cpu_vt:true'}                               |
| provision_state        | manageable                                                                                                                                                                                                              |
| provision_updated_at   | 2017-04-16T07:32:02+00:00                                                                                                                                                                                               |
| reservation            | None                                                                                                                                                                                                                    |
| target_power_state     | None                                                                                                                                                                                                                    |
| target_provision_state | None                                                                                                                                                                                                                    |
| updated_at             | 2017-04-16T07:38:57+00:00                                                                                                                                                                                               |
| uuid                   | 5a58fe7c-f04a-43dc-a9ba-48c6da1abced                                                                                                                                                                                    |
+------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

