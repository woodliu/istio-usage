# Istio的流量管理(实操三)(istio 系列五)

涵盖官方文档[Traffic Management](https://istio.io/docs/tasks/traffic-management/)章节中的egress部分。

[TOC]

## 访问外部服务

由于启用了istio的pod的出站流量默认都会被重定向到代理上，因此对集群外部URL的访问取决于代理的配置。默认情况下，Envoy代理透传对未知服务的访问，虽然这种方式为新手提供了便利，但最好配置更严格的访问控制。

本节展示使用如下三种方式访问外部服务:

1. 允许Envoy代理透传到网格外部的服务
2. 配置[service entries](https://istio.io/docs/reference/config/networking/service-entry/)来访问外部访问
3. 透传某一个IP端的请求

### 部署















## Egress TLS Origination(Egress TLS源)





































