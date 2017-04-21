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

每个服务对应的stack中的output字段都定义了role_data值，role_data是一个dict对象，该对象包含了以下几个属性：

* service_name，服务的名称
* monitoring_subscription，监控信息
* config_settings，该服务的配置信息
* service_config_settings，该服务所依赖的服务的配置信息，比如keystone, mysql
* step_config，该服务的puppet配置入口

在TripleO中的服务配置是采用Puppet+Hieradata的方式进行配置的，每个服务对应的stack中的role_data中的config_settings和service_config_settings最终会被转换成hieradata中的配置，然后step_config中的puppet入口代码最终会被整合到服务器中的puppet代码入口中，然后采用puppet apply的方式运行，完成该服务的配置。

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



#### 配置阶段





