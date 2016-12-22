## 部署overcloud

## 1. 准备overcloud 镜像

Overcloud 镜像可以自己制作也可以下载现成的。这里的演示使用下载的镜像。

[Overcloud 镜像下载地址](http://buildlogs.centos.org/centos/7/cloud/x86_64/tripleo_images/)

将这些文件放到stack用户的根目录底下

```
ironic-python-agent.initramfs
ironic-python-agent.kernel
overcloud-full.initrd
overcloud-full.qcow2
overcloud-full.vmlinuz
```

## 2. 上传镜像

```
$ . stackrc
$ openstack overcloud image upload
```

## 3. 收集物理机信息

示例中使用的是虚拟环境，虚拟环境与物理环境使用的ironic驱动不同。

将overcloud vm的信息写入instackenv.json。参照以下格式：

虚拟环境instackenv.json

```
{
  "arch": "x86_64",
  "host-ip": "192.168.122.1",
  "power_manager": "nova.virt.baremetal.virtual_power_driver.VirtualPowerManager",
  "seed-ip": "",
  "ssh-key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEogIBAAKCAQEAtBI6DK5CFAxPJJEoLgnI5v/kn0ncV26o9wP9+di7czMhD6WX\ndfLtn2WNALVRopIVXDwb78JqPQEpXgWEZGIv4JIteYdh/GrdQhnmqEL/6FpMjMfZ\nnGPclfzg6dM2khRFexaf50G+bLb5kgIpFLOG0DJBI/r36lMVRz5I2LwKixWNeEIX\nz445SwPj4lUlbfjoodAPEX8HLQanCvaavTNDVvq5q8Qb3fQ2gXScA1crRUN9uMv0\n+JZTFbwkqQHepMb9DJKxHF6BH9tE5+Ttmc0Ra1eel0rteXK2A6CYX+vjiqtQkuNt\ntYtyKNvmKmhv4udd2YaqK/nGoKZEgULpcgfeUQIDAQABAoIBAGJzQKekMl5xqGeO\nsVASa3PYXi+0mzJ2PwzmctpB46KFRsMePuPu0HoAdIn5mEtw4RrPhlqciacW1n4g\nOBUGFbULVq+GFE2EQ7obHR/Lmcx4ajfiIBjABF9ApdtRbhmJ2b8FTKGMMUeQ9nwc\nkEdQLBnyD+lTEm5bxFtyMzPEA2OsliuV+R/7W892+JBAaNsvTUrW1+rm/9oxCoQv\nu/dxIAgiNnUULZvcCEBoZaiHQsdDyM1zUAkpoWBmp4kLICXC8xYHZUyFHp3ScHZV\nKKqlxbS/+61qUt7egCosb3GTP+KV5etd0MO2zN8TpNgkRPzZ7y9B/LmkHTlaazD9\nqSa1IgECgYEA61fxkRvtVB80sn53rw1XUT1RyWTYcHobDWCEmXPmtQERyvSndO4i\nC2540s1QPkZbZ3Fk2pvgm2+8NqTf0EFJDSan84HS0j7+x4tG4F6ijPTXL0DsClY2\nbrJIDTLAhWLulHKVd3HRNK4OaUSLazFIknIA+C8uZvsJkNLSWG4xBbECgYEAw+BX\nH20SVRJpY3rK3wrYIVcpDXDQm61nyJVGIO3BBH/GmviCb+E9xecahZSO+qq1wnad\nIJpp1lsdzU9iBy402TFL9nRMHb4JsR6Id7sNXV7rX3zaC3JGKqJiJEPKo+U1Mn1f\nZrBi73t/ylVKX5n8gOfeFslwmDquQJ0mlSRYaqECgYAL+AQEEjyGq7OdZEsn7vDC\n4/B14pgTWFJp4r+7oiZYjD5gaQLfMoEuvaaNaf2rvR5G64BqkcThgtQ6nzX2vGs/\nrPibrL2RDb0dXtry7D0uGAGdmJqoh+vqw0xgx3T9E6P4jr9FPNeb60I2XlMM14vO\nTtf3x0Z/3EKHSAGEl84McQKBgHDc2RZwgHmoTDVX0YFG/FXppOvrryekeQJokKn0\nlJ0FCujMfEv+2tsnWG7TtLbWmjhcpBjfIFC0260rKm68vxLOhtiRFjKlB2yZDUT/\n8Kl2QeUZSYIC7E8wlaATt7VMIqTe/JNs2vTmkjGBh4MidQ3JjHxQwaHVXgY5Brw0\n3wVBAoGAJsbIHlcKsX8q4hU/Sp2VwohZUmwR3eooTfVMmQjXdI0h3g8H/I0XzA5W\nqHcMJ/5ba4w6sztYRnGn8jIlyozhI9lGv/ajYPcbS3nuE7nEl98vbve8hcLP+VCJ\nkbMz+s1SELnexCmGQHdHxUp3nuERwd2xzQPBEYE6N+VlsATKrgg=\n-----END RSA PRIVATE KEY-----\n",
  "ssh-user": "root",
  "nodes": [
    {
      "mac": [
        "00:f0:09:b0:54:80"
      ],
      "cpu": "1",
      "memory": "6144",
      "disk": "40",
      "arch": "x86_64",
      "pm_user": "root",
      "pm_addr": "192.168.122.1",
      "pm_password": "-----BEGIN RSA PRIVATE KEY-----\nMIIEogIBAAKCAQEAtBI6DK5CFAxPJJEoLgnI5v/kn0ncV26o9wP9+di7czMhD6WX\ndfLtn2WNALVRopIVXDwb78JqPQEpXgWEZGIv4JIteYdh/GrdQhnmqEL/6FpMjMfZ\nnGPclfzg6dM2khRFexaf50G+bLb5kgIpFLOG0DJBI/r36lMVRz5I2LwKixWNeEIX\nz445SwPj4lUlbfjoodAPEX8HLQanCvaavTNDVvq5q8Qb3fQ2gXScA1crRUN9uMv0\n+JZTFbwkqQHepMb9DJKxHF6BH9tE5+Ttmc0Ra1eel0rteXK2A6CYX+vjiqtQkuNt\ntYtyKNvmKmhv4udd2YaqK/nGoKZEgULpcgfeUQIDAQABAoIBAGJzQKekMl5xqGeO\nsVASa3PYXi+0mzJ2PwzmctpB46KFRsMePuPu0HoAdIn5mEtw4RrPhlqciacW1n4g\nOBUGFbULVq+GFE2EQ7obHR/Lmcx4ajfiIBjABF9ApdtRbhmJ2b8FTKGMMUeQ9nwc\nkEdQLBnyD+lTEm5bxFtyMzPEA2OsliuV+R/7W892+JBAaNsvTUrW1+rm/9oxCoQv\nu/dxIAgiNnUULZvcCEBoZaiHQsdDyM1zUAkpoWBmp4kLICXC8xYHZUyFHp3ScHZV\nKKqlxbS/+61qUt7egCosb3GTP+KV5etd0MO2zN8TpNgkRPzZ7y9B/LmkHTlaazD9\nqSa1IgECgYEA61fxkRvtVB80sn53rw1XUT1RyWTYcHobDWCEmXPmtQERyvSndO4i\nC2540s1QPkZbZ3Fk2pvgm2+8NqTf0EFJDSan84HS0j7+x4tG4F6ijPTXL0DsClY2\nbrJIDTLAhWLulHKVd3HRNK4OaUSLazFIknIA+C8uZvsJkNLSWG4xBbECgYEAw+BX\nH20SVRJpY3rK3wrYIVcpDXDQm61nyJVGIO3BBH/GmviCb+E9xecahZSO+qq1wnad\nIJpp1lsdzU9iBy402TFL9nRMHb4JsR6Id7sNXV7rX3zaC3JGKqJiJEPKo+U1Mn1f\nZrBi73t/ylVKX5n8gOfeFslwmDquQJ0mlSRYaqECgYAL+AQEEjyGq7OdZEsn7vDC\n4/B14pgTWFJp4r+7oiZYjD5gaQLfMoEuvaaNaf2rvR5G64BqkcThgtQ6nzX2vGs/\nrPibrL2RDb0dXtry7D0uGAGdmJqoh+vqw0xgx3T9E6P4jr9FPNeb60I2XlMM14vO\nTtf3x0Z/3EKHSAGEl84McQKBgHDc2RZwgHmoTDVX0YFG/FXppOvrryekeQJokKn0\nlJ0FCujMfEv+2tsnWG7TtLbWmjhcpBjfIFC0260rKm68vxLOhtiRFjKlB2yZDUT/\n8Kl2QeUZSYIC7E8wlaATt7VMIqTe/JNs2vTmkjGBh4MidQ3JjHxQwaHVXgY5Brw0\n3wVBAoGAJsbIHlcKsX8q4hU/Sp2VwohZUmwR3eooTfVMmQjXdI0h3g8H/I0XzA5W\nqHcMJ/5ba4w6sztYRnGn8jIlyozhI9lGv/ajYPcbS3nuE7nEl98vbve8hcLP+VCJ\nkbMz+s1SELnexCmGQHdHxUp3nuERwd2xzQPBEYE6N+VlsATKrgg=\n-----END RSA PRIVATE KEY-----\n",
      "pm_type": "pxe_ssh"
    },
    {
      "mac": [
        "00:ad:d2:7d:84:3a"
      ],
      "cpu": "1",
      "memory": "6144",
      "disk": "40",
      "arch": "x86_64",
      "pm_user": "root",
      "pm_addr": "192.168.122.1",
      "pm_password": "-----BEGIN RSA PRIVATE KEY-----\nMIIEogIBAAKCAQEAtBI6DK5CFAxPJJEoLgnI5v/kn0ncV26o9wP9+di7czMhD6WX\ndfLtn2WNALVRopIVXDwb78JqPQEpXgWEZGIv4JIteYdh/GrdQhnmqEL/6FpMjMfZ\nnGPclfzg6dM2khRFexaf50G+bLb5kgIpFLOG0DJBI/r36lMVRz5I2LwKixWNeEIX\nz445SwPj4lUlbfjoodAPEX8HLQanCvaavTNDVvq5q8Qb3fQ2gXScA1crRUN9uMv0\n+JZTFbwkqQHepMb9DJKxHF6BH9tE5+Ttmc0Ra1eel0rteXK2A6CYX+vjiqtQkuNt\ntYtyKNvmKmhv4udd2YaqK/nGoKZEgULpcgfeUQIDAQABAoIBAGJzQKekMl5xqGeO\nsVASa3PYXi+0mzJ2PwzmctpB46KFRsMePuPu0HoAdIn5mEtw4RrPhlqciacW1n4g\nOBUGFbULVq+GFE2EQ7obHR/Lmcx4ajfiIBjABF9ApdtRbhmJ2b8FTKGMMUeQ9nwc\nkEdQLBnyD+lTEm5bxFtyMzPEA2OsliuV+R/7W892+JBAaNsvTUrW1+rm/9oxCoQv\nu/dxIAgiNnUULZvcCEBoZaiHQsdDyM1zUAkpoWBmp4kLICXC8xYHZUyFHp3ScHZV\nKKqlxbS/+61qUt7egCosb3GTP+KV5etd0MO2zN8TpNgkRPzZ7y9B/LmkHTlaazD9\nqSa1IgECgYEA61fxkRvtVB80sn53rw1XUT1RyWTYcHobDWCEmXPmtQERyvSndO4i\nC2540s1QPkZbZ3Fk2pvgm2+8NqTf0EFJDSan84HS0j7+x4tG4F6ijPTXL0DsClY2\nbrJIDTLAhWLulHKVd3HRNK4OaUSLazFIknIA+C8uZvsJkNLSWG4xBbECgYEAw+BX\nH20SVRJpY3rK3wrYIVcpDXDQm61nyJVGIO3BBH/GmviCb+E9xecahZSO+qq1wnad\nIJpp1lsdzU9iBy402TFL9nRMHb4JsR6Id7sNXV7rX3zaC3JGKqJiJEPKo+U1Mn1f\nZrBi73t/ylVKX5n8gOfeFslwmDquQJ0mlSRYaqECgYAL+AQEEjyGq7OdZEsn7vDC\n4/B14pgTWFJp4r+7oiZYjD5gaQLfMoEuvaaNaf2rvR5G64BqkcThgtQ6nzX2vGs/\nrPibrL2RDb0dXtry7D0uGAGdmJqoh+vqw0xgx3T9E6P4jr9FPNeb60I2XlMM14vO\nTtf3x0Z/3EKHSAGEl84McQKBgHDc2RZwgHmoTDVX0YFG/FXppOvrryekeQJokKn0\nlJ0FCujMfEv+2tsnWG7TtLbWmjhcpBjfIFC0260rKm68vxLOhtiRFjKlB2yZDUT/\n8Kl2QeUZSYIC7E8wlaATt7VMIqTe/JNs2vTmkjGBh4MidQ3JjHxQwaHVXgY5Brw0\n3wVBAoGAJsbIHlcKsX8q4hU/Sp2VwohZUmwR3eooTfVMmQjXdI0h3g8H/I0XzA5W\nqHcMJ/5ba4w6sztYRnGn8jIlyozhI9lGv/ajYPcbS3nuE7nEl98vbve8hcLP+VCJ\nkbMz+s1SELnexCmGQHdHxUp3nuERwd2xzQPBEYE6N+VlsATKrgg=\n-----END RSA PRIVATE KEY-----\n",
      "pm_type": "pxe_ssh"
    }
  ]
}
```
导入json
```

```


## 3. 定义根磁盘

查看introspection得到的磁盘信息，确认sda是不是我们想要的根磁盘。

    list=(`ironic node-list|grep power|awk '{print $2}'`);for i in  ${list[*]} ;do openstack baremetal introspection data save $i | jq '.inventory.disks' ;done

### 如果sda是我们想要的根磁盘:

    list=(`ironic node-list|grep power|awk '{print $2}'`);for i in ${list[*]};do ironic node-update $i add properties/root_device='{"name": "/dev/sda"}';done

### 如果sda不是我们想要的根磁盘

那就需要使用serial或者wwn来定义根磁盘，手动对每一个node依次执行定义:

```
#wwn
ironic node-update $i add properties/root_device=properties/root_device='{"wwn": "xxx"}'

#serial
ironic node-update $i add properties/root_device=properties/root_device='{"serial": "xxx"}'
```

然后需要修正logic\_gb，可以重新执行introspection或者手动指定logic\_gb

* 重新执行introspection
  ```
  openstack baremetal introspection bulk start
  ```
* 或者更新logic\_gb
  ```
  ironic node-update <UUID> add properties/local_gb=<NEW VALUE>
  ```

## 4. 定义ceph

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
parameter_defaults:  #已存在的section
  ExtraConfig:       #这是我们要添加的section
    ceph::profile::params::osds:
```



