# TripleO UI

---

## 安装

在TripleO Newton 版中，已经包含了TripleO UI，我们只需要在部署undercloud时,在配置文件中将其启用即可。

**undercloud.conf**

```
# Whether to install the TripleO UI. (boolean value)
enable_ui = true
```

## 登录

TripleO UI 的端口号是3000,通过浏览器访问undercloud controlplane IP 的3000端口号即可。

```
http://10.0.130.31:3000
```
这是TripleO的登录界面，username和password 即undercloud的账户。从stackrc文件中获得。

![](/assets/TripleO-UI-1.png)

