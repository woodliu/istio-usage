## Istio多集群

参考自[官方文档](https://preliminary.istio.io/latest/docs/setup/install/multicluster/)。

[TOC]

### 复制控制面

本节将使用多个主集群(带控制面的集群)来部署Istio[多集群](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#multiple-clusters)，每个集群都有自己的[控制面](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#control-plane-models)，集群之间使用gateway进行通信。

由于不使用共享的控制面来管理网格，因此这种配置下，每个集群都有自己的控制面来管理后端应用。为了策略执行和安全目的，所有群集都处于一个共享的管理控制之下。

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

   - 为生成的CA创建一个kubernetes secret。它与[在Istio中插入自定义的CA](https://preliminary.istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/)一文中的方式类似。

     > 生产中不能使用samples 目录中的证书，有安全风险。

     ```shell
     $ kubectl create namespace istio-system
     $ kubectl create secret generic cacerts -n istio-system \
         --from-file=samples/certs/ca-cert.pem \
         --from-file=samples/certs/ca-key.pem \
         --from-file=samples/certs/root-cert.pem \
         --from-file=samples/certs/cert-chain.pem
     ```

   - 部署Istio

     ```shell
     $ istioctl install -f manifests/examples/multicluster/values-istio-multicluster-gateways.yaml
     ```

#### 配置DNS

为远端集群中的服务提供DNS解析将允许现有应用程序无需修改即可运行，因为应用程序通常希望访问通过DNS解析出的IP。Istio本身并不需要DNS在服务之间路由请求。本地服务会共享一个共同的DNS前缀(即，`svc.cluster.local`)。kubernetes DNS为这些服务提供了DNS解析。

为了给远端集群提供一个类似的服务配置，需要使用格式`<name>.<namespace>.global`来命名远端集群的服务。Istio附带了一个Core DNS服务，可以为这些服务提供DNS解析。为了使用该DNS，kubernetes的DNS必须配置为`.global`的域名存根。

在需要调用远程的服务的集群中创建或更新一个现有的ConfigMaps。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-dns
  namespace: kube-system
data:
  stubDomains: |
    {"global": ["$(kubectl get svc -n istio-system istiocoredns -o jsonpath={.spec.clusterIP})"]}
EOF
```

#### 配置应用服务

一个集群中的服务如果需要被远端集群访问，就需要在远端集群中配置一个`ServiceEntry`。service entry使用的host格式为`<name>.<namespace>.global`，name和namespace分别对应服务的name和namespace。

为了演示跨集群访问，在一个集群中配置sleep服务，使用该服务访问另一个集群中的httpbin服务。

- 选择两个Istio集群，分别为`kind`和`kind2`

- 使用如下命令列出集群的上下文：

  ```shell
  # kubectl config get-contexts
  CURRENT   NAME         CLUSTER      AUTHINFO     NAMESPACE
  *         kind-kind    kind-kind    kind-kind
            kind-kind2   kind-kind2   kind-kind2
  ```

- 使用环境变量保存集群的上下文名称：

  ```shell
  # export CTX_CLUSTER1=$(kubectl config view -o jsonpath='{.contexts[0].name}')
  # export CTX_CLUSTER2=$(kubectl config view -o jsonpath='{.contexts[1].name}')
  # echo "CTX_CLUSTER1 = ${CTX_CLUSTER1}, CTX_CLUSTER2 = ${CTX_CLUSTER2}"
  CTX_CLUSTER1 = kind-kind, CTX_CLUSTER2 = kind-kind2
  ```

#### 配置用例服务

1. 在`kind`集群中部署`sleep`应用

   ```shell
   $ kubectl create --context=$CTX_CLUSTER1 namespace foo
   $ kubectl label --context=$CTX_CLUSTER1 namespace foo istio-injection=enabled
   $ kubectl apply --context=$CTX_CLUSTER1 -n foo -f samples/sleep/sleep.yaml
   $ export SLEEP_POD=$(kubectl get --context=$CTX_CLUSTER1 -n foo pod -l app=sleep -o jsonpath={.items..metadata.name})
   ```

2. 在`kind2`集群中部署`httpbin`应用

   ```shell
   $ kubectl create --context=$CTX_CLUSTER2 namespace bar
   $ kubectl label --context=$CTX_CLUSTER2 namespace bar istio-injection=enabled
   $ kubectl apply --context=$CTX_CLUSTER2 -n bar -f samples/httpbin/httpbin.yaml
   ```

3. 暴露`cluster2`的网关地址

   本地部署的kubernetes由于没有loadBalancer，因此使用nodeport方式

   ```shell
   export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
   export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
   export TLS_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tls")].nodePort}')
   ```

4. 为了允许`cluster1`中的`sleep`访问`cluster2`中的`httpbin`，需要在`cluster1`中为`httpbin`创建一个service entry。service entry的host名称的格式应该为`<name>.<namespace>.global`，name和namespace分别对应远端服务的name和namespace。

   为了让DNS解析`.global`域下的服务，需要给这些服务分配IP地址。

















