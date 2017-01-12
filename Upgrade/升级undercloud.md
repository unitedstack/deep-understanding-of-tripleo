# M版本升级N版

---

使用 tripleO 进行大版本升级的过程也可以大致分为三个步骤：

1. 升级 undercloud
2. 升级 undercloud 中的 image
3. 升级 overcloud

但在升级之前有很多注意事项：

> **建议对要进行升级的 overcloud 节点进行备份**
>
> **在正式升级之前应该在测试环境中进行预测试，预测试环境应该与正式环境相同并且进行过相同的配置变更**
>
> **如果 overcloud 通过非 tripleO 的方式进行过管理，升级则不被支持**
>
> **在启用了虚拟机高可用的集群中，升级不被支持**
>
> **tripleO 任一版本只支持将平台从上一版本升级到当前版本，如 Newton 版本的 tripleO 仅支持将 Mitaka 版本的平台进行升级到 N 版**
>
> **请检查某些功能模块是否在具体实现方式上进行了变化，针对相应的内容需要进行数据迁移等工作**

## 升级 undercloud

升级之前，停止所有 openstack 相关服务：

```shell
$ systemctl stop 'openstack-*'
$ systemctl stop 'neutron-*'
```

升级 tripleO 相关软件包：

```shell
$ yum update python-tripleoclient
```

进行 undercloud 升级，该命令并不会删除任何数据：

```shell
$ openstack undercloud upgrade
```

升级后确认服务及相关内容的状态：

```shell
$ systemctl list-units openstack-*
$ source ~/stackrc
$ openstack server list
$ ironic node-list
$ heat stack-list
```

如果在之前的部署过程中，针对 heat 的核心模板进行了修改，请确认修改内容并修改相应的升级模板，用如下命令比对不同之处并加入到升级所使用的模板中：

```shell
$ diff -Nary /usr/share/openstack-tripleo-heat-templates/ ~/templates/my-overcloud/
```

## 升级镜像

下载新版本的镜像并进行上传升级：

```shell
$ openstack overcloud image upload --update-existing --image-path ~/images/.
```

> **注意：镜像版本与 undercloud 版本一定要向匹配**

## 升级 overcloud

在升级过程中，需要多次运行之前安装 overcloud 的命令，且每次加入一个不同的 environment 文件，这些文件包括：

1. major-upgrade-ceilometer-wsgi-mitaka-newton.yaml 

   N版中的 ceilometer 从单独的服务变成了 WSGI 方式
   
2. major-upgrade-pacemaker-init.yaml
   
   基础升级相关内容
   
3. major-upgrade-pacemaker.yaml
   
   控制节点升级相关内容
   
4. ajor-upgrade-remove-sahara.yaml
   
   可选，M 版和 N 版对 sahara 服务处理不同
   
5. major-upgrade-pacemaker-converge.yaml

   完成升级，这一步将会按照新的模板去将 overcloud 进行升级
   
6. major-upgrade-aodh-migration.yaml
   
   将 aodh 从 mongoDB 迁移至 MariaDB
   
   
更多细节可参考 Red Hat 官方指导（https://access.redhat.com/documentation/en/red-hat-openstack-platform/10/paged/upgrading-red-hat-openstack-platform/chapter-3-director-based-environments-performing-upgrades-to-major-versions）
