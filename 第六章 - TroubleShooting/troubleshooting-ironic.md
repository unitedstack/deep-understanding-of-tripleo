#Troubleshooting ironic

---

## ironic log location
ironic 的 log 有一部分存在 `/var/log` 底下，有的存在 `journald` 中。在排错的时候根据情况排查。
ironic 有两个服务，openstack-ironic-api 和 openstack-ironic-conductor。ironic-inspector 也有两个服务 openstack-ironic-inspector 和 openstack-ironic-inspector-dnsmasq。

比如我们要查`openstack-ironic-inspector-dnsmasq` 的log， 可以使用`jouralctl`来查看。
```
sudo journalctl -u openstack-ironic-inspector -u openstack-ironic-inspector-dnsmasq
```


## 如何修改 ironic node 注册时的mac地址

1. 查出node的port的UUID。
```
ironic node-port-list <NODE UUID>
```

2. 执行port-update 更行mac地址
```
ironic port-update <PORT UUID> replace address=<NEW MAC>
```


## 修改注册node时注册的ipmi 地址
```
ironic node-update <NODE UUID> replace driver_info/ipmi_address=<NEW IPMI ADDRESS>
```

ref:[Troubleshooting Node Management Failures](http://docs.openstack.org/developer/tripleo-docs/troubleshooting/troubleshooting.html#troubleshooting-node-management-failures)


## introspection 单个node
除了bulk introspection 之外，还可以对单个ironic node进行introspection。
首先要将node设为manage状态，
1. 将node设为 manage 状态
```
ironic node-set-provision-state UUID manage
```
2. 开始introspection
```
openstack baremetal introspection start UUID
```
3. 查询introspection状态
```
openstack baremetal introspection status UUID
```
4. 将node改为available
```
ironic node-set-provision-state UUID provide
```
