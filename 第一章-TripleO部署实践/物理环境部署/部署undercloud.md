# undercloud物理环境部署

---
## 1. 修改主机名
```
$ hostname # 查看基础的主机名
$ hostname -f # 查看完成的主机名
```
如果当前的主机名不是你想要的，你可以自己定义你的undercloud的主机名:

```
[root@zhaozhilong ~]# hostnamectl set-hostname director.ustack.com
[root@zhaozhilong ~]# hostnamectl set-hostname --transient director.ustack.com
```

## 2. 创建undercloud的部署用户

```
[root@director ~]# useradd stack
[root@director ~]# passwd stack # specify a password
```
上面创建了这个用户，然后我们就需要赋予这个用户sudo的权限
```
[root@director ~]# echo "stack ALL=(root) NOPASSWD:ALL" | tee -a
/etc/sudoers.d/stack
[root@director ~]# chmod 0440 /etc/sudoers.d/stack
```
尝试使用stack这个用户进行登陆
```
[root@director ~]# su - stack
[stack@director ~]$
```

## 3. 配置yum源

我们这边使用ustack公司内部源:
```
[tripleO]
name = tripleO
baseurl = http://tripleO.ustack.com/repo/Newton
gpgcheck = 0

[tripleO-dep]
name = tripleO-dep
baseurl = http://tripleO.ustack.com/repo/Newton-dep
gpgcheck = 0
```
你也可以使用Centos社区的源:
```
[tripleO-centos]
name = tripleO-centos
http://mirror.centos.org/centos/7/cloud/x86_64/openstack-newton/
gpgcheck = 0
```

## 4. 更新undercloud
为了确保undercloud的kernel版本和上游版本一直，还有一些边缘组件也需要和上游同步，我们需要更新我们的系统：
```
[stack@director ~]$ sudo yum update -y
```
完成之后，就可以重启系统了
```
[stack@director ~]$ sudo reboot
```

## 5. 安装tripleO
上面的过程都是在准备我们的基础环境，现在我们需要安装我们的tripleO.
```
[stack@director ~]$ sudo yum install -y python-tripleoclient
```


## 6. 编写undercloud的配置文件




