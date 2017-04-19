# 自定义Ceph存储部署

---




# 4. 配置 ceph osd

在stack用户目录下建立`templates`目录

```
mkdir ~/templates
```

复制ceph 配置文件到templates目录中

```
cp /usr/share/openstack-tripleo-heat-templates/environments/storage-environment.yaml ~/templates/
```

在storage-environment.yaml中添加一段ExtraConfig 的section，格式大致如下：

```
parameter_defaults: #已存在的section
ExtraConfig: #这是我们要添加的section
ceph::profile::params::osds:
```

分别存储journal 和 osd

```
ceph::profile::params::osds:
'/dev/sdc':
journal: '/dev/sdb'
'/dev/sdd':
journal: '/dev/sdb'
```

将journal 和osd 数据放在一个硬盘内

```
ceph::profile::params::osds:
'/dev/sdb': {}
'/dev/sdc': {}
'/dev/sdd': {}
```

---


**wipe-disks.yaml**
```yaml
heat_template_version: 2014-10-16

description: >
  Wipe and convert all disks to GPT (except the disk containing the root file system)

resources:
  userdata:
    type: OS::Heat::MultipartMime
    properties:
      parts:
      - config: {get_resource: wipe_disk}

  wipe_disk:
    type: OS::Heat::SoftwareConfig
    properties:
      config: {get_file: wipe-disk.sh}

outputs:
  OS::stack_id:
    value: {get_resource: userdata}
```


**wipe-disk.sh**
```yaml
#!/bin/bash
if [[ `hostname` = *"ceph"* ]]
then
  echo "Number of disks detected: $(lsblk -no NAME,TYPE,MOUNTPOINT | grep "disk" | awk '{print $1}' | wc -l)"
  for DEVICE in `lsblk -no NAME,TYPE,MOUNTPOINT | grep "disk" | awk '{print $1}'`
  do
    ROOTFOUND=0
    echo "Checking /dev/$DEVICE..."
    echo "Number of partitions on /dev/$DEVICE: $(expr $(lsblk -n /dev/$DEVICE | awk '{print $7}' | wc -l) - 1)"
    for MOUNTS in `lsblk -n /dev/$DEVICE | awk '{print $7}'`
    do
      if [ "$MOUNTS" = "/" ]
      then
        ROOTFOUND=1
      fi
    done
    if [ $ROOTFOUND = 0 ]
    then
      echo "Root not found in /dev/${DEVICE}"
      echo "Wiping disk /dev/${DEVICE}"
      sgdisk -Z /dev/${DEVICE}
      sgdisk -g /dev/${DEVICE}
    else
      echo "Root found in /dev/${DEVICE}"
    fi
  done
fi
```



