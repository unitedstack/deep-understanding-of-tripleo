#Troubleshooting ironic

---

## ironic log location
ironic 的 log 有一部分存在 `/var/log` 底下，有的存在 `journald` 中。在排错的时候根据情况排查。
ironic 有两个服务，openstack-ironic-api 和 openstack-ironic-conductor。ironic-inspector 也有两个服务 openstack-ironic-inspector 和 openstack-ironic-inspector-dnsmasq。

比如我们要查`openstack-ironic-inspector-dnsmasq` 的log， 可以使用`jouralctl`来查看。
```
sudo journalctl -u openstack-ironic-inspector -u openstack-ironic-inspector-dnsmasq
```


## 如何替换ironic node 的mac地址

ref:[Troubleshooting Node Management Failures](http://docs.openstack.org/developer/tripleo-docs/troubleshooting/troubleshooting.html#troubleshooting-node-management-failures)