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

除此之外，就是配置一些iptables规则，比如允许ip forwarding，可以让overcloud节点能访问外网，配置169.254.169.254的NAT规则，打通虚拟机访问metadata的通道等等，最终建立出来的undercloud网络拓扑如下：![](/assets/tripleo-undercloud-topology2.png)经过上面这几步，就完成了undercloud的安装，整体来看undercloud采用了脚本+puppet的方式进行安装，安装过程非常复杂，定制化主要也是写elements，需要非常了解其中的原理才能定制undercloud，在现在的master分支，也就是Pike版本的TripleO中，采用了新的方法去安装undercloud，即也采用heat去部署，见[这里](https://github.com/openstack/python-tripleoclient/commit/5f58088ff52636724d53f5b0590eefb8de55434c)，不再依赖instack-undercloud中的各种elements，这样undercloud和overcloud的安装方法就统一了。

### 附录1

安装undercloud时，生成的环境变量示例：

```
{  
   'UNDERCLOUD_HORIZON_SECRET_KEY':'51a4c2aee7cfa93efee549c2a1bd48e9a3501494',
   'TARGET_ROOT':'/',
   'UNDERCLOUD_IRONIC_PASSWORD':'7558a0701f802a84faa625e0bc563170a97aa6cb',
   'INSPECTION_COLLECTORS':'default,extra-hardware,logs',
   'ENABLE_MISTRAL':'True',
   'SHELL':'/bin/bash',
   'UNDERCLOUD_ENDPOINT_SWIFT_ADMIN':'http://192.168.24.1:8080',
   'UNDERCLOUD_ENDPOINT_GLANCE_ADMIN':'http://192.168.24.1:9292',
   'UNDERCLOUD_CEILOMETER_SNMPD_PASSWORD':'7e698c4fdb9a0eccc0a079c456c7a32a816120df',
   'UNDERCLOUD_ENDPOINT_NOVA_ADMIN':'http://192.168.24.1:8774/v2.1',
   'NODE_DIST':'centos7',
   'HISTSIZE':'1000',
   'UNDERCLOUD_DEBUG':'True',
   'UNDERCLOUD_DB_PASSWORD':'f2f25d1df2f05b810d7819bcfea45803185df833',
   'INSPECTION_RUNBENCH':'False',
   'SERVICE_PRINCIPAL':'',
   'UNDERCLOUD_SWIFT_HASH_SUFFIX':'b3c4eef1928b9b84427c06f2a1acc5d7f1b2f198',
   'UNDERCLOUD_ENDPOINT_IRONIC_INTERNAL':'http://192.168.24.1:6385',
   'XDG_RUNTIME_DIR':'/run/user/1000',
   'TRIPLEO_INSTALL_USER':'stack',
   'UNDERCLOUD_ENDPOINT_KEYSTONE_PUBLIC':'https://192.168.24.2:13000',
   'INSPECTION_KERNEL_ARGS':'ipa-debug=1 ipa-inspection-dhcp-all-interfaces=1 ipa-collect-lldp=1',
   'STORE_EVENTS':'False',
   'XDG_SESSION_ID':'198',
   'UNDERCLOUD_ADMIN_VIP':'192.168.24.3',
   'UNDERCLOUD_ENDPOINT_CEILOMETER_ADMIN':'http://192.168.24.1:8777',
   'JSONFILE':'/usr/share/instack-undercloud/json-files/centos-7-undercloud-packages.json',
   'HOSTNAME':'undercloud.localdomain',
   'SELINUX_LEVEL_REQUESTED':'',
   'UNDERCLOUD_ENDPOINT_AODH_PUBLIC':'https://192.168.24.2:13042',
   'UNDERCLOUD_ENDPOINT_AODH_INTERNAL':'http://192.168.24.1:8042',
   'UNDERCLOUD_ENDPOINT_MISTRAL_PUBLIC':'https://192.168.24.2:13989/v2',
   'DIB_INIT_SYSTEM':'systemd',
   'INSPECTION_INTERFACE':'br-ctlplane',
   'UNDERCLOUD_ENDPOINT_KEYSTONE_INTERNAL':'http://192.168.24.1:5000',
   'MAIL':'/var/spool/mail/stack',
   'TRIPLEO_UNDERCLOUD_PASSWORD_FILE':'/home/stack/undercloud-passwords.conf',
   'SCHEDULER_MAX_ATTEMPTS':'30',
   'UNDERCLOUD_ENDPOINT_HEAT_INTERNAL':'http://192.168.24.1:8004/v1/%(tenant_id)s',
   'UNDERCLOUD_ENDPOINT_AODH_ADMIN':'http://192.168.24.1:8042',
   'UNDERCLOUD_HEAT_PASSWORD':'561b715ce0f8ffe0c1482eeaed3df09cd2590a91',
   'LOCAL_INTERFACE':'eth1',
   'LESSOPEN':'||/usr/bin/lesspipe.sh %s',
   'MASQUERADE_NETWORK':'192.168.24.0/24',
   'USER':'root',
   'UNDERCLOUD_ENDPOINT_HEAT_PUBLIC':'https://192.168.24.2:13004/v1/%(tenant_id)s',
   'UNDERCLOUD_HEAT_STACK_DOMAIN_ADMIN_PASSWORD':'38e14254ddf98458ba09dcf600d936d3631a28b7',
   'UNDERCLOUD_ENDPOINT_NEUTRON_ADMIN':'http://192.168.24.1:9696',
   'SHLVL':'3',
   'UNDERCLOUD_ENDPOINT_ZAQAR_ADMIN':'http://192.168.24.1:8888',
   'UNDERCLOUD_ENDPOINT_CEILOMETER_PUBLIC':'https://192.168.24.2:13777',
   'UNDERCLOUD_ENDPOINT_KEYSTONE_ADMIN':'http://192.168.24.1:35357',
   'SUDO_USER':'stack',
   'UNDERCLOUD_ENDPOINT_IRONIC_INSPECTOR_INTERNAL':'http://192.168.24.1:5050',
   'ELEMENTS_PATH':'/usr/share/tripleo-puppet-elements:/usr/share/instack-undercloud:/usr/share/tripleo-image-elements:/usr/share/diskimage-builder/elements',
   'ENABLE_VALIDATIONS':'True',
   'UNDERCLOUD_ENDPOINT_ZAQAR_WEBSOCKET_INTERNAL':'ws://192.168.24.1:9000',
   'UNDERCLOUD_ADMIN_PASSWORD':'78ad5a42318e7d41e5a2e0f1f9600375ba985b16',
   'LOCAL_MTU':'1500',
   'TMP_MOUNT_PATH':'/tmp/instack.gbtlM6/mnt',
   'UNDERCLOUD_ENDPOINT_MISTRAL_INTERNAL':'http://192.168.24.1:8989/v2',
   'SSH_CONNECTION':'192.168.23.1 37621 192.168.23.47 22',
   'IMAGE_PATH':'.',
   'GUESTFISH_OUTPUT':'\\e[0m',
   'UNDERCLOUD_ENDPOINT_SWIFT_INTERNAL':'http://192.168.24.1:8080/v1/AUTH_%(tenant_id)s',
   'GENERATE_SERVICE_CERTIFICATE':'True',
   'UNDERCLOUD_CEILOMETER_METERING_SECRET':'989d08bc16f00298e7b0cf9fd396d5477a46c959',
   'DIB_IMAGE_CACHE':'/root/.cache/image-create',
   'DIB_DEFAULT_INSTALLTYPE':'package',
   'IPXE_ENABLED':'True',
   'UNDERCLOUD_ENDPOINT_IRONIC_INSPECTOR_PUBLIC':'https://192.168.24.2:13050',
   'SELINUX_USE_CURRENT_RANGE':'',
   'CLEAN_NODES':'False',
   'UNDERCLOUD_ENDPOINT_NOVA_PUBLIC':'https://192.168.24.2:13774/v2.1',
   'HOME':'/root',
   'UNDERCLOUD_ENDPOINT_ZAQAR_INTERNAL':'http://192.168.24.1:8888',
   'UNDERCLOUD_ENDPOINT_IRONIC_INSPECTOR_ADMIN':'http://192.168.24.1:5050',
   'GUESTFISH_INIT':'\\e[1;34m',
   'LANG':'en_US.utf8',
   'UNDERCLOUD_SERVICE_CERTIFICATE':'/etc/pki/tls/certs/undercloud-192.168.24.2.pem',
   'ENABLE_TELEMETRY':'True',
   'DHCP_START':'192.168.24.5',
   'IMAGE_NAME':'instack',
   'DIB_OFFLINE':'',
   'ENABLE_UI':'True',
   '_':'/usr/bin/python',
   'NET_CONFIG_OVERRIDE':'',
   'PUBLIC_INTERFACE_IP':'192.168.24.1/24',
   'UNDERCLOUD_ENDPOINT_IRONIC_ADMIN':'http://192.168.24.1:6385',
   'USERNAME':'root',
   'UNDERCLOUD_ENDPOINT_HEAT_ADMIN':'http://192.168.24.1:8004/v1/%(tenant_id)s',
   'LOCAL_IP':'192.168.24.1',
   'SELINUX_ROLE_REQUESTED':'',
   'UNDERCLOUD_ENDPOINT_GLANCE_INTERNAL':'http://192.168.24.1:9292',
   'SUDO_GID':'1000',
   'UNDERCLOUD_ENDPOINT_IRONIC_PUBLIC':'https://192.168.24.2:13385',
   '_LIB':'/usr/share/diskimage-builder/lib',
   'SSH_TTY':'/dev/pts/1',
   'DHCP_END':'192.168.24.30',
   'UNDERCLOUD_ENDPOINT_NEUTRON_INTERNAL':'http://192.168.24.1:9696',
   'UNDERCLOUD_ENDPOINT_ZAQAR_WEBSOCKET_ADMIN':'ws://192.168.24.1:9000',
   'UNDERCLOUD_RABBIT_COOKIE':'274d7049a7cdd3be29759776b6054907be3cb410',
   'NETWORK_GATEWAY':'192.168.24.1',
   'UNDERCLOUD_ENDPOINT_NOVA_INTERNAL':'http://192.168.24.1:8774/v2.1',
   'INSPECTION_ENABLE_UEFI':'True',
   'TRIPLEO_UNDERCLOUD_CONF_FILE':'/home/stack/undercloud.conf',
   'CERTIFICATE_GENERATION_CA':'local',
   'HIERADATA_OVERRIDE':'quickstart-hieradata-overrides.yaml',
   'UNDERCLOUD_NEUTRON_PASSWORD':'4bde341a0b0183b0b7d40d71a9f4f7d78b62f163',
   'ENABLE_ZAQAR':'True',
   'SSH_CLIENT':'192.168.23.1 37621 22',
   'UNDERCLOUD_ENDPOINT_MISTRAL_ADMIN':'http://192.168.24.1:8989/v2',
   'LOGNAME':'root',
   'UNDERCLOUD_CEILOMETER_PASSWORD':'457c7fb371fe0228e41484de9b77d7515146c3c1',
   'PATH':'/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin:/tmp/tmp0o4uKR/bin',
   'UNDERCLOUD_ENDPOINT_SWIFT_PUBLIC':'https://192.168.24.2:13808/v1/AUTH_%(tenant_id)s',
   'INSPECTION_IPRANGE':'192.168.24.100,192.168.24.120',
   'GUESTFISH_RESTORE':'\\e[0m',
   'MEMBER_ROLE_EXISTS':'False',
   'UNDERCLOUD_ENDPOINT_ZAQAR_PUBLIC':'https://192.168.24.2:13888',
   'TERM':'screen',
   'ENABLE_TEMPEST':'True',
   'ARCH':'amd64',
   'IMAGE_ELEMENT':u'element-manifest enable-packages-install hiera install-types manifests os-apply-config os-refresh-config puppet-modules puppet-stack-config source-repositories undercloud-install',
   'UNDERCLOUD_ZAQAR_PASSWORD':'a563879e04e18c7c4cf3f2b4e66637b1d84b9e10',
   'DIB_ARGS':"['/bin/instack', '-p', '/usr/share/tripleo-puppet-elements:/usr/share/instack-undercloud:/usr/share/tripleo-image-elements:/usr/share/diskimage-builder/elements', '-j', '/usr/share/instack-undercloud/json-files/centos-7-undercloud-packages.json']",
   'UNDERCLOUD_AODH_PASSWORD':'9b4aacbb5ea53d59946e22152e4296e9e157fb8b',
   'UNDERCLOUD_ADMIN_TOKEN':'f4d890602dc406c76cf3c156f3a8c374a2028b8b',
   'UNDERCLOUD_ENDPOINT_ZAQAR_WEBSOCKET_PUBLIC':'wss://192.168.24.2:9000',
   'UNDERCLOUD_CEILOMETER_SNMPD_USER':'ro_snmp_user',
   'GUESTFISH_PS1':'\\[\\e[1;32m\\]><fs>\\[\\e[0;31m\\] ',
   'UNDERCLOUD_SWIFT_PASSWORD':'6646c8abbc69900be43ad9aeb2fb9bcde084b83c',
   'UNDERCLOUD_ENDPOINT_GLANCE_PUBLIC':'https://192.168.24.2:13292',
   'SUDO_UID':'1000',
   'UNDERCLOUD_ENDPOINT_CEILOMETER_INTERNAL':'http://192.168.24.1:8777',
   'TMP_HOOKS_PATH':'/tmp/tmp0o4uKR',
   'UNDERCLOUD_ENDPOINT_NEUTRON_PUBLIC':'https://192.168.24.2:13696',
   'UNDERCLOUD_HAPROXY_STATS_PASSWORD':'cfe22a516eb260a515368b42ecdafd0c31a4eab0',
   'SUDO_COMMAND':'/bin/instack -p /usr/share/tripleo-puppet-elements:/usr/share/instack-undercloud:/usr/share/tripleo-image-elements:/usr/share/diskimage-builder/elements -j /usr/share/instack-undercloud/json-files/centos-7-undercloud-packages.json',
   'UNDERCLOUD_GLANCE_PASSWORD':'432b0417d7e43907e9df98fae3b509306932275c',
   'UNDERCLOUD_RABBIT_PASSWORD':'3f63076c9b2ec748d7021311acc0a4fcfcbf1b30',
   'UNDERCLOUD_MISTRAL_PASSWORD':'a4e093153909d0d57c627e3b3b704c286fee2fa0',
   'UNDERCLOUD_NOVA_PASSWORD':'54fca7b5d7905511604f69de11d67d4144bb669f',
   'NETWORK_CIDR':'192.168.24.0/24',
   'HISTCONTROL':'ignoredups',
   'PWD':'/home/stack',
   'UNDERCLOUD_PUBLIC_VIP':'192.168.24.2',
   'INSPECTION_EXTRAS':'True',
   'UNDERCLOUD_HEAT_ENCRYPTION_KEY':'9f0926190c3bd6c45203bb645af27039',
   'UNDERCLOUD_HOSTNAME':'None',
   'UNDERCLOUD_RABBIT_USERNAME':'44422a8e431da7e68defea8712790cf69bca46b1'
}
```

### 附录2

os-apply-config使用的metadata配置文件示例：

```
{
 "hiera": {
  "hierarchy": [
   "quickstart-hieradata-overrides",
   "\"%{::operatingsystem}\"",
   "\"%{::osfamily}\"",
   "puppet-stack-config"
  ]},
  "local-ip": "192.168.24.1",
  "masquerade_networks": ["192.168.24.0/24"],
  "service_certificate": "/etc/pki/tls/certs/undercloud-192.168.24.2.pem",
  "public_vip": "192.168.24.2",
  "neutron": {
    "dhcp_start": "192.168.24.5",
    "dhcp_end": "192.168.24.30",
    "network_cidr": "192.168.24.0/24",
    "network_gateway": "192.168.24.1"
  },
  "inspection": {
    "interface": "",
    "iprange": "",
    "runbench": ""
  },
  "os_net_config": {
  "network_config": [
   {
    "type": "ovs_bridge",
    "name": "br-ctlplane",
    "ovs_extra": [
     "br-set-external-id br-ctlplane bridge-id br-ctlplane"
    ],
    "members": [
     {
      "type": "interface",
      "name": "eth1",
      "primary": "true",
      "mtu": 1500
     }
    ],
    "addresses": [
      {
        "ip_netmask": "192.168.24.1/24"
      }
    ],
    "mtu": 1500
  }
  ]
  },
  "keystone": {
    "host": "192.168.24.1"
  },
  "ironic": {
    "service-password": "7558a0701f802a84faa625e0bc563170a97aa6cb"
  },
  "bootstrap_host": {
    "bootstrap_nodeid": "undercloud",
    "nodeid": "undercloud"
  }
}
```

### 附录3

instack-undercloud中使用到的elements以及其中hook和脚本示例：

```
        * install-types(diskimage-builder)
            * extra-data.d
                * 99-enable-install-types
            * The base element enables the chosen install type by symlinking the correct hook scripts under install.d directly
        * element-manifest(diskimage-builder)
            * extra-data.d
                * 75-inject-element-manifest
        * manifests(diskimage-builder)
            * extra-data.d
                * 20-manifest-dir
            * environment.d
                * 14-manifests
            * cleanup.d
                * 01-copy-manifests-dir
        * source-repositories(diskimage-builder)
            * extra-data.d
                * 98-source-repositories

        * puppet-modules(tripleo-puppet-elements)
            * install.d/
                * puppet-modules-package-install/
                    * 75-puppet-modules-package
                * puppet-modules-source-install/
                    * 75-puppet-modules-source
                * 根据install_type会将对应目录下的脚本link到install.d目录下
            * environment.d/
                * 01-puppet-module-pins.sh
                * 02-puppet-modules-install-types.sh
            * source-repository-puppet-modules
        * hiera(tripleo-puppet-elements)
            * 10-hiera-disable
            * 40-hiera-datafiles
            * install.d/
                * 10-hiera-yaml-symlink
                * 11-hiera-orc-install
            * os-apply-config/
                * etc/puppet/hiera.yaml

        * enable-packages-install(tripleo-image-elements)
            * environment.d/
                * 01-export-install-types.bash
        * os-apply-config(tripleo-image-elements)
            * environment.d/
                * 10-os-apply-config-venv-dir.bash
            * install.d/
                * 11-create-template-root
                * 99-install-config-templates
                * os-apply-config-source-install/
                    * 10-os-apply-config
            * os-refresh-config/
                * configure.d/
                    * 20-os-apply-config
        * os-refresh-config(tripleo-image-elements)
            * install.d/
                * 99-os-refresh-config-install-scripts
                * os-refresh-config-source-install/
                    * 10-os-refresh-config
            * os-refresh-config/
                * post-configure.d/
                    * 99-refresh-completed

        * undercloud-install(instack-undercloud)
            * os-apply-config/
                * etc/
                * root/
                * var/
            * os-refresh-config/
                * configure.d/
                    * 30-reload-keepalived
                * post-configure.d/
                    * 80-seedstack-masquerade
                    * 98-undercloud-setup
        * puppet-stack-config(instack-undercloud)
            * extra-data.d/
                * 10-install-git
            * install.d/
                * 02-puppet-stack-config
                * 10-puppet-stack-config-puppet-module
            * os-apply-config/
                * etc/
            * os-refresh-config/
                * configure.d/
                    * 50-puppet-stack-config
                * post-configure.d/
                    * 10-iptables
```

### 附录4

instack-undercloud中使用到的elements以及其中hook和脚本，在instack中被合并之后的示例：

```
▾ cleanup.d/
    01-copy-manifests-dir*
▾ environment.d/
    01-export-install-types.bash
    02-puppet-modules-install-types.sh
    10-os-apply-config-venv-dir.bash
    14-manifests
▾ extra-data.d/
    10-install-git*
    20-manifest-dir*
    75-inject-element-manifest*
    98-source-repositories*
    99-enable-install-types*
▾ install.d/
  ▾ os-apply-config-source-install/
      10-os-apply-config*
  ▾ os-refresh-config-source-install/
      10-os-refresh-config*
  ▾ puppet-modules-package-install/
      75-puppet-modules-package*
  ▾ puppet-modules-source-install/
      75-puppet-modules-source*
    02-puppet-stack-config*
    10-hiera-yaml-symlink*
    10-puppet-stack-config-puppet-module*
    11-create-template-root*
    99-install-config-templates*
    99-os-refresh-config-install-scripts*
    package-installs-hiera
```



