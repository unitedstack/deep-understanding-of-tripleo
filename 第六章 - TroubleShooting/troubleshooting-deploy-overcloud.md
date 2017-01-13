# 部署overcloud 时的排错

---
部署overcloud 时，在进入step1之前卡住，ssh 登录overcloud ，执行`journalctl -u os-collect-config` 发现，os-collect-config 无法连接169.254.169.254. 
手动运行 `curl http://169.254.169.254` 结果相同。
在undercloud 上执行`sudo iptables -nL PREROUTING -t nat` 发现正常。
```
sudo iptables -nL PREROUTING -t nat
You should see something like:
REDIRECT   tcp  --  0.0.0.0/0            169.254.169.254      tcp
dpt:80 redir ports 8775

```

