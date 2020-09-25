## Istio多集群

参考自[官方文档](https://preliminary.istio.io/latest/docs/setup/install/multicluster/)。

[TOC]

### 复制控制面

本节将使用多个主集群(带控制面的集群)来部署Istio[多集群](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#multiple-clusters)，每个集群都有自己的[控制面](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#control-plane-models)，集群之间使用gateway进行通信。

由于不使用共享的控制面来管理网格，因此这种配置下，每个集群都使用自己的控制面来管理后端应用。为了策略执行和安全目的，所有群集都处于一个共享的管理控制之下。

通过复制共享服务和命名空间，并在所有集群中使用一个公共的根CA证书，可以实现单个Istio服务网格跨集群通信。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200925125143585-1811497649.png)

#### 要求

- 两个或多个kubernees集群，版本为1.17，1.18，1.19
- 在每个kubernetes集群上[部署Istio控制面](https://preliminary.istio.io/latest/docs/setup/install/istioctl/)
- 每个集群中的istio-ingressgateway的服务IP地址必须能够被所有的集群访问，理想情况下，使用L4网络负载平衡器（NLB）。
- 根CA。跨集群通信需要在服务之间使用mutual TLS。为了在跨集群通信时启用mutual TLS，每个集群的Istio CA必须配置使用共享的CA证书生成的中间CA。出于演示目的，将会使用Istio安装的`samples/certs`目录下的根CA证书

#### 在每个集群中部署Istio控制面

1. 使用自己的根CA为每个集群生成中间CA证书，使用共享的根CA来为跨集群的通信启用mutual TLS。

   处于演示目的，下面使用了istio样例目录中的证书。在真实部署时，应该为每个集群选择不同的CA证书，这些证书由一个共同的根CA签发。

2. 在每个集群中运行如下命令来为所有集群部署相同的Istio控制面。

   - 













