## TripleO-Heat-Templates解析

TripleO的Heat Template是整个TripleO的核心，它定义了一系列复杂的模板，运用了Heat中一些高级的语法，抽象出了一个个嵌套的stack，来完成OverCloud中安装系统，网络配置，服务配置，高可用等一系列部署操作。本节重点解析下这些Template，讲解下TripleO中是如何使用Heat来安装部署OverCloud的，该项目的地址在[tripleo-heat-templates](https://github.com/openstack/tripleo-heat-templates)。如果对Heat的基本语法不熟悉的，请参考[Heat基础知识](/mechanism/dependencies/heat_basics.md)小节。

在[OverCloud部署解析](/mechanism/overcloud/overcloud_deploy.md)小节讲过，部署OverCloud只需要一条命令就可以：

```
openstack overcloud deploy --templates /usr/share/openstack-tripleo-heat-templates/ \
  -r /home/stack/templates/roles/roles_data.yaml \
  -e /home/stack/templates/environments/low-memory-usage.yaml \
  -e /home/stack/templates/environments/environment.yaml
```

`openstack overcloud deploy`命令最终是要创建一个stack，这个stack对应的上层模板文件为`overcloud.yaml`，为整个模板的入口，该模板文件是由`overcloud.j2.yaml`这个模板文件渲染过来的，此外还有`overcloud-resource-registry-puppet.yaml`文件，该文件指定了Heat的环境变量，主要映射了各种自定义的resource对应的模板，该文件也是由`overcloud-resource-registry-puppet.j2.yaml`整个模板文件渲染过来的，模板变量来自于`roles_data.yaml`文件，该文件定义了要部署哪种角色的节点，以及每个角色都包含哪些服务。

下面我们以定义Controller和Compute两个角色为例，来解析这些模板的关系。创建stack的过程就是创建该stack中定义的resource的过程，而这些resource有可能又嵌套了子stack，所以这些resource的创建是有一定依赖关系的，规则一般是使用`depends_on`显示的指定，或者是某个resource会引用到另外一个resource的输出值，这样必须先创建被依赖的resource。因此整个创建过程，我们大致可以分为三个阶段：

#### 准备阶段

在准备阶段主要是创建被依赖的resource，主要有3种：

##### DefaultPasswords

如下图，主要是创建了MySQL的Root密码，RabbitMQ Cooike等resource，这会在后面的配置阶段用到。这些resource的类型都是`OS::Heat::RandomString`，都是生成的随机变量。

![](/assets/overcloud1.png)

该图有点UML的类图的概念，但不是严格意义的类图，仅仅是为了说明resource之间的关系。第一行说明的是该资源的类型，第二行是该类型资源的一个实例，第三行是该实例的属性，实心箭头表示引用，空心箭头表示依赖。

##### EndpointMap

如下图，该resource主要是创建了各个服务的endpoint，即最终要配置到keystone中的endpoint：

![](/assets/overcloud2.png)

EndpointMap这个resource依赖于Network这个resource，Network中定义了TripleO中抽象出来的各种网络，有External, InternalApi，Storage, StorageMgmt等，该resource会在Neutron中创建相应的network以及subnet。

EndpointMap中包含了各个服务的internal, public, admin这三种endpoint，被定义在一个yaml文件中，然后由一个脚本生成heat的template，因为该heat的template较为复杂，所以写了一个脚本进行转换，该yaml文件的格式为：

```
Aodh:
    Internal:
        net_param: AodhApi
    Public:
        net_param: Public
    Admin:
        net_param: AodhApi
    port: 8042
```

##### ServiceChain

ServiceChain主要是在Heat中创建各个服务的stack，如下图：  
![](/assets/overcloud3.png)

每个服务对应的stack中的output字段都定义了role\_data值，role\_data是一个dict对象，该对象包含了以下几个属性：

* service\_name，服务的名称
* monitoring\_subscription，监控信息
* config\_settings，该服务的配置信息
* service\_config\_settings，该服务所依赖的服务的配置信息，比如keystone, mysql
* step\_config，该服务的puppet配置入口

在TripleO中的服务配置是采用Puppet+Hieradata的方式进行配置的，每个服务对应的stack中的role\_data中的config\_settings和service\_config\_settings最终会被转换成hieradata中的配置，然后step\_config中的puppet入口代码最终会被整合到服务器中的puppet代码入口中，然后采用puppet apply的方式运行，完成该服务的配置。

下面为AodhApi这个服务的service模板：

```
outputs:
  role_data:
    description: Role data for the Aodh API service.
    value:
      service_name: aodh_api
      monitoring_subscription: {get_param: MonitoringSubscriptionAodhApi}
      config_settings:
        map_merge:
          - get_attr: [AodhBase, role_data, config_settings]
          - get_attr: [ApacheServiceBase, role_data, config_settings]
          - aodh::wsgi::apache::ssl: false
            aodh::wsgi::apache::servername:
              str_replace:
                template:
                  '"%{::fqdn_$NETWORK}"'
                params:
                  $NETWORK: {get_param: [ServiceNetMap, AodhApiNetwork]}
            aodh::api::service_name: 'httpd'
            aodh::api::enable_proxy_headers_parsing: true
            tripleo.aodh_api.firewall_rules:
              '128 aodh-api':
                dport:
                  - 8042
                  - 13042
            aodh::api::host: {get_param: [ServiceNetMap, AodhApiNetwork]}
            aodh::wsgi::apache::bind_host: {get_param: [ServiceNetMap, AodhApiNetwork]}
            tripleo::profile::base::aodh::api::enable_combination_alarms: {get_param: EnableCombinationAlarms}
      service_config_settings:
        get_attr: [AodhBase, role_data, service_config_settings]
      step_config: |
        include tripleo::profile::base::aodh::api
```

TripleO中还有一个项目，[puppet-tripleo](https://github.com/openstack/puppet-tripleo)，是一个Puppet的转发层，里面集成了所有服务的Puppet入口，如上面的`include tripleo::profile::base::aodh::api`，就是指向的aodh api的puppet入口代码，在服务器上跑该代码时，会去hieradata中找该服务的配置信息，进行配置。

#### 创建服务器阶段

在该阶段，主要是创建服务器，安装系统，并且进行系统配置，如下图：

![](/assets/overcloud4.png)

创建服务器采用`OS::Heat::ResourceGroup`资源类型，一次创建多个Nova的Server，在UnderCloud上配置的Nova的Driver是Ironic，这里会调用Ironic去创建服务器，进行装机操作。在创建服务器之前，会先创建userdata，对服务器做一些初始化操作，该模板如下：

```
  Controller:
    type: OS::TripleO::Server
    metadata:
      os-collect-config:
        command: {get_param: ConfigCommand}
    properties:
      image: {get_param: controllerImage}
      image_update_policy: {get_param: ImageUpdatePolicy}
      flavor: {get_param: OvercloudControlFlavor}
      key_name: {get_param: KeyName}
      networks:
        - network: ctlplane
      user_data_format: SOFTWARE_CONFIG
      user_data: {get_resource: UserData}
      name:
        str_replace:
            template: {get_param: Hostname}
            params: {get_param: HostnameMap}
      software_config_transport: {get_param: SoftwareConfigTransport}
      metadata: {get_param: ServerMetadata}
      scheduler_hints: {get_param: ControllerSchedulerHints}
```

`OS::TripleO::Server`这个resource type会被映射成`OS::Nova::Server`，注意上面metadata字段配置的为os-collect-config，该字段在Heat里会被处理成为userdata的一部分，在系统安装好之后，会启动os-collect-config服务，并且进行配置，该服务充当了运行在每一个服务器中的agent，它会从我们前面提到过的Metadata服务器中收集Metadata配置信息，然后进行配置，即和Heat的SoftwareDeployment配合工作，它会周期性的检查Metadata服务器中的配置信息，如果有变化，则会调用相应的程序执行这些配置。

当服务器装机完成之后，会创建`UpdateDeployment`来更新系统中的包，并且配置该服务器的网络，对服务器的网络配置，定义在`NetworkConfig`中，在服务器上，会调用os-net-config来配置网卡，可以配置bond，linux-bridge等信息。

然后在`ControllerDeployment`会在服务器上生成本节点的hieradata数据，为后面的配置阶段准备好配置信息。

#### 配置阶段

在该阶段主要是跑Puppet，即对各个服务进行配置，在介绍相关模板之前，先来介绍下在服务器上，是如何进行配置的。主要使用了下面几个组件：

* os-collect-config
* os-apply-config
* os-refresh-config
* heat-agent

os-collect-config是一个周期运行的daemon进程，它的配置文件如下：

```
[DEFAULT]
command = os-refresh-config --timeout 14400
collectors = ec2
collectors = request
collectors = local

[request]
metadata_url = http://10.0.141.2:8080/v1/AUTH_c1ca7a85e40e440080a610aed86a2cdc/ov-6p7e5ekmrx7-0-si2dtcgirg4m-Controller-2gftczryrcla/1695b2a0-0245-4d04-8406-1b169920fef4?temp_url_sig=8a36df7706992216c101e8ee2c88f1287a3fa2e0&temp_url_expires=2147483586
```

在os-collect-config中定义了3个collector，起主要作用的是request这个collector，在它的section中定义了metadata\_url，它指向的是Swift中的地址，在部署时，Heat会为每一个节点在Swift中生成一个Container，该Container中保存了本节点的配置，os-collect-config使用request collector周期性的从swift中拉取配置，如果发现配置有变化，则会运行command中指定的命令，这里配置的为os-refresh-config，os-refresh-config中则指定了一系列HOOK，会按照顺序执行这些Hook，这些Hook是被直接安装好在镜像中的，存放的路径为：`/usr/libexec/os-refresh-config/configure.d`，有如下HOOK：

* 20-os-apply-config，执行os-apply-config命令，生成配置文件
* 20-os-net-config，执行os-net-config配置网络
* 25-set-network-gateway
* 40-hiera-datafiles，生成hieradata文件
* 40-truncate-nova-config
* 51-hosts，配置hosts文件
* 55-heat-config，执行heat agent

在os-apply-config主要是从模板生成配置文件，模板存放的路径为`/usr/libexec/os-apply-config/templates`：
```
[heat-admin@overcloud-novacompute-0 os-apply-config]$ tree
.
└── templates
    ├── etc
    │   ├── os-collect-config.conf
    │   ├── os-collect-config.conf.oac
    │   ├── os-net-config
    │   │   └── config.json
    │   └── puppet
    │       └── hiera.yaml
    └── var
        └── run
            └── heat-config
                └── heat-config
```
可以看到生成了/etc/os-net-config/config.json文件，该文件保存对网卡的配置，/etc/puppet/hiera.yaml文件，该文件为hieradata的配置文件，还有/var/run/heat-config/heat-config，该文件保存从heat解析出来的针对本节点的各种配置信息。

40-hiera-datafiles会从/var/run/heat-config/heat-config读取配置信息，然后生成Hieradata数据文件：
```
-rw-r--r--. 1 root root 27454 Apr 10 13:33 all_nodes.yaml
-rw-r--r--. 1 root root    75 Apr 10 13:33 bootstrap_node.yaml
-rw-r--r--. 1 root root   125 Apr 10 13:33 controller_extraconfig.yaml
-rw-r--r--. 1 root root   154 Apr 10 13:33 controller.yaml
-rw-r--r--. 1 root root   414 Apr 10 13:33 extraconfig.yaml
-rw-------. 1 root root   833 Apr 10 13:16 heat_config_ControllerDeployment_Step1.json
-rw-------. 1 root root   831 Apr 10 13:18 heat_config_ControllerDeployment_Step2.json
-rw-------. 1 root root   839 Apr 10 13:20 heat_config_ControllerDeployment_Step3.json
-rw-------. 1 root root   839 Apr 10 13:23 heat_config_ControllerDeployment_Step4.json
-rw-------. 1 root root   831 Apr 10 13:27 heat_config_ControllerDeployment_Step5.json
-rw-r--r--. 1 root root 40365 Apr 10 13:33 service_configs.yaml
-rw-r--r--. 1 root root  2275 Apr 10 13:33 service_names.yaml
-rw-r--r--. 1 root root  1652 Apr 10 13:33 vip_data.yaml
```
55-heat-config会去根据SoftwareConfig中的group信息选择不同的heat-agent执行不同的hook，如果group信息为puppet，则会从/var/run/heat-config/heat-config读取step_config信息，生成puppet代码，生成的puppet代码存放的路径为：/var/lib/heat-config/heat-config-puppet，会生成一个以该节点的node——id为名称的pp文件，然后使用puppet apply执行这个pp文件；如果group信息为script，则执行相应的脚本。调用这些程序执行的结果被保存在/var/run/heat-config/deployed目录下。

明白了配置的原理之后，来看下配置阶段的模板，如下图：

![](/assets/overcloud5.png)

这个阶段最重要的操作是执行puppet，配置各个服务，在TripleO中把Puppet的执行过程分为了5个步骤，每一个步骤依赖于前面的一个步骤执行完成，在hieradata中为每一个步骤分别生成了一个配置文件，这种把puppet分开步骤执行的设计，非常好的解决了因为各个服务的依赖问题而导致可能出现的各种问题。



