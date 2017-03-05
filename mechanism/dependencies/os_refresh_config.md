# os-refresh-config

os-refresh-config是一个用来有序执行一组脚本的工具，它将脚本分组定义成了4个阶段：

* pre-configure
* configure
* post-configure
* migration

每个阶段对应一个目录：

* pre-configure.d
* configure.d
* post-configure.d
* migration.d

这些目录的位置默认是在`/usr/libexec/os-refresh-config`目录下，也可以由`OS_REFRESH_CONFIG_BASE_DIR`环境变量指定位置。在每个目录中放置了一些脚本，使用dib-run-parts来依次执行这些目录中的脚本，dib-run-parts是[dib-utils](https://github.com/openstack/dib-utils\)中的工具，这个工具最初是在diskimage-builder中被使用的。由于脚本在执行过程中，会使用到各种参数，都是通过环境变量指定的，dib-run-parts在执行这些脚本之前，会先导出环境变量，这些环境变量需要被定义在environment.d目录下，dib-run-parts会先source这些环境变量。

这个工具有点类似于ansible的编排功能，通过执行这4个阶段的脚本，来完成相应的配置，在部署undercloud时，就是用它来编排整个过程的：

    [stack@undercloud os-refresh-config]$ tree
    .
    |-- configure.d
    |   |-- 20-os-apply-config
    |   |-- 30-reload-keepalived
    |   |-- 40-hiera-datafiles
    |   |-- 40-truncate-nova-config
    |   `-- 50-puppet-stack-config
    `-- post-configure.d
        |-- 10-iptables
        |-- 80-seedstack-masquerade
        |-- 98-undercloud-setup
        `-- 99-refresh-completed



