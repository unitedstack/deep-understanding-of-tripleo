# 部署overcloud 时的排错

---

## overcloud 无法访问169.254.169.254
部署overcloud 时，在进入step1之前卡住，ssh 登录overcloud ，执行`journalctl -u os-collect-config` 发现，os-collect-config 无法连接169.254.169.254. 
手动运行 `curl http://169.254.169.254` 连接超时。
在undercloud 上执行`sudo iptables -nL PREROUTING -t nat` 发现正常。
```
sudo iptables -nL PREROUTING -t nat
You should see something like:
REDIRECT   tcp  --  0.0.0.0/0            169.254.169.254      tcp
dpt:80 redir ports 8775

```

应该是我在一次“no valid host” 的问题处理中把10.0.130.5(dhcp namespace 中的端口)删掉了有关。



## 部署时提示 Message: No valid host was found. There are not enough hosts available
1. 检查节点是否定义了根磁盘
2. 检查node的profile标签，比如profile:compute


## 部署物理机时，只有少数机器可以安装上系统，其余的机器无法安装系统
故障描述：
在ironic部署系统时，所有机器都可以拿到ipxe的地址，并且进去ipxe，但是进入ipxe的系统之后，无法和ironic-api 通信。导致无法系统。

使用ironic node-list查看node状态，发现出问题的机器的Provisioning State全部hang在wait call-back状态

问题分析:
ironic-dnsmasq服务bug。

解决方法：
重启所有ironic服务
```
systemctl restart openstack-ironic*
```



