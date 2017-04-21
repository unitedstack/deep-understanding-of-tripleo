## TripleO-Heat-Templates解析

TripleO的Heat Template是整个TripleO的核心，它定义了一系列复杂的模板，运用了Heat中一些高级的语法，抽象出了一个个嵌套的stack，来完成OverCloud中安装系统，网络配置，服务配置，高可用等一系列部署操作。本节重点解析下这些Template，讲解下TripleO中是如何使用Heat来安装部署OverCloud的。

### Heat基础知识

在TripleO中使用到了很多Heat中的语法，需要理解这些基本的语法才能够理解Heat Template。Heat中自定义了一套描述式语言，称为HOT\(Heat Orchestration Template\)，用来描述整个软件栈，包括如何创建资源，如何配置资源等等，HOT中描述的软件栈被抽象成`stack`，可以定义输入，输出，参数，资源等等，一个stack还可以嵌套其他stack，Heat默认集成了OpenStackh很多组件，可以通过HOT来管理OpenStack资源，此外，还集成了Puppet/Ansible等配置管理工具，形成了一套完整的体系，功能十分强大，这里就主要介绍下部署TripleO过程中用到的Heat一些基础知识。

#### 什么是Stack以及嵌套Stack

一个Stack可以说是一个模板描述的软件栈的一个实例，在一个Stack中定义了输入，输出，以及资源这3个主要属性，给出特定的输入，这个Stack会创建定义的资源，然后给出特定的输出，一个包含了输入，输出以及资源的模板，就可以实例化出一个Stack。在TripleO里，充分运用了嵌套stack的概念，只有理解什么是一个Stack，才能够更好的界定TripleO的中定义的模板。比如有如下模板：

```
[stack@pre4-undercloud demos]$ cat test_security.yaml
heat_template_version: 2016-10-14

parameters:
  CloudName:
    default: overcloud.localdomain
    description: the cloud name
    type: string


resources:
  HorizonSecurity:
    type: OS::Heat::RandomString

  NovaSecurity:
    type: ./test_nova_security.yaml


outputs:
  SecurityOutput:
    description: some descs
    value:
      HorizonSecurityOutput: {get_attr: [HorizonSecurity, value]}
      NovaSecurityOutput: {get_attr: [NovaSecurity, NovaSecurityOutput]}

  SecurityStringOutput:
    description: some desc
    value:
      str_replace:
        template: This is HorizonSecurityOutput and NovaSecurityOutput in CloudName
        params:
          HorizonSecurityOutput: {get_attr: [HorizonSecurity, value]}
          NovaSecurityOutput: {get_attr: [NovaSecurity, NovaSecurityOutput]}
          CloudName: {get_param: CloudName}
```

```
[stack@pre4-undercloud demos]$ cat test_nova_security.yaml
heat_template_version: 2016-10-14

resources:
  NovaSecurity:
    type: OS::Heat::RandomString


outputs:
  NovaSecurityOutput:
    description: some descs
    value: {get_attr: [NovaSecurity, value]}
```

test\_security.yaml嵌套了test\_nova\_security.yaml，两个模板都定义了输入，输出以及资源，该例子中使用的资源类型是Heat中定义的生成随机字符串，使用如下的命令可以从模板创建stack:

```
openstack stack create -t test_security.yaml test_security
```

创建好之后，查看创建的stack:

```
[stack@pre4-undercloud demos]$ openstack stack list
+--------------------------------------+---------------+-----------------+----------------------+--------------+
| ID                                   | Stack Name    | Stack Status    | Creation Time        | Updated Time |
+--------------------------------------+---------------+-----------------+----------------------+--------------+
| b3c8a10f-9b7e-49a2-a150-4fa2078c1093 | test_security | CREATE_COMPLETE | 2017-04-20T14:34:39Z | None         |
+--------------------------------------+---------------+-----------------+----------------------+--------------+
```

加上--nested可以查看所有嵌套的stack:

```
[stack@pre4-undercloud demos]$ openstack stack list --nested
+--------------------------------+--------------------------------+-----------------+----------------------+--------------+----------------------------------+
| ID                             | Stack Name                     | Stack Status    | Creation Time        | Updated Time | Parent                           |
+--------------------------------+--------------------------------+-----------------+----------------------+--------------+----------------------------------+
| 9844af34-1dcc-489a-80db-       | test_security-NovaSecurity-    | CREATE_COMPLETE | 2017-04-20T14:34:40Z | None         | b3c8a10f-9b7e-                   |
| 953f76baea4a                   | fl2f6un4mvpt                   |                 |                      |              | 49a2-a150-4fa2078c1093           |
| b3c8a10f-9b7e-                 | test_security                  | CREATE_COMPLETE | 2017-04-20T14:34:39Z | None         | None                             |
| 49a2-a150-4fa2078c1093         |                                |                 |                      |              |                                  |
+--------------------------------+--------------------------------+-----------------+----------------------+--------------+----------------------------------+
```

可以查看某个stack的输出：

```
[stack@pre4-undercloud demos]$ openstack stack output show test_security --all
+----------------------+------------------------------------------------------------------------------------------------------------------------------+
| Field                | Value                                                                                                                        |
+----------------------+------------------------------------------------------------------------------------------------------------------------------+
| SecurityStringOutput | {                                                                                                                            |
|                      |   "output_value": "This is zwinXyb5q12VTMSNyEnyfyGBL6mWwbLF and qa31seo3eLE9UJTfegmaPhUfIqJXDiQ7 in overcloud.localdomain",  |
|                      |   "output_key": "SecurityStringOutput",                                                                                      |
|                      |   "description": "some desc"                                                                                                 |
|                      | }                                                                                                                            |
| SecurityOutput       | {                                                                                                                            |
|                      |   "output_value": {                                                                                                          |
|                      |     "HorizonSecurityOutput": "zwinXyb5q12VTMSNyEnyfyGBL6mWwbLF",                                                             |
|                      |     "NovaSecurityOutput": "qa31seo3eLE9UJTfegmaPhUfIqJXDiQ7"                                                                 |
|                      |   },                                                                                                                         |
|                      |   "output_key": "SecurityOutput",                                                                                            |
|                      |   "description": "some descs"                                                                                                |
|                      | }                                                                                                                            |
+----------------------+------------------------------------------------------------------------------------------------------------------------------+
```

```
[stack@pre4-undercloud demos]$ openstack stack output show test_security-NovaSecurity-fl2f6un4mvpt --all
+--------------------+--------------------------------------------------------+
| Field              | Value                                                  |
+--------------------+--------------------------------------------------------+
| NovaSecurityOutput | {                                                      |
|                    |   "output_value": "qa31seo3eLE9UJTfegmaPhUfIqJXDiQ7",  |
|                    |   "output_key": "NovaSecurityOutput",                  |
|                    |   "description": "some descs"                          |
|                    | }                                                      |
+--------------------+--------------------------------------------------------+
```

从以上的例子中可以看到，只要一个模板定义了输入，输出，以及资源，那它就是一个stack，一个stack的输出是保存在Heat的数据库中的，生成的output类型可以是字符串，可以是对象，还可以是数组等等。Heat中还定义了一些内置的方法，用来执行一些特定任务，比如该例子中用到的str\_template，就是定义了一个模板，然后传递了模板变量，在Heat中渲染成了一个字符串，这些内置方法只能在resources的properties字段和outputs字段使用。

知道了一个Stack的构成之后，我们重点介绍几个Resource，这几个Resource是在TripleO中频繁使用的，对理解TripleO Heat Template非常重要。

#### OS::Heat::RandomString

这个是生成一个随机字符串，这在上面的例子中已经介绍过了。

#### OS::Heat::ResourceChain

使用相同的配置创建一个或多个资源，这些资源是通过一个列表来指定的，由`resources`参数指定，此外还可以加上`concurrent`参数，并发的创建这些资源，如果不指定`concurrent`或者设置为false，那么就会按照顺序一个一个创建。

#### OS::Heat::ResourceGroup

也是创建一组配置相同的资源，但是这个和ResourceChain不同的是，它指定的是创建数量，由`count`参数指定，而不是一个列表，并且是并发创建的，不能控制顺序。它在TripleO里面用来创建Nova里面的Server，即一次性创建指定数量的节点。

#### OS::Heat::SoftwareConfig

SoftwareConfig在TripleO里面作用很重要，是用来定义配置信息的，有两个参数比较重要：一个是`config`，用来指定配置信息，比如一段脚本，或者Puppet代码，一个是`group`，用来指定是哪种类型的配置信息，如果`config`指定的是一段脚本，那么`group`就是`script`，如果`config`是一段`puppet`代码，那么`group`就是`puppet`，根据`group`的不同，在应用这段配置的时候，会相应的选择不同的策略去执行。需要注意的是`config`里指定的配置信息是一段字符串，当创建了SoftwareConfig这个resource时，并没有应用这个配置，而仅仅是把配置信息保存在了Heat的数据库中，会被其他的resource引用。

#### OS::Heat::StructuredConfig

StructuredConfig和SoftwareConfig的作用是一样的，不同的是SoftwareConfig中的config指定的是一段字符串，而StructuredConfig中的config指定的是一个结构化的数据，即是一个Map，这对一些使用YAML或者JSON作为配置语言的配置工具来说是有用的。

#### OS::Heat::SoftwareDeployment

SoftwareDeployment就是真正的将上面定义的配置信息部署到某个Server上，需要指定两个参数：一个是`config`，即对上面定义的SoftwareConfig或者是StructuredConfig的引用，一个是`server`，是对某个资源的引用，通常是一个Nova Server的ID。这里可能会有一个疑问：Heat到底是怎么把一段配置最终配置到服务器上的呢？其实创建一个SoftwareDeployment主要做了两件事：一个是在Heat的数据库中产生相应的记录，一个是会将这些配置信息上传到外部的一个Metadata服务器上，而在将要配置的服务器上，即server里，会安装相应的heat agent，这个agent会去Metadata服务器上拉取本节点的配置信息，进行配置。

#### OS::Heat::StructuredDeployment

#### OS::Heat::SoftwareDeploymentGroup

#### OS::Heat::StructuredDeploymentGroup



