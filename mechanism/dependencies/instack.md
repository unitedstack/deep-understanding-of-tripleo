# instack

instack是一个用来在当前系统中执行diskimage-builder格式的elements的工具。elements最初是在diskimage-builder中出现的，diskimage-builder是一个制作镜像的工具，因为定制镜像，需要安装各种各样的东西，elements就是它抽象出来的一个个功能的集合，这有点类似于ansible中role的概念，需要在镜像中加入什么功能，那么指定相应的element就可以了，diskimage-builder按顺序执行elements中的脚本，从而定制镜像，因此elements是可以分发的，每个人都可以写自己的elements，然后供别人使用。

这种抽象机制非常不错，因此也被应用到tripleo中，但是并没有使用diskimage-builder来执行elements，而是使用instack来执行，instack底层又是使用的dib-run-parts工具来执行的，并且加上了自己的一些逻辑。instack是一个相对底层的工具，在tripleo中，被封装在instack-undercloud中，在部署undercloud时被用到。

instack使用方法如下：

```
[stack@undercloud ~]$ instack -h
usage: instack [-h] [-e [ELEMENT [ELEMENT ...]]]
               [-p ELEMENT_PATH [ELEMENT_PATH ...]] [-k [HOOK [HOOK ...]]]
               [-b [BLACKLIST [BLACKLIST ...]]]
               [-x [EXCLUDE_ELEMENT [EXCLUDE_ELEMENT ...]]] [-j JSON_FILE]
               [-d] [-i] [--dry-run] [--no-cleanup] [-l LOGFILE]

Execute diskimage-builder elements on the current system.

optional arguments:
  -h, --help            show this help message and exit
  -e [ELEMENT [ELEMENT ...]], --element [ELEMENT [ELEMENT ...]]
                        element(s) to execute
  -p ELEMENT_PATH [ELEMENT_PATH ...], --element-path ELEMENT_PATH [ELEMENT_PATH ...]
                        element path(s) to search for elements (ELEMENTS_PATH
                        environment variable will take precedence if defined)
  -k [HOOK [HOOK ...]], --hook [HOOK [HOOK ...]]
                        hook(s) to execute for each element
  -b [BLACKLIST [BLACKLIST ...]], --blacklist [BLACKLIST [BLACKLIST ...]]
                        script names, that if found, will be blacklisted and
                        not run
  -x [EXCLUDE_ELEMENT [EXCLUDE_ELEMENT ...]], --exclude-element [EXCLUDE_ELEMENT [EXCLUDE_ELEMENT ...]]
                        element names that will be excluded from running even
                        if they are listed as dependencies
  -j JSON_FILE, --json-file JSON_FILE
                        read runtime configuration from json file
  -d, --debug           Debugging output
  -i, --interactive     If set, prompt to continue running after a failed
                        script.
  --dry-run             Dry run only, don't actually modify system, prints out
                        what would have been run.
  --no-cleanup          Do not cleanup tmp directories
  -l LOGFILE, --logfile LOGFILE
                        Logfile to log all actions
```

需要说明的是hook，hook用来指定要执行elements中的哪些脚本，因为在每一个element中，都按照格式定义了一些目录，如extra-data, pre-install, install, post-install，在这些目录中放置了一些脚本，hook就是用来执行要执行哪些目录中的脚本，没有被指定的目录，将会被跳过。

可以在命令行中直接指定elements，和相应的hook来执行，如下：

```
sudo -E instack \
    -p /usr/share/tripleo-image-elements /usr/share/diskimage-builder/elements \
    -e fedora base keystone mariadb \
    -k extra-data pre-install install post-install \
    -b 15-remove-grub 10-cloud-init 05-fstab-rootfs-label
```

也可以将这些选项全都配置在一个json格式的配置文件中，直接指定这些配置文件就可以了，如下：

```
sudo -E instack \
    -p /usr/share/tripleo-image-elements /usr/share/diskimage-builder/elements \
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



