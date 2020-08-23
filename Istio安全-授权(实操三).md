# Istio安全-授权

[TOC]

## 授权HTTP流量

本节展示如何在istio网格中授权HTTP流量。

部署[Bookinfo](https://istio.io/latest/docs/examples/bookinfo/#deploying-the-application) 

### 为使用HTTP流量的负载配置访问控制

本任务展示了如何使用istio的授权设置访问控制。首先，使用简单的`deny-all`策略拒绝所有到负载的请求，然后增量地授权到负载的访问。

1. 在`default`命名空间中创建`deny-all`策略。该策略没有`selector`字段，将会应用到default命名空间中的所有负载上。`sepc:`字段为空`{}`，意味着禁止所有的流量。

