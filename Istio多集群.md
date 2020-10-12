## Istio多集群

参考自[官方文档](https://istio.io/latest/docs/setup/install/multicluster/gateways/)。

[TOC]

### 复制控制面

本节将使用多个主集群(带控制面的集群)来部署Istio[多集群](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#multiple-clusters)，每个集群都有自己的[控制面](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#control-plane-models)，集群之间使用gateway进行通信。

由于不使用共享的控制面来管理网格，因此这种配置下，每个集群都有自己的控制面来管理后端应用。为了策略执行和安全目的，所有的群集都处于一个公共的管理控制之下。

通过复制共享服务和命名空间，并在所有集群中使用一个公共的根CA证书，可以实现单个Istio服务网格跨集群通信。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200925125143585-1811497649.png)

#### 要求

- 两个或多个kubernees集群，版本为1.17，1.18，1.19
- 在每个kubernetes集群上[部署Istio控制面](https://preliminary.istio.io/latest/docs/setup/install/istioctl/)
- 每个集群中的`istio-ingressgateway`的服务IP地址必须能够被所有的集群访问，理想情况下，使用L4网络负载平衡器（NLB）。
- 根CA。跨集群通信需要在服务之间使用mutual TLS。为了在跨集群通信时启用mutual TLS，每个集群的Istio CA必须配置使用共享的CA证书生成的中间CA。出于演示目的，将会使用Istio的`samples/certs`安装目录下的根CA证书

#### 在每个集群中部署Istio控制面

1. 使用自己的根CA为每个集群生成中间CA证书，使用共享的根CA来为跨集群的通信启用mutual TLS。

   出于演示目的，下面使用了istio样例目录中的证书。在真实部署时，应该为每个集群选择不同的CA证书，这些证书由一个共同的根CA签发。

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

   - 部署Istio，下面命令会在`istio-system`命名空间中创建一个pod `istiocoredns`，用于提供外部服务的DNS解析，其配置文件如下：

     ```shell
     # cat Corefile
     .:53 {
           errors
           health
     
           # Removed support for the proxy plugin: https://coredns.io/2019/03/03/coredns-1.4.0-release/
           grpc global 127.0.0.1:8053
           forward . /etc/resolv.conf {
             except global
           }
     
           prometheus :9153
           cache 30
           reload
         }
     ```
     
     ```shell
     $ istioctl install -f manifests/examples/multicluster/values-istio-multicluster-gateways.yaml
     ```

#### 配置DNS

当为远端集群中的服务提供DNS解析时，现有应用程序无需修改即可运行，因为应用程序通常会访问通过DNS解析出的IP。Istio本身并不需要DNS在服务之间路由请求。本地服务会共享一个共同的DNS前缀(即，`svc.cluster.local`)。kubernetes DNS为这些服务提供了DNS解析。

为了给远端集群提供一个类似的服务配置，需要使用格式`<name>.<namespace>.global`来命名远端集群中的服务。Istio附带了一个Core DNS服务，可以为这些服务提供DNS解析。为了使用该DNS，kubernetes的DNS必须配置为`.global`的域名存根。

在每个需要调用远程的服务的集群中创建或更新一个现有的k8s的ConfigMap，本环境中使用的coredns为1.7.0版本，使用的配置文件如下：

> 注意不能直接采用[官方](https://istio.io/latest/docs/setup/install/multicluster/gateways/#setup-dns)配置文件，可能会导致k8s的coredns无法正常启动。正确做法是在kube-system命名空间下获取k8s coredns的configmap配置，然后在后面追加global域有关的配置即可。
>
> 另外使用如下命令apply之后，k8s的coredns可能并不会生效，可以手动重启k8s的dns来使其生效。注意如下配置需要在cluster1和cluster2中**同时**生效

```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 { #k8s的coredns的原始配置
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
    global:53 { #新增了对global的解析，将其转发到istio-system下面的istiocoredns服务，即上面istio创建的coredns
        errors
        cache 30
        forward . $(kubectl get svc -n istio-system istiocoredns -o jsonpath={.spec.clusterIP}):53
    }
EOF
```

> 可以使用如下方式判断cluster1的DNS解析是否正确：
>
> - 首先在cluster1中的sleep容器中通过istio的coredns解析`httpbin.bar.global`，10.96.199.197为istio的coredns的service
>
>   ```shell
>   # nslookup -type=a httpbin.bar.global 10.96.199.197
>   Server:         10.96.199.197
>   Address:        10.96.199.197:53
>   
>   Name:   httpbin.bar.global
>   Address: 240.0.0.2 #解析成功
>   ```
>
> - 在cluster1中的sleep容器中通过k8s的coredns解析`httpbin.bar.global`，10.96.0.10为k8s的coredns的service，可以看到k8s的coredns将`httpbin.bar.global`的解析转发给了istio的coredns，并且解析成功。
>
>   ```shell
>   # nslookup -type=a httpbin.bar.global 10.96.0.10
>   Server:         10.96.0.10
>   Address:        10.96.0.10:53
>   
>   Name:   httpbin.bar.global
>   Address: 240.0.0.2 #解析成功
>   ```
>
> DNS的解析路径为：sleep容器的/etc/resolv.conf-->k8s coredns-->istio coredns-->istio-coredns-plugin
>
> [istio-coredns-plugin](https://github.com/istio-ecosystem/istio-coredns-plugin)是istio的CoreDNS gRPC插件，用于从Istio `ServiceEntries`中提供DNS记录。(**注：该插件将集成到Istio 1.8的sidecar中，后续将会被废弃**)
>
> 可以在istio-coredns-plugin的log日志中查看到对global域的操作：
>
> ```shell
> # crictl inspect e8d3f73c4d38d|grep "logPath"
>     "logPath": "/var/log/pods/istio-system_istiocoredns-75dd7c7dc8-cg55l_8c960b74-419c-44e8-8992-58293e36d6fd/istio-coredns-plugin/0.log"
> ```
>
> 例如该插件会读取下面创建的到`httpbin.bar.global`的`ServiceEntries`，并将其做DNS映射：
>
> ```
> ... info    Reading service entries at 2020-10-09 17:53:38.710306063 +0000 UTC m=+19500.216843506
> ... info    Have 1 service entries
> ... info    adding DNS mapping: httpbin.bar.global.->[240.0.0.2]
> ```

#### 配置应用服务

一个集群中的服务如果需要被远端集群访问，就需要在远端集群中配置一个`ServiceEntry`。service entry使用的host格式为`<name>.<namespace>.global`，name和namespace分别对应服务的name和namespace。

为了演示跨集群访问，在一个集群中配置sleep服务，使用该服务访问另一个集群中的httpbin服务。

- 选择两个Istio集群，分别为`cluster1 `和`cluster2`

- 使用如下命令列出集群的上下文：

  ```shell
  # kubectl config get-contexts
  CURRENT   NAME            CLUSTER         AUTHINFO        NAMESPACE
  *         kind-cluster1   kind-cluster1   kind-cluster1
            kind-cluster2   kind-cluster2   kind-cluster2
  ```

- 使用环境变量保存集群的上下文名称：

  ```shell
  # export CTX_CLUSTER1=$(kubectl config view -o jsonpath='{.contexts[0].name}')
  # export CTX_CLUSTER2=$(kubectl config view -o jsonpath='{.contexts[1].name}')
  # echo "CTX_CLUSTER1 = ${CTX_CLUSTER1}, CTX_CLUSTER2 = ${CTX_CLUSTER2}"
  CTX_CLUSTER1 = kind-cluster1, CTX_CLUSTER2 = kind-cluster2
  ```

##### 配置用例服务

1. 在`cluster1`集群中部署`sleep`应用

   ```shell
   $ kubectl create --context=$CTX_CLUSTER1 namespace foo
   $ kubectl label --context=$CTX_CLUSTER1 namespace foo istio-injection=enabled
   $ kubectl apply --context=$CTX_CLUSTER1 -n foo -f samples/sleep/sleep.yaml
   $ export SLEEP_POD=$(kubectl get --context=$CTX_CLUSTER1 -n foo pod -l app=sleep -o jsonpath={.items..metadata.name})
   ```

2. 在`cluster2`集群中部署`httpbin`应用

   ```shell
   $ kubectl create --context=$CTX_CLUSTER2 namespace bar
   $ kubectl label --context=$CTX_CLUSTER2 namespace bar istio-injection=enabled
   $ kubectl apply --context=$CTX_CLUSTER2 -n bar -f samples/httpbin/httpbin.yaml
   ```

3. 暴露`cluster2`的网关地址

   本地部署的kubernetes由于没有loadBalancer，因此使用nodeport方式(如果使用kind部署kubernetes，此时需要手动修改service istio-ingressgateway的nodeport，使其与kind暴露的端口一致)。

   ```shell
   export INGRESS_PORT=$(kubectl -n istio-system --context=$CTX_CLUSTER2 get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
   export SECURE_INGRESS_PORT=$(kubectl -n istio-system --context=$CTX_CLUSTER2 get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
   export TLS_INGRESS_PORT=$(kubectl -n istio-system --context=$CTX_CLUSTER2 get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tls")].nodePort}')
   ```

   INGRESS_HOST的获取方式[如下](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)：

   ```shell
   export INGRESS_HOST=$(kubectl --context=$CTX_CLUSTER2 get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
   ```

4. 为了允许`cluster1`中的`sleep`访问`cluster2`中的`httpbin`，需要在`cluster1`中为`httpbin`创建一个service entry。service entry的host名称的格式应该为`<name>.<namespace>.global`，name和namespace分别对应远端服务的name和namespace。

   为了让DNS解析`.global`域下的服务，需要给这些服务分配IP地址。

   > 每个.global DNS域下的服务都必须在集群中拥有唯一的IP。

   如果global服务已经有了实际的VIPs，那么可以直接使用这类地址，否则建议使用范围为240.0.0.0/4的E类IP地址。应用使用这些IP处理流量时，流量会被sidecar捕获，并路由到合适的远端服务。

   > 不能使用多播地址(224.0.0.0 ~ 239.255.255.255)，因为默认情况下不会有到达这些地址的路由。同时也不能使用环回地址(127.0.0.0/8)，因为发往该地址的流量会被重定向到sidecar的inbound listener。

   ```yaml
   $ kubectl apply --context=$CTX_CLUSTER1 -n foo -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: httpbin-bar
   spec:
     hosts:
     # must be of form name.namespace.global
     - httpbin.bar.global
     # Treat remote cluster services as part of the service mesh
     # as all clusters in the service mesh share the same root of trust.
     location: MESH_INTERNAL #标注为网格内的服务
     ports: # 对端bar服务的端口
     - name: http1
       number: 8000
       protocol: http
     resolution: DNS
     addresses: # 下面指定了httpbin.bar.global:8000服务对应的一个后端${INGRESS_HOST}:30615
     # the IP address to which httpbin.bar.global will resolve to
     # must be unique for each remote service, within a given cluster.
     # This address need not be routable. Traffic for this IP will be captured
     # by the sidecar and routed appropriately.
     - 240.0.0.2 # host对应的虚拟地址，必须包含，否则istio-coredns-plugin无法进行DNS解析
     endpoints: #指定非本
     # This is the routable address of the ingress gateway in cluster2 that
     # sits in front of sleep.foo service. Traffic from the sidecar will be
     # routed to this address.
     - address: ${INGRESS_HOST} # 将其替换为对应的node地址值即可
    ports:
         http1: 30615 # 替换为对应的nodeport
   EOF
   ```

   > 由于使用了nodeport方式，因此需要使用容器15443端口对应的nodeport端口，使用如下方式获取：
   >
   > ```shell
   > # kubectl --context=$CTX_CLUSTER2 get svc -n istio-system istio-ingressgateway -o=jsonpath='{.spec.ports[?(@.port==15443)].nodePort}'
   > 30615
   > ```

   上述配置会将`cluster1`中`httpbin.bar.global`服务的所有端口上的流量(通过mutual TLS)路由到`$INGRESS_HOST:15443`。

   网关的15443端口是一个感知SNI的Envoy配置，在安装Istio控制面时部署。到达15443端口的流量会在目标集群的内部服务的pod上进行负载均衡(即`cluster2`的`httpbin.bar`)。

   >下面是从cluster1的sleep中导出的istio-proxy配置，可以看到`httpbin.bar.global`的后端为172.18.0.5:30615,即`$INGRESS_HOST:$NODE_PORT`
   >
   >```json
   >     "cluster": {
   >      "load_assignment": {
   >       "cluster_name": "outbound|8000||httpbin.bar.global",
   >       "endpoints": [
   >        {
   >         "locality": {},
   >         "lb_endpoints": [
   >          {
   >           "endpoint": {
   >            "address": {
   >             "socket_address": {
   >              "address": "172.18.0.5",
   >              "port_value": 30615
   >             }
   >            }
   >           },
   >           "load_balancing_weight": 1
   >          }
   >         ],
   >         "load_balancing_weight": 1
   >        }
   >       ]
   >      },
   >	  ...
   >    },
   >```
   >
   >对应的路由如下，可以看到240.0.0.2只是作为了SNI匹配的一种，将匹配到的请求转发给上面的`"cluster": "outbound|8000||httpbin.bar.global"`进行处理：
   >
   >```json
   >     "route_config": {
   >      "@type": "type.googleapis.com/envoy.config.route.v3.RouteConfiguration",
   >      "name": "8000",
   >      "virtual_hosts": [
   >       ...
   >       {
   >        "name": "httpbin.bar.global:8000",
   >        "domains": [
   >         "httpbin.bar.global",
   >         "httpbin.bar.global:8000",
   >         "240.0.0.2",
   >         "240.0.0.2:8000"
   >        ],
   >        "routes": [
   >         {
   >          "match": {
   >           "prefix": "/"
   >          },
   >          "route": {
   >           "cluster": "outbound|8000||httpbin.bar.global",
   >           ...
   >     },
   >```
   >
   >另外需要注意的是cluster1和cluster2都使用了一个Gateway和DestinationRule，对到`*.global`域的请求使用mTLS进行加密，并在网关上使用`AUTO_PASSTHROUGH`模式，此模式会根据SNI将请求直接转发给后端应用，无需virtualservice进行绑定。
   >
   >```yaml
   >apiVersion: networking.istio.io/v1beta1
   >kind: Gateway
   >spec:
   >  selector:
   >    istio: ingressgateway
   >  servers:
   >  - hosts:
   >    - '*.global'
   >    port:
   >      name: tls
   >      number: 15443
   >      protocol: TLS
   >    tls:
   >      mode: AUTO_PASSTHROUGH
   >
   >apiVersion: networking.istio.io/v1beta1
   >kind: DestinationRule
   >spec:
   >  host: '*.global'
   >  trafficPolicy:
   >    tls:
   >      mode: ISTIO_MUTUAL
   >```

5. 校验可以通过sleep服务访问httpbin服务。

   ```shell
   $ kubectl exec --context=$CTX_CLUSTER1 $SLEEP_POD -n foo -c sleep -- curl -I httpbin.bar.global:8000/headers
   ```
   
   > 在官方文档中使用如上命令即可在cluster1的sleep Pod中访问cluster2的httpbin服务。但从上面分析可以看到，当SNI为`httpbin.bar.global`的请求到达cluster2的ingress pod上时，它会按照k8s的coredns配置将该请求转发到istio的coredns进行解析，但cluster2并没有配置`httpbin.bar.global`对应的serviceentry，因此，istio的coredns也无法解析该dns，因此会返回503错误。在cluster2的istio-coredns-plugin容器的日志中可以找到如下信息：
   >
   > ```verilog
   > ... info    Query A record: httpbin.bar.global.->{httpbin.bar.global. 1 1}
   > ... info    Could not find the service requested
   > ... info    DNS query  ;; opcode: QUERY, status: NOERROR, id: 64168
   > ```
   >
   > 在cluster2中创建如下serviceentry
   >
   > ```yaml
   > $ kubectl apply --context=$CTX_CLUSTER2 -n bar -f - <<EOF
   > apiVersion: networking.istio.io/v1alpha3
   > kind: ServiceEntry
   > metadata:
   >   name: httpbin-bar
   > spec:
   >   hosts:
   >   - httpbin.bar.global
   >   location: MESH_INTERNAL
   >   ports:
   >   - name: http1
   >     number: 8000
   >     protocol: http
   >   resolution: DNS
   >   addresses:
   >   - 240.0.0.3
   >   endpoints:
   >   - address: httpbin.bar.svc.cluster.local #httpbin的k8s service
   > EOF
   > ```
   >
   > 在cluster2中创建一个`sleep` pod，并在该pod中访问cluster2的`bar`命名空间下的`httpbin`服务，可以看到访问成功：
   >
   > ```shell
   > # curl -I httpbin.bar.global:8000/headers
   > HTTP/1.1 200 OK
   > server: envoy
   > date: Sat, 10 Oct 2020 12:40:23 GMT
   > content-type: application/json
   > content-length: 554
   > access-control-allow-origin: *
   > access-control-allow-credentials: true
   > x-envoy-upstream-service-time: 206
   > ```
   >
   > 此时从cluster1 仍然无法访问cluster2,跟踪issue https://github.com/istio/istio.io/issues/8285
   
   



##### 通过egress网关向远端发送流量











参考：

- [Using CoreDNS to Conceal Network Identities of Services in Istio](https://thecloudblog.net/post/using-coredns-to-conceal-network-identities-of-services-in-istio/)

























