undercloud 虚拟环境部署（公网）
---
添加Newton的repo
```
sudo curl -L -o /etc/yum.repos.d/delorean-newton.repo https://trunk.rdoproject.org/centos7-newton/current/delorean.repo

sudo curl -L -o /etc/yum.repos.d/delorean-deps-newton.repo http://trunk.rdoproject.org/centos7-newton/delorean-deps.repo

sudo yum -y install --enablerepo=extras centos-release-ceph-jewel
sudo sed -i -e 's%gpgcheck=.*%gpgcheck=0%' /etc/yum.repos.d/CentOS-Ceph-Jewel.repo
```

安装yum-plugin-priorites
```
sudo yum -y install yum-plugin-priorities

```


安装 TripleO CLI
```
sudo yum install -y python-tripleoclient

```


