## 部署overcloud失败记录

## 1. 执行 openstack overcloud deploy 时失败
执行下面的命令，获取physical_resource_id。
```
openstack stack resource list overcloud -n 5|grep -vi "complete"
```

通过`heat resource-show <physical_resource_id>` 查看输出。


参考:[overcloud deployment failed](https://bugzilla.redhat.com/show_bug.cgi?id=1310956)