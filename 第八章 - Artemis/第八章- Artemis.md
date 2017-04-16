Artemis
---
Artemis 是用于定制UOS4.0部署的puppet模块。有两个作用：配置基线，集成自研项目。


Artemis有以下组件来配置UOS4：
- nova
  - compute
  - conductor
- cinder
- neutron
- halo
 - mirana
 - kunkka




下面逐一说明各组件所管理的配置项
nova:
cpu_allocation_ratio
ram_allocation_ratio
disk_allocation_ratio