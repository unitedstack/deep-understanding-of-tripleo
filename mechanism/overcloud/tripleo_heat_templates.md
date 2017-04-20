## TripleO-Heat-Templates解析

TripleO的Heat Template是整个TripleO的核心，它定义了一系列复杂的模板，运用了Heat中一些高级的语法，抽象出了一个个嵌套的stack，来完成OverCloud中安装系统，网络配置，服务配置，高可用等一系列部署操作。本节重点解析下这些Template，讲解下TripleO中是如何使用Heat来安装部署OverCloud的。

### Heat语法

在TripleO中使用到了很多Heat中的语法，需要理解这些基本的语法才能够理解Heat Template。Heat中自定义了一套描述式语言，称为HOT(Heat Orchestration Template)，来描述将要创建的stack。Stack在Heat中表示