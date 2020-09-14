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

`pilot-agent`根据启动参数和K8S API Server中的配置信息生成Envoy的[bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#bootstrap)文件(`/etc/istio/proxy/envoy-rev0.json`)，并负责启动Envoy进程(可以看到`Envoy`进程的父进程是`pilot-agent`)；`envoy`会通过xDS接口从istiod动态获取配置文件。`envoy-rev0.json`初始配置文件结构如下：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200914110435327-1158938665.png)

- [node](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#config-core-v3-node)：给出了Envoy 实例的信息

  ```json
    "node": {
      "id": "sidecar~10.80.3.109~sleep-856d589c9b-x6szk.default~default.svc.cluster.local",
      "cluster": "sleep.default",
      "locality": {
      },
      "metadata": {"APP_CONTAINERS":"sleep,istio-proxy","CLUSTER_ID":"Kubernetes","EXCHANGE_KEYS":"NAME,NAMESPACE,INSTANCE_IPS,LABELS,OWNER,PLATFORM_METADATA,WORKLOAD_NAME,MESH_ID,SERVICE_ACCOUNT,CLUSTER_ID","INSTANCE_IPS":"10.80.3.109,fe80::40fb:daff:feed:e56c","INTERCEPTION_MODE":"REDIRECT","ISTIO_PROXY_SHA":"istio-proxy:f642a7fd07d0a99944a6e3529566e7985829839c","ISTIO_VERSION":"1.7.0","LABELS":{"app":"sleep","istio.io/rev":"default","pod-template-hash":"856d589c9b","security.istio.io/tlsMode":"istio","service.istio.io/canonical-name":"sleep","service.istio.io/canonical-revision":"latest"},"MESH_ID":"cluster.local","NAME":"sleep-856d589c9b-x6szk","NAMESPACE":"default","OWNER":"kubernetes://apis/apps/v1/namespaces/default/deployments/sleep","POD_PORTS":"[{\"name\":\"http-envoy-prom\",\"containerPort\":15090,\"protocol\":\"TCP\"}]","PROXY_CONFIG":{"binaryPath":"/usr/local/bin/envoy","concurrency":2,"configPath":"./etc/istio/proxy","controlPlaneAuthPolicy":"MUTUAL_TLS","discoveryAddress":"istiod.istio-system.svc:15012","drainDuration":"45s","envoyAccessLogService":{},"envoyMetricsService":{},"parentShutdownDuration":"60s","proxyAdminPort":15000,"proxyMetadata":{"DNS_AGENT":""},"serviceCluster":"sleep.default","statNameLength":189,"statusPort":15020,"terminationDrainDuration":"5s","tracing":{"zipkin":{"address":"zipkin.istio-system:9411"}}},"SDS":"true","SERVICE_ACCOUNT":"sleep","WORKLOAD_NAME":"sleep","k8s.v1.cni.cncf.io/networks":"istio-cni","sidecar.istio.io/interceptionMode":"REDIRECT","sidecar.istio.io/status":"{\"version\":\"8e6e902b765af607513b28d284940ee1421e9a0d07698741693b2663c7161c11\",\"initContainers\":[\"istio-validation\"],\"containers\":[\"istio-proxy\"],\"volumes\":[\"istio-envoy\",\"istio-data\",\"istio-podinfo\",\"istiod-ca-cert\"],\"imagePullSecrets\":null}","traffic.sidecar.istio.io/excludeInboundPorts":"15020","traffic.sidecar.istio.io/includeOutboundIPRanges":"*"}
    },
  ```

- [admin](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#config-bootstrap-v3-admin)：给出了Envoy的日志路径和管理端口，例如可以通过`curl -X POST localhost:15000/logging?level=trace`设置日志级别为`trace`。

  ```json
    "admin": {
      "access_log_path": "/dev/null", /* 管理服务器的访问日志路径 */
      "profile_path": "/var/lib/istio/data/envoy.prof", /* 管理服务器的CPU输出路径 */
      "address": { /* 管理服务器监听的TCP地址 */
        "socket_address": {
          "address": "127.0.0.1",
          "port_value": 15000
        }
      }
    },
  ```

- [dynamic_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#config-bootstrap-v3-bootstrap-dynamicresources)：配置动态资源，用于配置[lds_config](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/config_source.proto#config-core-v3-configsource)，[cds_config](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/config_source.proto#config-core-v3-configsource)和[ads_config](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/config_source.proto#config-core-v3-apiconfigsource)

  > Envoy通过`xds-grpc `cluster(参见`static_resources`)来获得xDS服务的地址。

  ```json
    "dynamic_resources": {
      "lds_config": { /* 通过一个LDS配置Listeners */
        "ads": {},
        "resource_api_version": "V3" /* LDS使用的API版本号 */
      },
      "cds_config": { /* 通过一个CDS配置Cluster */
        "ads": {},
        "resource_api_version": "V3"
      },
      "ads_config": { /* API配置资源，用于指定API类型和Envoy获取xDS API的cluster */
        "api_type": "GRPC", /* 使用GRPC获取xDS信息 */
        "transport_api_version": "V3", /* xDS传输协议的版本号 */
        "grpc_services": [
          {
            "envoy_grpc": {
              "cluster_name": "xds-grpc" /* 动态获取xDS配置的cluster */
            }
          }
        ]
      }
    },
  ```

- static_resources：配置静态资源，主要包括`clusters`和`listeners`两种资源：

  ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200914154659614-940913736.png)

  - clusters：

    ```json
        "clusters": [
          {
            "name": "prometheus_stats", /* 使用Prometheus暴露metrics，接口为127.0.0.1:15000/stats/prometheus */
            "type": "STATIC", /* 明确指定上游host的网络名(IP地址/端口等) */
            "connect_timeout": "0.250s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": { /* 仅用于类型为STATIC, STRICT_DNS或LOGICAL_DNS */
              "cluster_name": "prometheus_stats", 
              "endpoints": [{
                "lb_endpoints": [{ /* 负载均衡的后端 */
                  "endpoint": {
                    "address":{
                      "socket_address": {
                        "protocol": "TCP",
                        "address": "127.0.0.1",
                        "port_value": 15000
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "agent", /* 暴露健康检查接口，可以使用curl http://127.0.0.1:15020/healthz/ready -v查看 */
            "type": "STATIC",
            "connect_timeout": "0.250s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "prometheus_stats",
              "endpoints": [{
                "lb_endpoints": [{ /* 负载均衡的后端 */
                  "endpoint": {
                    "address":{
                      "socket_address": {
                        "protocol": "TCP",
                        "address": "127.0.0.1",
                        "port_value": 15020
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "sds-grpc",
            "type": "STATIC",
            "http2_protocol_options": {},
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "sds-grpc",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "pipe": {
                        "path": "./etc/istio/proxy/SDS" /* UNIX socket路径*/
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "xds-grpc", /* 动态xDS使用的grpc服务器配置 */
            "type": "STRICT_DNS",
            "respect_dns_ttl": true,
            "dns_lookup_family": "V4_ONLY",
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "transport_socket": {
              "name": "envoy.transport_sockets.tls",
              "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext",
                "sni": "istiod.istio-system.svc",
                "common_tls_context": {
                  "alpn_protocols": [
                    "h2"
                  ],
                  "tls_certificate_sds_secret_configs": [ /* SDS配置 */
                    {
                      "name": "default",
                      "sds_config": {
                        "resource_api_version": "V3",
                        "initial_fetch_timeout": "0s",
                        "api_config_source": {
                          "api_type": "GRPC",
                          "transport_api_version": "V3",
                          "grpc_services": [
                            { /* SDS服务器对应上面配置的sds-grpc cluster */
                              "envoy_grpc": { "cluster_name": "sds-grpc" }
                            }
                          ]
                        }
                      }
                    }
                  ],
                  "validation_context": {
                    "trusted_ca": {
                      "filename": "./var/run/secrets/istio/root-cert.pem" /* 挂载当前命名空间下的config istio-ca-root-cert，其中的CA证书与istio-system命名空间下的istio-ca-secret中的CA证书相同 */
                    },
                    "match_subject_alt_names": [{"exact":"istiod.istio-system.svc"}]
                  }
                }
              }
            },
            "load_assignment": {
              "cluster_name": "xds-grpc",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "socket_address": {"address": "istiod.istio-system.svc", "port_value": 15012}
                    }
                  }
                }]
              }]
            },
            "circuit_breakers": { /* 断路器配置 */
              "thresholds": [
                {
                  "priority": "DEFAULT",
                  "max_connections": 100000,
                  "max_pending_requests": 100000,
                  "max_requests": 100000
                },
                {
                  "priority": "HIGH",
                  "max_connections": 100000,
                  "max_pending_requests": 100000,
                  "max_requests": 100000
                }
              ]
            },
            "upstream_connection_options": {
              "tcp_keepalive": {
                "keepalive_time": 300
              }
            },
            "max_requests_per_connection": 1,
            "http2_protocol_options": { }
          }
    ```

    可以使用`istioctl pc cluster`命令查看静态cluster资源，第一列对应上面的`Cluster.name`

    ```shell
    # istioctl pc cluster sleep-856d589c9b-x6szk.default |grep STATIC
    BlackHoleCluster                        -         -          -             STATIC
    agent                                   -         -          -             STATIC
    prometheus_stats                        -         -          -             STATIC
    sds-grpc                                -         -          -             STATIC
    sleep.default.svc.cluster.local         80        http       inbound       STATIC
    ```

    

  - listener：

    ```json
        "listeners":[
          {
            "address": {
              "socket_address": {
                "protocol": "TCP",
                "address": "0.0.0.0",
                "port_value": 15090
              }
            },
            "filter_chains": [
              {
                "filters": [
                  {
                    "name": "envoy.http_connection_manager",
                    "typed_config": {
                      "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                      "codec_type": "AUTO",
                      "stat_prefix": "stats",
                      "route_config": {
                        "virtual_hosts": [
                          {
                            "name": "backend",
                            "domains": [
                              "*"
                            ],
                            "routes": [
                              {
                                "match": {
                                  "prefix": "/stats/prometheus"
                                },
                                "route": {
                                  "cluster": "prometheus_stats"
                                }
                              }
                            ]
                          }
                        ]
                      },
                      "http_filters": [{
                        "name": "envoy.router",
                        "typed_config": {
                          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                        }
                      }]
                    }
                  }
                ]
              }
            ]
          },
          {
            "address": {
              "socket_address": {
                "protocol": "TCP",
                "address": "0.0.0.0",
                "port_value": 15021
              }
            },
            "filter_chains": [
              {
                "filters": [
                  {
                    "name": "envoy.http_connection_manager",
                    "typed_config": {
                      "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                      "codec_type": "AUTO",
                      "stat_prefix": "agent",
                      "route_config": {
                        "virtual_hosts": [
                          {
                            "name": "backend",
                            "domains": [
                              "*"
                            ],
                            "routes": [
                              {
                                "match": {
                                  "prefix": "/healthz/ready"
                                },
                                "route": {
                                  "cluster": "agent"
                                }
                              }
                            ]
                          }
                        ]
                      },
                      "http_filters": [{
                        "name": "envoy.router",
                        "typed_config": {
                          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                        }
                      }]
                    }
                  }
                ]
              }
            ]
          }
        ]
      }
    ```

    

  







### 参考

- Istio流量管理实现机制深度解析