定义根磁盘
---
在执行完`openstack baremetal introspection bulk start`之后，根据得到的信息来定义ceph 节点的根磁盘。
根磁盘可以通过以下参数来指定,下文以serial来展示如何定义根磁盘。
```
model (String): Device identifier.
vendor (String): Device vendor.
serial (String): Disk serial number.
wwn (String): Unique storage identifier.
size (Integer): Size of the device in GB.
```

1. 下载introspection的信息。它的信息存在swift中
```bash
$ mkdir swift-data
$ cd swift-data
$ export IRONIC_DISCOVERD_PASSWORD=`sudo grep admin_password /etc/ironic-inspector/inspector.conf | egrep -v '^#'  | awk '{print $NF}'`
$ for node in $(ironic node-list | grep -v UUID| awk '{print $2}'); do swift -U service:ironic -K $IRONIC_DISCOVERD_PASSWORD download ironic-inspector inspector_data-$node; done
```
所有节点的introspection的信息以各自的UUID结尾。
```
$ ls -1
inspector_data-15fc0edc-eb8d-4c7f-8dc0-a2a25d5e09e3
inspector_data-46b90a4d-769b-4b26-bb93-50eaefcdb3f4
inspector_data-662376ed-faa8-409c-b8ef-212f9754c9c7
inspector_data-6fc70fe4-92ea-457b-9713-eed499eda206
inspector_data-9238a73a-ec8b-4976-9409-3fcff9a8dca3
inspector_data-9cbfe693-8d55-47c2-a9d5-10e059a14e07
inspector_data-ad31b32d-e607-4495-815c-2b55ee04cdb1
inspector_data-d376f613-bc3e-4c4b-ad21-847c4ec850f8
```

2. 显示每个节点的磁盘信息
```
$ for node in $(ironic node-list | grep -v UUID| awk '{print $2}'); do echo "NODE: $node" ; cat inspector_data-$node | jq '.inventory.disks' ; echo "-----" ; done
```
输出类似于这样：
```
NODE: 46b90a4d-769b-4b26-bb93-50eaefcdb3f4
[
  {
    "size": 1000215724032,
    "vendor": "ATA",
    "name": "/dev/sda",
    "model": "WDC WD1002F9YZ",
    "wwn": "0x0000000000000001",
    "serial": "WD-000000000001"
  },
  {
    "size": 1000215724032,
    "vendor": "ATA",
    "name": "/dev/sdb",
    "model": "WDC WD1002F9YZ",
    "wwn": "0x0000000000000002",
    "serial": "WD-000000000002"
  },
  {
    "size": 1000215724032,
    "vendor": "ATA",
    "name": "/dev/sdc",
    "model": "WDC WD1002F9YZ",
    "wwn": "0x0000000000000003",
    "serial": "WD-000000000003"
  },
]
```
3. 使用serial属性为节点指定根磁盘
```
$ ironic node-update 97e3f7b3-5629-473e-a187-2193ebe0b5c7 \
add properties/root_device='{"serial": "WD-000000000002"}'
```
