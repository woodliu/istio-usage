# Istio的流量管理(实操)

使用官方的[Bookinfo](https://istio.io/docs/examples/bookinfo/)应用进行测试。涵盖官方文档[Traffic Management](https://istio.io/docs/tasks/traffic-management/)章节。

[TOC]

## 部署Bookinfo应用

### Bookinfo应用说明

官方提供的测试应用如下，包含如下4个组件：

- `productpage`： `productpage` 服务会调用`details` 和 `reviews`来填充web页面.
- `details`：`details` 服务包含book信息.
- `reviews`： `reviews` 服务包含书评，它会调用 `ratings` 服务.
- `ratings`：`ratings` 服务包与书评相关的含排名信息

`reviews` 包含3个版本：

- v1版本不会调用 `ratings` 服务.
- v2版本会调用 `ratings` 服务,并按照1到5的黑色星展示排名
- v2版本会调用 `ratings` 服务,并按照1到5的红色星展示排名

![](./images/bookinfo_application.png)

### 部署

Bookinfo应用部署在`default`命名空间下，使用自动注入sidecar的方式：

- 通过如下命令在`default`命名空间(当然也可以部署在其他命名空间下面，Bookinfo配置文件中并没有指定部署的命名空间)中启用自动注入sidecar:

  ```shell
  $ cat <<EOF | oc -n <target-namespace> create -f -
  apiVersion: "k8s.cni.cncf.io/v1"
  kind: NetworkAttachmentDefinition
  metadata:
    name: istio-cni
  EOF
  ```

  ```shell
  $ kubectl label namespace default istio-injection=enabled
  ```

- 切换在`default`命名空间下，部署Bookinfo应用：

  ```shell
  $ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
  ```

  等待一段时间，Bookinfo的所有pod就可以成功启动，查看pod和service：

  ```shell
  $ oc get pod
  NAME                              READY   STATUS    RESTARTS   AGE
  details-v1-78d78fbddf-5mfv9       2/2     Running   0          2m27s
  productpage-v1-85b9bf9cd7-mfn47   2/2     Running   0          2m27s
  ratings-v1-6c9dbf6b45-nm6cs       2/2     Running   0          2m27s
  reviews-v1-564b97f875-ns9vz       2/2     Running   0          2m27s
  reviews-v2-568c7c9d8f-6r6rq       2/2     Running   0          2m27s
  reviews-v3-67b4988599-ddknm       2/2     Running   0          2m27s
  ```

  ```shell
  $ oc get svc                                              
  NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
  details       ClusterIP      10.84.97.183   <none>        9080/TCP   3m33s
  kubernetes    ClusterIP      10.84.0.1      <none>        443/TCP    14d
  productpage   ClusterIP      10.84.98.111   <none>        9080/TCP   3m33s
  ratings       ClusterIP      10.84.237.68   <none>        9080/TCP   3m33s
  reviews       ClusterIP      10.84.39.249   <none>        9080/TCP   3m33s
  ```

  使用如下命令判断Bookinfo应用是否正确安装：

  ```shell
  $ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
  
  <title>Simple Bookstore App</title> #返回的结果
  ```

  也可以直接通过svc的endpoint进行访问

  ```shell
  $ oc describe svc productpage|grep Endpoint
  Endpoints:         10.83.1.85:9080
  ```

  ```shell
  $ curl -s 10.83.1.85:9080/productpage | grep -o "<title>.*</title>"
  ```

  可在openshift中创建`router`进行访问(将${HOST_NAME}替换为实际的主机名)

  ```yaml
  kind: Route
  apiVersion: route.openshift.io/v1
  metadata:
    name: productpage
    namespace: default
    labels:
      app: productpage
      service: productpage
    annotations:
      openshift.io/host.generated: 'true'
  spec:
    host: ${HOST_NAME}
    to:
      kind: Service
      name: productpage
      weight: 100
    port:
      targetPort: http
    wildcardPolicy: None
  ```

  > 此处先不根据官方文档配置ingress，后续再配置

- 配置默认的destination rules

  配置带mutual TLS(一开始学习istio时不建议配置)

  ```shell
  $ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
  ```

  配置不带mutual TLS

  ```shell
  $ kubectl get destinationrules -o yaml
  ```

  获取配置的destination rules

  ```shell
  $ kubectl get destinationrules -o yaml
  ```

  获取到的destination rules如下，注意默认安装下，除了`reviews`外的service只有`v1`版本

  ```yaml
  - apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      annotations:
        ...
      name: details
      namespace: default
    spec:
      host: details    #对应kubernetes service "details"
      subsets:
      - labels:        #实际的details的deployment只有一个标签"version: v1"
          version: v1
        name: v1
      - labels:
          version: v2
        name: v2
  	  
  	  
  - apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      annotations:
        ...
      name: productpage
      namespace: default
    spec:
      host: productpage
      subsets:
      - labels:
          version: v1
        name: v1
  	  
  	  
  - apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      annotations:
        ...
      name: ratings
      namespace: default
    spec:
      host: ratings
      subsets:
      - labels:
          version: v1
        name: v1
      - labels:
          version: v2
        name: v2
      - labels:
          version: v2-mysql
        name: v2-mysql
      - labels:
          version: v2-mysql-vm
        name: v2-mysql-vm
  	  
  	  
  - apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      annotations:
        ...
      name: reviews     # kubernetes service "reviews"实际中有3个版本
      namespace: default
    spec:
      host: reviews
      subsets:
      - labels:
          version: v1
        name: v1
      - labels:
          version: v2
        name: v2
      - labels:
          version: v3
        name: v3
  ```

### 卸载

使用如下命令可以卸载Bookinfo

```shell
$ samples/bookinfo/platform/kube/cleanup.sh
```

## 流量管理

### [请求路由](https://istio.io/docs/tasks/traffic-management/request-routing/)

下面展示如何根据微服务的多个版本动态地路由请求。





### [故障注入](https://istio.io/docs/tasks/traffic-management/fault-injection/)





















