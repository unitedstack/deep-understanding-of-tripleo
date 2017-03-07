# UnderCloud安装原理

本节介绍UnderCloud的安装原理，UnderCloud主要是依赖于OpenStack本身的功能来安装OverCloud，OverCloud就是最终要交付的OpenStack环境。UnderCloud是一个单机版的OpenStack环境，主要安装的组件有nova, glance, keystone, neutron, heat, ironic, swift，还可以选择性的安装telemetry, mistral, zaqar等组件。

安装undercloud只使用一个简单的命令: `openstack undercloud install`就可以了，这个命令是python-tripleoclient提供的，它是python-openstackclient的一个插件，除了有安装undercloud的命令外，还有安装overcloud的命令，而在安装undercloud时，python-tripleoclient只是简单的调用了instack-undercloud中提供的命令: `instack-install-undercloud`，instack-undercloud就是专门用来安装和升级undercloud的，在instack-undercloud中，又使用了 instack + os-refresh-config + puppet来部署undercloud，所以这些组件的依赖关系如下：

```
.
└── python-openstackclient
    └── python-tripleoclient
        └── instack-undercloud
            ├── instack
            └── os-refresh-config
                ├── os-apply-config
                └── puppet
```

instack和os-refresh-config的具体细节请参见本章“依赖组件”一节中的内容，instack里主要应用了dib elements来定制当前系统，在instack-undercloud中，指定了多个elements来执行，主要是为后面安装undercloud做些准备工作，比如生成os-refresh-config需要的脚本，生成hieradata等等；而os-refresh-config则在安装undercloud过程中起到了整体的编排作用，它会去调用os-apply-config配置当前系统，跑puppet安装OpenStack组件，然后建立安装overcloud使用的网络等等；puppet是在os-refresh-config过程中被执行的，使用puppet apply在本地执行puppet代码，安装undercloud用到的各个OpenStack组件。

下面来详细介绍下instack-undercloud安装undercloud中的主要步骤：

### 生成环境变量

在安装undercloud之前，先要在stack用户的home目录下创建一个undercloud.conf配置文件，在该文件中定义了安装undercloud需要用的配置项，因为在安装时，本质上是在跑各种脚本，在脚本中会用到各种变量，这些变量的值需要从环境变量中获取，因此instack-install-undercloud命令先要将读取undercloud.conf中的配置项，然后将其转化为环境变量，方便后面的脚本使用，当然，脚本使用的环境变量不仅仅是在这个阶段生成的，在各个elements中，也定义了各种环境变量，在执行elemenets之前，会先被导出来。



