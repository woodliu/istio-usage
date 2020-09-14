EnvoyFilter的使用

[TOC]

### Istio注入的容器

Istio的数据面会在pod中注入两个容器：`istio-init`和`istio-proxy`。

#### Istio-init

`istio-init`会通过创建iptables规则来接管流量：

- 命令行参数 -p `15001`表示出向流量被iptable重定向到Envoy的`15001`端口

- 命令行参数 -z `15006`表示入向流量被iptable重定向到Envoy的`15006`端口

- 命令行参数 -u `1337`参数用于排除用户ID为`1337`，即Envoy自身的流量，以避免Iptable把Envoy发出的数据又重定向到Envoy，形成死循环。在istio-proxy容器中执行如下命令可以看到Envoy使用的用户id为`1337`。

  ```shell
  $ id
  uid=1337(istio-proxy) gid=1337(istio-proxy) groups=1337(istio-proxy)
  ```

```shell
  istio-iptables
  -p
  15001
  -z
  15006
  -u
  1337
  -m
  REDIRECT
  -i
  *
  -x

  -b
  *
  -d
  15090,15021,15020
  --run-validation
  --skip-rule-apply
```
#### istio-proxy

`istio-proxy`容器中会运行两个程序：`pilot-agent`和`envoy`。

```shell
$ ps -ef|cat
UID          PID    PPID  C STIME TTY          TIME CMD
istio-p+       1       0  0 Sep10 ?        00:03:39 /usr/local/bin/pilot-agent proxy sidecar --domain default.svc.cluster.local --serviceCluster sleep.default --proxyLogLevel=warning --proxyComponentLogLevel=misc:error --trust-domain=cluster.local --concurrency 2
istio-p+      27       1  0 Sep10 ?        00:14:30 /usr/local/bin/envoy -c etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster sleep.default --service-node sidecar~10.80.3.109~sleep-856d589c9b-x6szk.default~default.svc.cluster.local --local-address-ip-version v4 --log-format-prefix-with-location 0 --log-format %Y-%m-%dT%T.%fZ.%l.envoy %n.%v -l warning --component-log-level misc:error --concurrency 2
```

`pilot-agent`根据启动参数和K8S API Server中的配置信息生成Envoy的初始配置文件(`/etc/istio/proxy/envoy-rev0.json`)，并负责启动Envoy进程(可以看到`Envoy`进程的父进程是`pilot-agent`)；`envoy`会通过xDS接口从istiod动态获取配置文件。`envoy-rev0.json`初始配置文件结构如下：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200914110435327-1158938665.png)

- node：给出了Envoy








### 参考

- Istio流量管理实现机制深度解析