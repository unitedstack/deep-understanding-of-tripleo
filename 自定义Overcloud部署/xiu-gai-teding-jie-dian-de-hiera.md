修改特定节点的Hiera
---
在某些情况下，会需要修改未暴露在overcloud.yaml中的puppet参数。通过NodeExtraConfig可以实现传递任意额外的配置给特定的role。

#修改配置
修改计算节点的某项未暴露在overcloud中的配置：
```
cat > compute_params.yaml << EOF
parameter_defaults:
    NovaComputeExtraConfig:
      nova::compute::reserved_host_memory: some_value
EOF

openstack overcloud deploy -e compute_params.yaml
```

- ExtraConfig: 传递hieradata到所有role。
- NovaComputeExtraConfig: 传递hieradata到所有计算节点。
- ControllerExtraConfig: 传递hieradata到所有控制节点。
- BlockStorageExtraConfig: 传递hieradata到所有Cinder节点。
- ObjectStorageExtraConfig: 传递hieradata到所有swift节点。
- CephStorageExtraConfig: 传递hieradata到所有Ceph节点。