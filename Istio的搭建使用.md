## openshift 4.3 Istio的搭建

本文档覆盖了官方文档的[Setup](https://istio.io/docs/setup/)的所有章节

[TOC]

### 安装Istio

本次安装的Istio版本为1.6.0，环境为openshift 4.3

注：不建议使用openshift 1.11(即kubernetes 3.11)安装istio，可能会出现如下兼容性问题，参见此[issue](https://github.com/jetstack/cert-manager/issues/2200)

```
must only have "properties", "required" or "description" at the root if the status subresource is enabled
```

#### openshift安装Istio

istio的安装涉及到两个文件：profile和manifest。前者用于控制组件的安装和组件的参数，profile配置文件所在的目录为`install/kubernetes/operator/profiles`；后者为安装所使用的yaml文件，如service，deployment等，会用到profile提供的参数，manifest配置文件所在的目录为`install/kubernetes/operator/charts`。因此可以通过两种方式安装istio，一种是通过profile进行安装，istio默认使用这种方式，如：

```shell
$ istioctl manifest apply --set profile=default
```

第二种是通过导出的manifest进行安装，如：

```shell
$ kubectl apply -f $HOME/generated-manifest.yaml
```

参考不同平台可以参考对应的[SetUp](https://istio.io/docs/setup/platform-setup/)。在openshift下面部署istio需要注意[版本](https://istio.io/docs/setup/platform-setup/openshift/)：

> OpenShift 4.1 and above use `nftables`, which is incompatible with the Istio `proxy-init` container. Make sure to use [CNI](https://istio.io/docs/setup/additional-setup/cni/) instead.

首先创建`istio-system`命名空间

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: istio-system
  labels:
    istio-injection: disabled
EOF
```

允许istio的serviceaccount使用UID为0的用户，使用的命名空间为`istio-system`

```shell
$ oc adm policy add-scc-to-group anyuid system:serviceaccounts:istio-system
```

istio默认会注入一个名为`istio-init`的`initContainer`，用于将pod的网络流量导向istio的sidecar proxy，该`initContainer`需要用到完整的`NET_ADMIN`和`NET_RAW` capabilities来配置网络，由此可能造成安全问题，使用istio CNI插件可以**替换**`istio-init`，且无需提升kubernetes RBAC权限，区别见下：

```yaml
# with istio-cni
          capabilities:
            drop:
            - ALL
          privileged: false
          readOnlyRootFilesystem: true
          runAsGroup: 1337
          runAsNonRoot: true
          runAsUser: 1337
          
# without istio-cni
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
            drop:
            - ALL
          privileged: false
          readOnlyRootFilesystem: false
          runAsGroup: 0
          runAsNonRoot: false
          runAsUser: 0
```

由于openshift 4.1以上版本不再使用iptables，转而使用nftables，因此需要[安装istio CNI](https://istio.io/docs/setup/additional-setup/cni/#helm-chart-parameters)插件，否则在sidecar注入时会出现如下`istio iptables-restore: unable to initialize table 'nat'`的错误，即无法执行`iptables-resotre`命令。

执行如下命令安装istio-cni并使用default类型的profile(见下)安装istio，具体参数含义参见[官方文档](https://istio.io/docs/setup/additional-setup/cni/#helm-chart-parameters)。

```yaml
cat <<'EOF' > cni-annotations.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    cni:
      enabled: true
      namespace: kube-system
  values:
    cni:
      excludeNamespaces:
       - istio-system
       - kube-system
      chained: false
      cniBinDir: /var/lib/cni/bin
      cniConfDir: /etc/cni/multus/net.d
      cniConfFileName: istio-cni.conf
    sidecarInjectorWebhook:
      injectedAnnotations:
        "k8s.v1.cni.cncf.io/networks": istio-cni
EOF
```

```shell
$ istioctl manifest apply -f cni-annotations.yaml
```

安装结果如下：

```shell
# oc get pod
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-64f6f9d5c6-lwcp8   1/1     Running   0          41m
istiod-5bb879d86c-6ttml                 1/1     Running   0          42m
prometheus-77b9c64b9c-r2pld             2/2     Running   0          41m
# oc get pod -nkube-system
NAME                   READY   STATUS    RESTARTS   AGE
istio-cni-node-2xzl8   2/2     Running   0          41m
istio-cni-node-4nb6k   2/2     Running   0          41m
istio-cni-node-7j5ck   2/2     Running   0          41m
istio-cni-node-f9bnf   2/2     Running   0          41m
istio-cni-node-lp7v6   2/2     Running   0          41m
```

为ingress gateway暴露router：

```shell
$ oc -n istio-system expose svc/istio-ingressgateway --port=http2
```

至此istio的基本组件已经安装完毕，可以使用如下方式导出本次安装的`profile`

```shell
$ istioctl profile dump -f cni-annotations.yaml > generated-profile.yaml
```

使用如下命令导出安装istio的`manefest`的内容

```shell
$ istioctl manifest generate -f cni-annotations.yaml  > generated-manifest.yaml
```

校验安装结果

```shell
$ istioctl verify-install -f generated-manifest.yaml
```

istio会使用UID为1337的用户将sidecar注入到应用中，openshift默认不允许使用该用户，执行如下命令进行授权。`target-namespace`为应用所在的命名空间。

```shell
$ oc adm policy add-scc-to-group privileged system:serviceaccounts:<target-namespace>
$ oc adm policy add-scc-to-group anyuid system:serviceaccounts:<target-namespace>
```

当清理应用pod后，需要删除添加的权限

```shell
$ oc adm policy remove-scc-from-group privileged system:serviceaccounts:<target-namespace>
$ oc adm policy remove-scc-from-group anyuid system:serviceaccounts:<target-namespace>
```

openshift下使用`multus`管理CNI，它需要在应用的命名空间中部署[`NetworkAttachmentDefinition`](https://istio.io/docs/setup/platform-setup/openshift/#additional-requirements-for-the-application-namespace)来使用istio-cni插件，使用如下命令创建`NetworkAttachmentDefinition`，`target-namespace`替换为实际应用所在的命名空间。

```shell
$ cat <<EOF | oc -n <target-namespace> create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: istio-cni
EOF
```

移除应用后，使用如下方式移除`NetworkAttachmentDefinition`

```shell
$ oc -n <target-namespace> delete network-attachment-definition istio-cni
```

##### 更新istio配置

例如可以使用如下方式卸载已经安装的第三方工具Prometheus，注意必须带上文件`cni-annotations.yaml`，否则会使用默认的profile重新配置istio，这样会导致删除istio-cni。如果组件的配置没有改变，则不会重新该组件的pod

```shell
$ istioctl manifest apply -f cni-annotations.yaml --set addonComponents.prometheus.enabled=false
```

##### openshif卸载istio

```shell
$ istioctl manifest generate -f cni-annotations.yaml | kubectl delete -f -
```

#### [标准安装istio](https://istio.io/docs/setup/install/istioctl/)

`istioctl` 使用内置的charts生成manifest，这些charts位于目录`install/kubernetes/operator/charts`。

```shell
# ll
total 36
drwxr-xr-x. 4 root root 4096 Apr 21 06:51 base
drwxr-xr-x. 4 root root 4096 Apr 21 06:51 gateways
drwxr-xr-x. 3 root root 4096 Apr 21 06:51 istio-cni
drwxr-xr-x. 5 root root 4096 Apr 21 06:51 istio-control
drwxr-xr-x. 3 root root 4096 Apr 21 06:51 istiocoredns
drwxr-xr-x. 3 root root 4096 Apr 21 06:51 istio-operator
drwxr-xr-x. 3 root root 4096 Apr 21 06:51 istio-policy
drwxr-xr-x. 8 root root 4096 Apr 21 06:51 istio-telemetry
drwxr-xr-x. 4 root root 4096 Apr 21 06:51 security
```

直接执行如下命令即可安装[官方](https://istio.io/docs/setup/install/istioctl/#install-istio-using-the-default-profile)默认配置的istio。

```shell
$ istioctl manifest apply --set profile=default
```

istio默认支持如下6种profile

```shell
# istioctl profile list
Istio configuration profiles:
    default
    demo
    empty
    minimal
    preview
    remote
```

安装的组件的区别如下：

|                        | default | demo | minimal | remote |
| ---------------------- | ------- | ---- | ------- | ------ |
| Core components        |         |      |         |        |
| `istio-egressgateway`  |         | X    |         |        |
| `istio-ingressgateway` | X       | X    |         |        |
| `istio-pilot`          | X       | X    | X       |        |
| Addons                 |         |      |         |        |
| `grafana`              |         | X    |         |        |
| `istio-tracing`        |         | X    |         |        |
| `kiali`                |         | X    |         |        |
| `prometheus`           | X       | X    |         | X      |

使用如下方式可以查看某个profile的配置信息，profile类型helm的values.yaml，用于给部署用的yaml提供配置参数。每个组件包含两部分内容：components下的组件以及组件的参数value

istio提倡使用[ `IstioOperator API` ](https://istio.io/docs/reference/config/istio.operator.v1alpha1/)进行定制化配置。

```shell
$ istioctl profile dump default
```

使用如下方式可以查看某个组件的配置：

```shell
$ istioctl profile dump --config-path components.pilot default
```

在安装前可以使用如下方式导出需要安装的所有yaml信息 ，包括CRD，deployment，service等

```shell
$ istioctl manifest generate --set profile=default --set hub=docker-local.com/openshift4 > $HOME/generated-manifest.yaml
```

在进行确认或修改之后可以使用`apply`命令执行安装

```shell
$ kubectl apply -f $HOME/generated-manifest.yaml
```

使用如下方式校验安装结果

```shell
$ istioctl verify-install -f $HOME/generated-manifest.yaml
```

自定义配置：

- 使用`--set`选项进行设置，如下面用于设置profile中的`global.controlPlaneSecurityEnabled`为true

  ```shell
  $ istioctl manifest apply --set values.global.controlPlaneSecurityEnabled=true
  ```

- 如果修改的参数比较多，可以使用yaml文件统一进行配置，实际使用[ `IstioOperator  API`](https://istio.io/docs/reference/config/istio.operator.v1alpha1/)进行profile的修改

  ```shell
  $ istioctl manifest apply -f samples/operator/pilot-k8s.yaml
  ```

istio的核心组件定义在 `IstioOperator` API的components下面：

| Components        |
| ----------------- |
| `base`            |
| `pilot`           |
| `proxy`           |
| `sidecarInjector` |
| `telemetry`       |
| `policy`          |
| `citadel`         |
| `nodeagent`       |
| `galley`          |
| `ingressGateways` |
| `egressGateways`  |
| `cni`             |

第三方插件可以在`IstioOperator API` 的addonComponents下指定，如：

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  addonComponents:
    grafana:
      enabled: true
```

每个组件都有一个[`KubernetesResourceSpec`](https://istio.io/docs/reference/config/istio.operator.v1alpha1/#KubernetesResourcesSpec),用于设置如下k8s属性

1. [Resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container)
2. [Readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
3. [Replica count](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
4. [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
5. [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#how-disruption-budgets-work)
6. [Pod annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
7. [Service annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
8. [ImagePullPolicy](https://kubernetes.io/docs/concepts/containers/images/)
9. [Priority class name](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#priorityclass)
10. [Node selector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector)
11. [Affinity and anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)
12. [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
13. [Toleration](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)
14. [Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
15. [Env](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)

##### 标准卸载istio

```shell
$ istioctl manifest generate <your original installation options> | kubectl delete -f -
```

### 更新Istio

#### 金丝雀升级

`revision`安装方式支持同时部署多个版本的istio，在升级时可以将流量逐步转移到新版本的istio上。每个修订`revision`都是一个完整的istio控制面，具有独立的`Deployment`, `Service`等。

##### 控制面升级

使用如下方式可以安装一个名为`canary`的istio修订版本。下面方式默认会使用`default` profile创建一套新的istio，如果新的istio和老的istio有组件重叠，则可能导致组件重建。从下面的`AGE`字段可以看出，新的`Prometheus`覆盖了老的`Prometheus`，`ingressgateway`也一样，因此最好的方式是指定文件，参见[istio install](https://istio.io/docs/reference/commands/istioctl/#istioctl-install)

```shell
$ istioctl install --set revision=canary
```

命令执行成功后，可以看到存在2个控制面，每个控制面有各自的`Deployment`，`Service`等。

```shell
$ oc get pod
NAME                                   READY   STATUS    RESTARTS   AGE
istio-ingressgateway-c9648ffbd-9pmwk   1/1     Running   0          113m
istiod-788cf6c878-cmn47                1/1     Running   0          139m
istiod-canary-79599d745b-ph7gj         1/1     Running   0          113m
prometheus-597596ffdd-v28hk            2/2     Running   0          113m

$ oc get deploy
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
istio-ingressgateway   1/1     1            1           138m
istiod                 1/1     1            1           139m
istiod-canary          1/1     1            1           113m
prometheus             1/1     1            1           138m
```

sidecar注入配置也有2套

```shell
$ oc get mutatingwebhookconfigurations
NAME                            CREATED AT
istio-sidecar-injector          2020-05-22T02:42:57Z
istio-sidecar-injector-canary   2020-05-22T03:08:28Z
```

##### 数据面升级

创建新的istio修订版本并不会影响现有的代理。为了升级代理，需要将代理的配置指向新的控制面。通过命名空间标签`istio.io/rev`设置控制sidecar注入的控制面。

下面操作升级了`test-ns`命名空间，移除标签`istio-injection`，并增加标签`istio.io/rev`，指向新的istio `canary`。注意必须移除标签`istio-injection`，否则istio会优先处理`istio-injection`(原因是为了向后兼容)

```shell
$ kubectl label namespace test-ns istio-injection- istio.io/rev=canary
```

在升级命名空间后，需要重启pod来触发sidecar的注入，如下使用滚动升级

```shell
$ kubectl rollout restart deployment -n test-ns
```

通过如下方式可以查看使用`canary`istio修订版的pod

```shell
$ kubectl get pods -n test-ns -l istio.io/rev=canary
```

为了验证test-ns命名空间中的新pod使用了istiod-canary服务，可以选择一个pod使用如下命令进行验证。从输出中可以看到其使用了`istiod-canary`控制面

```shell
$ istioctl proxy-config endpoints ${pod_name}.test-ns --cluster xds-grpc -ojson | grep hostname
"hostname": "istiod-canary.istio-system.svc"
```

可以通过如下方式导出`canary`修订版的配置信息

```shell
$ istioctl manifest generate --revision=canary >canary.yaml
```

#### 替换升级

> 升级过程中可能会造成流量中断，为了最小化影响，需要确保istio中的各个组件(除Citadel)至少有两个副本正在运行，此外需要通过[PodDistruptionBudgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)保证至少有一个可用的pod

- 首先下载最新的istio

- 校验当前环境中支持的升级到的版本，如下命令会给出推荐的版本

  ```shell
  $ istioctl manifest versions
  ```

- 确认需要升级的cluster是否正确

  ```shell
  $ kubectl config view
  ```

- 通过如下命令执行升级，`<your-custom-configuration-file>`为当前版本的 [IstioOperator API 配置](https://istio.io/docs/setup/install/istioctl/#configure-component-settings) 文件

  ```shell
  $ istioctl upgrade -f `<your-custom-configuration-file>`
  ```

  > istioctl upgrade不支持`--set`命令

- 在更新完毕之后，需要手动重启带有istio sidecar的pod来更新istio数据面

  ```shell
  $ kubectl rollout restart deployment
  ```

### sidecar注入

为了使用Istio的特性，pods必须运行在istio sidecar proxy的网格中。下面介绍两种注入istio sidecar的方式：手动注入和自动注入。

手动注入通过直接修改，如deployment的配置信息，将proxy配置注入到配置中；当应用所在的命名空间启用自动注入时，会在pod创建时通过[mutating webhook admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) 注入proxy配置。

sidecar的(手动或自动)注入会用到`istio-sidecar-injector` configmap。

- 手动注入

  当前版本手动注入时有一个问题，就是使用istio CNI之后无法将annotation `k8s.v1.cni.cncf.io/networks`注入(或导出)到配置文件中，导致出现如下问题，参见该[issue](https://github.com/istio/istio/issues/23651)：

  ```
  in new validator: 10.80.2.222
  Listening on 127.0.0.1:15001
  Listening on 127.0.0.1:15006
  Error connecting to 127.0.0.6:15002: dial tcp 127.0.0.1:0->127.0.0.6:15002: connect: connection refused
  Error connecting to 127.0.0.6:15002: dial tcp 127.0.0.1:0->127.0.0.6:15002: connect: connection refused
  Error connecting to 127.0.0.6:15002: dial tcp 127.0.0.1:0->127.0.0.6:15002: connect: connection refused
  Error connecting to 127.0.0.6:15002: dial tcp 127.0.0.1:0->127.0.0.6:15002: connect: connection refused
  Error connecting to 127.0.0.6:15002: dial tcp 127.0.0.1:0->127.0.0.6:15002: connect: connection refused
  ```

  - 手动注入需要满足一个条件，即在应用所在的命名空间中创建[NetworkAttachmentDefinition](https://istio.io/docs/setup/platform-setup/openshift/#additional-requirements-for-the-application-namespace)

  - 一种是直接使用`istio-sidecar-injector`的默认配置直接注入sidecar

    ```shell
    $ istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f -
    ```

    可以使用如下方式直接先导出注入sidecar的deployment，然后使用kubectl直接部署

    ```shell
    $ istioctl kube-inject -f samples/sleep/sleep.yaml -o sleep-injected.yaml --injectConfigMapName istio-sidecar-injector
    $ kubectl apply -f deployment-injected.yaml
    ```

  - 另一种是先导出`istio-sidecar-injector`的默认配置，可以修改后再手动注入sidecar

    导出配置：

    ```shell
    $ kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}' > inject-config.yaml
    $ kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.values}' > inject-values.yaml
    $ kubectl -n istio-system get configmap istio -o=jsonpath='{.data.mesh}' > mesh-config.yaml
    ```

    手动注入：

    ```shell
    $ istioctl kube-inject \
        --injectConfigFile inject-config.yaml \
        --meshConfigFile mesh-config.yaml \
        --valuesFile inject-values.yaml \
        --filename samples/sleep/sleep.yaml \
        | kubectl apply -f -
    ```

- 自动注入

  自动注入时需要满足两个条件：
  - 给应用所在的命名空间打上标签`istio-injection=enabled`

    ```shell
    $ kubectl label namespace <app-namespace> istio-injection=enabled
    ```

  - 与手动注入相同，需要在应用所在的命名空间中创建[NetworkAttachmentDefinition](https://istio.io/docs/setup/platform-setup/openshift/#additional-requirements-for-the-application-namespace)，然后在该命名空间下面正常创建(或删除并重建)pod即可自动注入sidecar。需要注意的是自动注入发生在pod层，并不体现在deployment上面，可以使用`describe pod`查看注入的sidecar。自动注入通过mutatingwebhookconfiguration定义了注入sidecar的规则，当前是`istio-injection=enabled`

    ```yaml
  namespaceSelector:
        matchLabels:
          istio-injection: enabled
    ```
    
    可以通过如下命令修改注入的规则，修改后需要重启已经注入sidecar的pod，使规则生效

    ```shell
$ kubectl edit mutatingwebhookconfiguration istio-sidecar-injector
    ```
    
    自动注入下，sidecar inject的webhook默认是打开的，如果要禁用该webhook，可以在上面cni-annotations.yaml文件中将`sidecarInjectorWebhook.enabled`置为`false`，这样就不会自动注入。
    

#### sidecar的注入控制

有两种方式可以控制sidecar的注入：

- 第一种是修改pod template spec的`sidecar.istio.io/inject` annotation：当该annotation为`true`时会进行自动注入sidecar，为`false`则不会注入sidecar。默认为`true`。这种情况需要修改应用的deployment。

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: ignored
  spec:
    template:
      metadata:
        annotations:
          sidecar.istio.io/inject: "false"
      spec:
        containers:
        - name: ignored
          image: tutum/curl
          command: ["/bin/sleep","infinity"]
  ```

- 第二种是修改`istio-sidecar-injector` configmap，通过在`neverInjectSelector`数组中罗列出标签，并禁止对匹配这些标签的pod注入sidecar(各个匹配项的关系为`OR`)。如下内容中不会对具有标签`openshift.io/build.name`或`openshift.io/deployer-pod-for.name`的pod注入sidecar。这种方式不需要修改应用的deployment。类似地，可以使用`alwaysInjectSelector`对某些具有特殊标签的pod注入sidecar

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: istio-sidecar-injector
  data:
    config: |-
      policy: enabled
      neverInjectSelector:
        - matchExpressions:
          - {key: openshift.io/build.name, operator: Exists}
        - matchExpressions:
          - {key: openshift.io/deployer-pod-for.name, operator: Exists}
      template: |-
        initContainers:
  ...
  ```

更多istio CNI与sidecar注入和流量重定向相关的参数参见[官方文档](https://istio.io/docs/setup/additional-setup/cni/#traffic-redirection-parameters)

#### 卸载自动注入

使用如下方式可以卸载自动注入功能：

```shell
$ kubectl delete mutatingwebhookconfiguration istio-sidecar-injector
$ kubectl -n istio-system delete service istio-sidecar-injector
$ kubectl -n istio-system delete deployment istio-sidecar-injector
$ kubectl -n istio-system delete serviceaccount istio-sidecar-injector-service-account
$ kubectl delete clusterrole istio-sidecar-injector-istio-system
$ kubectl delete clusterrolebinding istio-sidecar-injector-admin-role-binding-istio-system
```

注意上面命令并不会卸载已经注入到pod的sidecar，需要通过滚动更新或直接删除pod来生效

或者使用如下方式对某个命名空间禁用自动注入(推荐)：

```shell
$ kubectl label namespace <target-namespace> istio-injection-
```

### Istio CNI的兼容

#### 与init容器的兼容

当使用istio CNI的时候，kubelet会按照如下步骤启动注入sidecar的pod：

- 使用istio CNI插件进行配置，将流量导入到pod中的istio sidecar proxy容器
- 执行所有init容器，并成功运行结束
- 启动pod中的istio sidecar proxy和其他容器

由于init容器会在sidecar proxy容器之前运行，因此可能导致应用本身的init容器的通信中断。为了避免发生这种情况，可以通过如下配置避免重定向应用的init容器的流量：

- 设置`traffic.sidecar.istio.io/excludeOutboundIPRanges` annotation来禁用将流量重定向到与init容器通信的任何cidr。
- 设置`traffic.sidecar.istio.io/excludeOutboundPorts` annotation来禁止将流量重定向到init容器使用的出站端口

#### 与其他CNI插件的兼容

istio CNI插件作为CNI插件链中的一环，当创建或删除一个pod时，会按照顺序启动插件链上的每个插件，istio CNI插件仅仅(通过pod的网络命名空间中的iptables)将应用的pod流量重定向到注入的istio proxy sidecar容器。

*istio CNI插件不会干涉配置pod网络的基本CNI插件。*

更多细节参见[CNI specification reference](https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration-lists)

### TIPs:

- 不同平台下使用istio CNI执行initContainer时可能会出现istio-validation无法启动的错误，这种情况下默认会导致kubelet删除并重建pod，为了定位问题，可以将`repair.deletePods`配置为false，这样就不会立即删除pod

  ```yaml
  apiVersion: install.istio.io/v1alpha1
  kind: IstioOperator
  spec:
    components:
      cni:
        ...
    values:
      cni:
        repair:
          enabled: true
          deletePods: false
          ...
  ```
