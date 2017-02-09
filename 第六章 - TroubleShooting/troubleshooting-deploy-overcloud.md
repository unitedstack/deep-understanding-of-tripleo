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

