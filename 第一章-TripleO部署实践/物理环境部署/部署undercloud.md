# undercloud物理环境部署

---

## 1. 创建undercloud的部署用户

```
[root@zhaozhilong ~]# useradd stack
[root@zhaozhilong ~]# passwd stack # specify a password
```
上面创建了这个用户，然后我们就需要赋予这个用户sudo的权限
```
[root@zhaozhilong ~]# echo "stack ALL=(root) NOPASSWD:ALL" | tee -a
/etc/sudoers.d/stack
[root@zhaozhilong ~]# chmod 0440 /etc/sudoers.d/stack
```
尝试使用stack这个用户进行登陆
```
[root@zhaozhilong ~]# su - stack
[stack@zhaozhilong  ~]$
```





