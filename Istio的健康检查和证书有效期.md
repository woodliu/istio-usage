## Istio的健康检查和扩展证书有效期

[TOC]

### 健康检查

[kubernetes的liveness 和readiness 探针](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)描述了集中配置liveness和readness探针的方式，包括：

1. 命令
2. HTTP请求

不管是否启用了互TLS，命令方法都可以与Istio一起工作。

而在启用mutual TLS的请求下，如果使用了HTTP请求方式作为探针，则需要特殊的配置。这是因为对`liveness-http`服务的健康检查请求是由Kubelet发送的，而kubelet不包含Istio的证书。因此当启用mutual TLS时，健康检查的请求会失败。

Istio通过重写应用PodSpec中的readiness/liveness探针来解决该问题，这样探针会发生到[sidecar agent](https://istio.io/latest/docs/reference/commands/pilot-agent/)，然后将请求重定向到应用，剥离响应主体，仅返回响应代码。

该功能在Istio的[默认profie配置](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)中是被启用的。更多关于使用命令和HTTP请求方式进行健康检查的细节参见[官方文档](https://istio.io/latest/docs/ops/configuration/mesh/app-health-check/)。

### 扩展证书有效期

Istio的自签证书的默认有效期为1年。如果使用Istio自签证书，那么就需要注意根证书的过期时间。根证书的过期可能导致整个集群范围的意外中断。

使用下文的方式来估算根证书剩余的时间。

以下步骤展示了如何过渡到一个新的根证书。在过渡之后，新的根证书会有一个10年的生存时间。注意Envoy实例将会被热重启来重新加载新的根证书，这可能会影响长连接。更多关于Envoy热重启的细节，请参见[这里](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/hot_restart)和[这里](https://blog.envoyproxy.io/envoy-hot-restart-1d16b14555b5)。

#### 场景

如果当前没有在Istio中使用mutual TLS特性，且后续也不会使用，则不需要执行任何动作。

如果未来可能会使用mutual TLS，则应该按照如下步骤来执行根证书的过渡。

如果当前在mutual TLS中使用了自签证书作，请根据如下步骤来检查是否有影响。

#### 根证书过渡步骤

1. 校验根证书的过期时间：

   下载该[脚本](https://raw.githubusercontent.com/istio/tools/release-1.7/bin/root-transition.sh)，然后使用`kubectl`访问集群，获取证书的剩余有效时间。从脚本中可以看到，其获取的根证书就是`istio-system`命名空间下面的`istio-ca-secret` secret中的证书剩余有效时间。

   ```shell
   # wget https://raw.githubusercontent.com/istio/tools/release-1.7/bin/root-transition.sh
   # chmod +x root-transition.sh
   # ./root-transition.sh check-root
   ...
   =====YOU HAVE 3636 DAYS BEFORE THE ROOT CERT EXPIRES!=====
   ```

   在根证书到期之前执行其余步骤，以避免系统中断。

2. 校验sidecar的版本和更新：

   某些早期版本的Istio sidecar可能不支持自动加载新的根证书。运行如下命令来检查Istio sidecar 的版本：

   ```shell
   # ./root-transition.sh check-version
   Checking namespace: default
   Istio proxy version: 1.7.0
   ...
   Checking namespace: istio-system
   Istio proxy version: 1.7.0
   ...
   ```

   如果sidecar的版本低于1.0.8和1.1.8，请升级Istio控制面和sidecar，使其版本不低于1.0.8和1.1.8。根据Istio[升级步骤](https://istio.io/latest/docs/setup/upgrade/)进行升级。

3. 执行根证书过渡

   在过渡期间，Envoy sidecar可能需要热重启来加载新的证书，这个过程可能会对流量造成一定影响。参考[Envoy热重启](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/hot_restart)并阅读[该博文](https://blog.envoyproxy.io/envoy-hot-restart-1d16b14555b5)来了解更多细节。

   > 如果Pilot不包含Envoy sidecar，可以考虑安装一个。当Pilot使用老的根证书对新的负载证书进行校验时会产生问题，导致Pilot和Envoy的中断。查看该[文章](https://istio.io/latest/docs/ops/configuration/security/root-transition/#how-can-i-check-if-pilot-has-a-sidecar)来检查这个条件。查看Istio[升级指南](https://istio.io/latest/docs/setup/upgrade/)来为Pilot默认安装一个sidecar。

   ```shell
   $ ./root-transition.sh root-transition
   ```

   > 上述命令会生成一个有效期为10年的新证书来替换来证书，通过这种方式来扩展证书的有效期，即删除旧的`istio-ca-secret`，然后用新证书来生成新的`istio-ca-secret`。最后重启istiod容器以及其他容器(如带Envoy sidecar的容器和ingress/egress容器)。然后可以使用`istioctl analyze`查看更换结果

   

**注：本章节的脚本仍然是老版本的，无法自动删除istiod pod，且删除之后，还需要重启所有带Envoy sidecar的pod，以及istio-system命名空间中的ingress/egress pod。**

