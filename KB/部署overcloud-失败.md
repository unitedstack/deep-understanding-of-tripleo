## 部署overcloud失败记录

## 1. 执行 openstack overcloud deploy 时失败
执行下面的命令，获取physical_resource_id。
```
openstack stack resource list overcloud -n 5|grep -vi "complete"
```

通过`heat resource-show <physical_resource_id>` 查看输出。
```
[stack@undercloud-0 ~]$ [stack@undercloud-0 ~]$ heat deployment-show 001bbb95-cbc8-4779-92ae-bbe19b148b49
WARNING (shell) "heat deployment-show" is deprecated, please use "openstack software deployment show" instead
{
  "status": "FAILED", 
  "server_id": "ab8d95ab-dd30-4ad0-ba45-1a4d4df3d5f0", 
  "config_id": "e4d47391-283b-4b1c-aea4-f7e4192fd2a3", 
  "output_values": {
    "deploy_stdout": "ERROR: Unsupported file format.\n", 
    "deploy_stderr": "curl: (3) [globbing] error: bad range specification after pos 9\n", 
    "deploy_status_code": 1
  }, 
  "creation_time": "2016-11-08T05:21:05Z", 
  "updated_time": "2016-11-08T05:21:50Z", 
  "input_values": {}, 
  "action": "CREATE", 
  "status_reason": "deploy_status_code : Deployment exited with non-zero status code: 1", 
  "id": "001bbb95-cbc8-4779-92ae-bbe19b148b49"
}
```


参考:[overcloud deployment failed](https://bugzilla.redhat.com/show_bug.cgi?id=1310956)