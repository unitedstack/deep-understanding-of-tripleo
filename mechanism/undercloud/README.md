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
                ├── puppet
                └── os-cloud-config
```

instack和os-refresh-config的具体细节请参见本章“依赖组件”一节中的内容，instack里主要应用了dib elements来定制当前系统，在instack-undercloud中，指定了多个elements来执行，主要是为后面安装undercloud做些准备工作，比如生成os-refresh-config需要的脚本，生成hieradata等等；而os-refresh-config则在安装undercloud过程中起到了整体的**编排**作用，它会去调用os-apply-config配置当前系统，跑puppet安装OpenStack组件，然后建立安装overcloud使用的网络等等；puppet是在os-refresh-config过程中被执行的，使用puppet apply在本地执行puppet代码，安装undercloud用到的各个OpenStack组件；在安装完成OpenStack各个组件后，os-refresh-config会去调用os-cloud-config提供的命令setup-neutron去建立管理网络。

下面来详细介绍下instack-undercloud安装undercloud中的主要步骤：

### 1. 生成环境变量

在安装undercloud之前，先要在stack用户的home目录下创建一个undercloud.conf配置文件，在该文件中定义了安装undercloud需要用的配置项，因为在安装时，本质上是在跑各种脚本，在脚本中会用到各种变量，这些变量的值需要从环境变量中获取，因此instack-install-undercloud命令先要将读取undercloud.conf中的配置项，然后将其转化为环境变量，方便后面的脚本使用，当然，脚本使用的环境变量不仅仅是在这个阶段生成的，在各个elements中，也定义了各种环境变量，在执行elemenets之前，会先被导出来。在安装undercloud时，生成的环境变量见附录1.

### 2. 生成Metadata配置文件

在后面的步骤中会执行os-apply-config，os-apply-config需要用到一个json格式的metadata配置文件，用来渲染模板，生成系统中的配置，这个json格式的metadata配置文件就是在这个阶段生成的，里面包括了hieradata的配置，neutron的配置，os-net-config的配置等等，该文件的保存路径为：`/var/lib/heat-cfntools/cfn-init-data`，可能是由于历史原因，是以cfn命名的，这也是os-apply-config最低优先级去找的配置文件，该文件的示例请参见附录2.

### 3. 执行instack

在instack-undercloud中指定了一些elements去执行，这些elements分别来自不同的项目，有tripleo-image-elements中定义的，有instack-undercloud中定义的，还有diskimage-builder中定义的，这些信息都被配置在一个json格式的配置文件中，执行的命令如下：

```
sudo -E instack \
    -p /usr/share/tripleo-puppet-elements:/usr/share/instack-undercloud:/usr/share/tripleo-image-elements:/usr/share/diskimage-builder/elements \
    -j /usr/share/instack-undercloud/json-files/centos-7-undercloud-packages.json
```

centos-7-undercloud-packages.json文件的内容如下：

```
[
  {
    "name": "Installation",
    "element": [
      "install-types",
      "undercloud-install",
      "enable-packages-install",
      "element-manifest",
      "puppet-stack-config"
    ],
    "hook": [
      "extra-data",
      "pre-install",
      "install",
      "post-install"
    ],
    "exclude-element": [
      "pip-and-virtualenv",
      "os-collect-config",
      "svc-map",
      "pip-manifest",
      "package-installs",
      "pkg-map",
      "puppet",
      "cache-url",
      "dib-python",
      "os-svc-install",
      "install-bin"
    ],
    "blacklist": [
      "99-refresh-completed"
    ]
  }
]
```

即指定了instack-types, undercloud-install, enable-packages-install, element-manifest, puppet-stack-config这几个elements，因为每一个elements都有依赖，所以最终处理完依赖，要执行的elements全量为：

* install-types
* element-manifest
* manifests
* source-repositories
* puppet-modules
* hiera
* enable-packages-install
* os-apply-config
* os-refresh-config
* undercloud-install
* puppet-stack-config

每一个elements都包含了一些hook，instack的配置文件指定了只执行extra-data, pre-install, install, post-install这4个hook，这些hook以及hook中的脚本请参见附录3，合并之后的hook请参见附录4。

instack通过执行这些elements，大概做了几下几件事情：

1. 从package安装各个项目的puppet代码，因为还支持从源码安装puppet代码
2. 生成puppet的入口代码：puppet-stack-config.pp，以及生成puppet hieradata：puppet-stack-config.yaml
3. 生成os-apply-config使用的模板文件
4. 生成os-refresh-config使用的脚本

### 4. 执行os-refresh-config

在上一步中已经生成了os-refresh-config所需要使用的脚本，是通过os-refresh-config这个element生成的，这些脚本分散在每一个element中，每一个element除了包含instack执行过程中的hook外，还包含了os-refresh-config和os-apply-config需要用到的hook，通过os-refresh-config和os-apply-config这两个elements将这些hook合并到一起，如下：

```
▾ os-apply-config/
  ▾ etc/
    ▾ os-net-config/
        config.json
    ▾ puppet/
      ▾ hieradata/
          CentOS.yaml
          RedHat.yaml
        hiera.yaml
  ▾ root/
      stackrc
      stackrc.oac
      tripleo-undercloud-passwords
      tripleo-undercloud-passwords.oac
  ▾ var/
    ▾ opt/
      ▾ undercloud-stack/
          masquerade
▾ os-refresh-config/
  ▾ configure.d/
      20-os-apply-config*
      30-reload-keepalived*
      40-hiera-datafiles*
      40-truncate-nova-config*
      50-puppet-stack-config*
  ▾ post-configure.d/
      10-iptables*
      80-seedstack-masquerade*
      98-undercloud-setup*
      99-refresh-completed*
```

在生成的os-refresh-config中，只包含了configure和post-configure两个hook，因此先执行configure.d中的脚本，然后执行post-configure.d中的脚本，其中比较重要的是如下几个脚本：

* 在20-os-apply-config中执行了os-apply-config命令，将os-apply-config中的使用json metadata文件渲染模板，然后将生成的配置文件放置到对应的位置上去
* 50-puppet-stack-config就是执行puppet代码了，是通过puppet apply的方式执行：

  ```
  puppet apply --detailed-exitcodes /etc/puppet/manifests/puppet-stack-config.pp
  ```

  这一步就是安装undercloud需要使用到的各个OpenStack组件了。

* 98-undercloud-setup，在安装好OpenStack组件之后，调用os-cloud-config提供的命令setup-neutron去创建安装OverCloud需要使用的管理网络。此外，如果enable了mistral功能，还会去创建workbook。

除此之外，就是配置一些iptables规则，比如允许ip forwarding，可以让overcloud节点能访问外网，配置169.254.169.254的NAT规则，打通虚拟机访问metadata的通道等等。

经过这几步，就完成了undercloud的安装，整体来看undercloud采用了脚本+puppet的方式进行安装，安装过程非常复杂，定制化主要也是写elements，需要非常了解其中的原理才能定制undercloud，在现在的master分支，也就是Pike版本的TripleO中，采用了新的方法去安装undercloud，即也采用heat去部署，见[这里](https://github.com/openstack/python-tripleoclient/commit/5f58088ff52636724d53f5b0590eefb8de55434c)，不再依赖instack-undercloud中的各种elements，这样undercloud和overcloud的安装方法就统一了。

