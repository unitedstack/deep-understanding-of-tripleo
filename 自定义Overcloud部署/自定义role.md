# 自定义Role

---

overcloud 是由一系列角色组成的，比如controller、compute、Storage。每个默认角色都是在TripleO Heat Template 核心代码中定义的一组service。但是 TripleO Heat Template 还可以支持:
- 创建自定义role
- 从role 中添加或移除 service

这篇文章描述了如何自定义role、组合service。

## 指南和限制



