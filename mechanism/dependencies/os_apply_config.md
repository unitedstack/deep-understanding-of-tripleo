# os-apply-config

os-apply-config是一个配置工具，它主要是从多个json格式的配置文件中，读取配置项，然后渲染预先定制好的模板，生成最终的配置文件，然后放置到相应的位置中去。

这个json格式的配置文件，在os-apply-config中叫做metadata，就是定义了一些key-value值，用来渲染模板，os-apply-config提供了好几种方式来确定这些json文件，按照优先级如下：

1. 通过命令行中的`--metadata`选项来指定多个json文件：`--metadata [METADATA_FILE [METADATA_FILE ...]]`
2. 如果`--metadata`没有指定，那么通过环境变量`OS_CONFIG_FILES`来指定，每个json文件以":"分隔
3. 如果`OS_CONFIG_FILES`没有指定，那么通过命令行中的`--os-config-files`来指定，这个选项默认的值是`OS_CONFIG_FILES_PATH`环境变量指定的，这个环境变量默认的值为：`/var/lib/os-collect-config/os_config_files.json`
4. 如果前面的选项都没有找到json文件，那么用`/var/run/os-collect-config/os_config_files.json`这个位置的json文件，这个是以前旧的配置项，要被dreprecated了
5. 不管前面的选项有没有找到json文件，都会使用`--fallback-metadata`指定的json文件，这个选项默认指定了3个json文件：
   * `/var/cache/heat-cfntools/last_metadata`
   * `/var/lib/heat-cfntools/cfn-init-data`
   * `/var/lib/cloud/data/cfn-init-data`

这些json配置文件中的配置项最终都会被合并到一个dict对象中，用来渲染模板。

os-apply-config的模板由配置项`--templates`指定，这个配置项默认值由以下方式确定：

1. 由`OS_CONFIG_APPLIER_TEMPLATES`环境变量指定，默认为None
2. `/opt/stack/os-apply-config/templates`
3. `/opt/stack/os-config-applier/templates`
4. `/usr/libexec/os-apply-config/templates`，该值为默认值

生成的最终的配置文件，放置的位置由`--output`配置项指定，默认为根目录"/"。

举个例子，有如下的文件：

```
[root@localhost os-apply-config]# tree
.
├── config
│   └── os_config_files.json
├── output
└── templates
    └── etc
        └── nova.conf
```

templates/etc/nova.conf内容如下：

```
[database]
{{#nova.db}}
connection={{nova.db}}
{{/nova.db}}
```

config/os\__config\_files.json内容如下：_

```
{
  "nova":{
    "db": "mysql://nova:unset@localhost/nova"
  }
}
```

执行下面的命令：

```
# os-apply-config -t templates/ -m config/os_config_files.json -o output/
[2017/03/05 11:59:07 PM] [INFO] writing output/etc/nova.conf
[2017/03/05 11:59:07 PM] [INFO] success
```

在output目录下就会生成相应的配置文件：

```
[root@localhost os-apply-config]# tree
.
├── config
│   └── os_config_files.json
├── output
│   └── etc
│       └── nova.conf
└── templates
    └── etc
        └── nova.conf
```

output/etc/nova.conf内容如下：

```
[database]
connection=mysql://nova:unset@localhost/nova
```

可见，使用os-apply-config可以方便的生成一组配置文件，默认的output是根目录，就会将etc等配置文件全部配置到相应的位置，这在部署undercloud和overcloud时都会被用到。

os-apply-config还有一个作用就是指定key值，然后输出对应的value值，如下：

```
# os-apply-config -t templates/ -m config/os_config_files.json --key nova --type raw
{"db": "mysql://nova:unset@localhost/nova"}
# os-apply-config -t templates/ -m config/os_config_files.json --key nova.db --type raw
mysql://nova:unset@localhost/nova
```



