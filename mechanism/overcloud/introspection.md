## Introspection原理

Introspection是指能够通过程序自动的收集服务器的物理信息，比如CPU型号，网卡个数，Mac地址，磁盘大小以及个数等等，这个功能非常有用，试想一下，在部署几十台或者更多节点时，要手动收集每一台服务器的物理信息，这该是多么痛苦的一件事，而且还很容易出错，如果能将这个过程自动化，不仅可以提高效率，减少错误，还能够实现一些高级功能，比如服务器自动发现，自动注册等等。

### 介绍

本节主要来介绍下TripleO是如何实现Introspection的，主要涉及到4个项目：

* ironic
* ironic\_python\_agent
* ironic\_inspector
* swift

这4个项目的逻辑关系如下图：

![](/assets/introspection.png)

* Ironic是对裸机进行管理的项目，可以控制裸机的整个生命周期，开机，关机，重启等等；
* Ironic-Python-Agent\(IPA\)是通过ramdisk的方式运行在裸机里的Python程序，通过让服务器从PXE启动，加载ironic\_python\_agent镜像到ramdisk，从而启动IPA的，因为它是直接在物理服务器上，所以这为收集物理机信息提供了前提条件；Ramdisk是一种将内存当做磁盘使用的技术，使得系统运行在物理机的内存里，并没有写入磁盘；IPA内部实现了插件式的架构，可以指定多个collector进行收集，满足各种收集需求；
* Ironic-Inspector是整个introspection的主体，它提供了API去触发introspection操作，并且提供回调接口给IPA用，然后将收集来的数据进行处理，并且将部分信息主动更新Ironic中的node信息；对收集来的数据进行处理，Ironic-Inspector也采取了插件式的架构，指定了一系列Hook进行处理，比如检查一些数据的有效性，重组数据，更新Ironic中的node属性等等；
* Ironic-Inspector将收集来的数据存储到Swift中；

### 操作

Introspection操作非常简单，首先需要将node置为manageable状态：

```
[stack@pre4-undercloud ~]$ openstack baremetal node manage 5a58fe7c-f04a-43dc-a9ba-48c6da1abced
```

然后执行：

```
[stack@pre4-undercloud ~]$ openstack baremetal introspection start 5a58fe7c-f04a-43dc-a9ba-48c6da1abced
```

这样就触发了一个introspection操作，introspection是一个异步操作，可以通过下面的命令查询当前的introspection状态：

```
[stack@pre4-undercloud ~]$ openstack baremetal introspection status 5a58fe7c-f04a-43dc-a9ba-48c6da1abced
+----------+-------+
| Field    | Value |
+----------+-------+
| error    | None  |
| finished | True  |
+----------+-------+
```

等introspection完成后，可以对比下前后该node的属性变化：

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

可以看到node中自动填充上了一些内存，cpu，磁盘大小的属性。可以通过如下命令查看收集的该node的所有信息：

```
[stack@pre4-undercloud ~]$ openstack baremetal introspection data save 5a58fe7c-f04a-43dc-a9ba-48c6da1abced
```

因为收集来的数据是保存在Swift里的，该命令就是从swift中下载下来的。

TripleO也提供了批量introspection的操作，但是因为ironic-inspector只提供了对一台机器进行introspection的接口，所以批量操作被集成到了Mistral的workflow中，该命令如下：

```
[stack@pre4-undercloud ~]$ openstack baremetal introspection bulk start
```

### 数据流

可以看到introspection就一条命令就完成了，这条命令背后发生了什么？我们来梳理下它的数据流，该过程大概分为2个阶段：

#### 第一阶段
客户端发出命令后，该命令就是简单调用了ironic-inspector中的`POST /v1/introspection/<node_id>`接口，其数据流如下：

![](/assets/introspection_1.png)

ironic-inspector向ironic发送了两条指令，一个是设置该node从PXE启动，一个是重启该node，这样这个命令就执行完了。

#### 第二阶段
node重启之后，从PXE启动，IPA被加载进ramdisk开始执行，其内部数据流如下：

![](/assets/introspection_2.png)

IPA通过collector插件收集本机的物理信息，收集完成之后，回调由ironic-inspector提供的回调接口：`POST /v1/continue` 将收集到的数据回传给ironic-inspector，ironic-inspector收到数据之后，就开始通过Hook去处理数据，Hook的执行分为两个阶段，在`_run_pre_hooks`中主要进行一些检查工作，在`_run_post_hooks`中主要是处理数据，然后更新ironic中的node属性，随后将处理之后的数据保存到Swift中，然后调用ironic接口将该node关机，完成introspection。


