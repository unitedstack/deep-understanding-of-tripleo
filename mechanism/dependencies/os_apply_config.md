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



