## OverCloud部署解析

把物理网络配置好，镜像以及裸机信息录入系统，设置好Metadata，一切准备好之后，就可以开始部署OverCloud了，部署OverCloud非常简单，只需要执行一条命令：

```
openstack overcloud deploy --templates /usr/share/openstack-tripleo-heat-templates/ \
  -r /home/stack/templates/roles/roles_data.yaml \
  -e /home/stack/templates/environments/low-memory-usage.yaml \
  -e /home/stack/templates/environments/environment.yaml
```

这条命令背后到底发生了什么？在前面概览中介绍了部署OverCloud的过程其实是在跑Heat Template的过程，反映到Heat里，就是要创建一个stack，所以这个命令做的大部分工作都是在为创建这个stack做准备，创建环境变量，解析Template，生成自定义的Role，传递各种参数，然后创建stack，整体的数据流如下图：

![](/assets/overcloud_dataflow.png)

在部署OverCloud之前，首先会生成一个Plan，在前面介绍过，Heat的模板是在`tripleo-heat-templates`这个项目中定义的，这个里面有一些模板里面是有模板变量的，比如自定义的Role，需要根据参数进行解析，这些模板被解析完之后，会被上传到Swift里一个以将要创建的Heat Stack的名字命名的container里，默认为`overcloud`，在创建Heat Stack时，Heat所用到的模板，就是直接从Swift里获取的，所以这个过程大致如下：

* 客户端先调用Mistral里面的`tripleo.plan.create_container`这个workflow，在Swift里创建了一个名称为`overcloud`的container；
* 然后客户端直接调用swift接口，将原始模板上传到这个container里；
* 然后开始执行`tripleo.plan_management.v1.create_deployment_plan`这个workflow，这个workflow定义了3个action：
  * `tripleo.plan.create`，将`tripleo-heat-templates`里的`root_template`和`root_environment`存储到Mistral的Environment里，所谓`root_template`就是`overcloud.yaml`这个文件，它是创建这个stack里的最上层的stack，这个stack里又嵌套了非常多的stack，每一个stack都负责做一块事情；`root_environment`则是指的`overcloud-resource-registry-puppet.yaml`这个文件，这个文件是Heat里用到的Environment，定义了Heat里的Resource对应的模板文件；
  * `tripleo.parameters.generate_passwords`，这个action生成了各个服务的密码，即每个服务都会在Keystone里生成一个账户，后面会被配置到各个服务的配置文件中，比如Nova会创建一个名为nova的用户，它的密码就是在这个阶段生成的，这些密码也被上传到了Mistral的Environment里；
  * `tripleo.templates.process`，这个action主要是处理模板，根据指定的变量渲染模板，生成本次部署特定的模板，比如生成自定义的Role，此外，再结合前面的几个action生成的结果，这个action最终生成了创建stack需要使用的参数；
* 上面的准备工作做好之后，就开始执行`tripleo.deployment.v1.deploy_plan`这个workflow，这个workflow就是要实施前面定义的Plan了，其实就是调用heat，结合在这个Plan中定义的模板以及环境变量，开始创建这个stack了，部署整个overcloud的过程就是创建这个stack的过程，这个stack又嵌套了非常多的子stack，比如创建裸机，部署系统，配置系统，这些都被封装成了stack，通过执行这一个个stack，来完成部署过程。
* 等Heat Stack创建完成之后，会上传overcloudrc文件到各个服务器，整个部署过程就完成了。

可以看到，创建stack的过程，也就是`tripleo-heat-templates`这个项目，是整个TripleO的最核心部分，它决定了最终部署出来的集群是什么样子，应该如何配置各个服务，后面我们会专门介绍下TripleO的Template。