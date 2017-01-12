# undercloud物理环境部署

---
## 1. 修改主机名
```
[root@zhaozhilong ~]# hostname director.ustack.com
[root@zhaozhilong ~]# echo "director.ustack.com" > /etc/hostanme
```

## 2. 创建undercloud的部署用户

```
[root@director~]# useradd stack
[root@director~]# passwd stack # specify a password
```
上面创建了这个用户，然后我们就需要赋予这个用户sudo的权限
```
[root@director~]# echo "stack ALL=(root) NOPASSWD:ALL" | tee -a
/etc/sudoers.d/stack
[root@director~]# chmod 0440 /etc/sudoers.d/stack
```
尝试使用stack这个用户进行登陆
```
[root@director~]# su - stack
[stack@director~]$
```





