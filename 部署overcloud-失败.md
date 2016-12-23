## 部署overcloud失败记录

## 1. 部署overcloud时卡住

![](/assets/部署overcloud 卡住 1.png)

环境中有3台控制节点3台计算节点3台存储节点

查看log，有类似如下报错
```
Feb 22 23:06:26 overcloud-controller-0.localdomain os-collect-config[10347]: [2016-02-22 23:06:26,639] (os-refresh-config) [INFO] Completed phase post-configure
Feb 22 23:06:28 overcloud-controller-0.localdomain os-collect-config[10347]: 2016-02-22 23:06:28.294 10347 WARNING os-collect-config [-] Source [request] Unavailable.
Feb 22 23:06:28 overcloud-controller-0.localdomain os-collect-config[10347]: 2016-02-22 23:06:28.295 10347 WARNING os_collect_config.local [-] /var/lib/os-collect-config/local-data not found. Skipping
Feb 22 23:06:28 overcloud-controller-0.localdomain os-collect-config[10347]: 2016-02-22 23:06:28.295 10347 WARNING os_collect_config.local [-] No local metadata found (['/var/lib/os-collect-config/local-data'])
Feb 22 23:06:28 overcloud-controller-0.localdomain os-collect-config[10347]: 2016-02-22 23:06:28.295 10347 WARNING os_collect_config.zaqar [-] No auth_url 
```
最终查到其中有一台计算节点2，与其他节点网络不通（部署网络除外）。