## Istio问题总结

[TOC]

### 定位方式

Istio Envoy sidecar暴露了一个15000管理端口，可以用于调试相关的问题，支持的接口(对应[Envoy管理接口](https://www.envoyproxy.io/docs/envoy/v1.7.1/operations/admin))如下：

```shell
# kubectl -n default exec -it -c istio-proxy sleep-856d589c9b-5tv8c -- curl localhost:15000/help
admin commands are:
  /: Admin home page
  /certs: print certs on machine
  /clusters: upstream cluster status
  /config_dump: dump current Envoy configs (experimental)
  /contention: dump current Envoy mutex contention stats (if enabled)
  /cpuprofiler: enable/disable the CPU profiler
  /drain_listeners: drain listeners
  /healthcheck/fail: cause the server to fail health checks
  /healthcheck/ok: cause the server to pass health checks
  /heapprofiler: enable/disable the heap profiler
  /help: print out list of admin commands
  /hot_restart_version: print the hot restart compatibility version
  /listeners: print listener info
  /logging: query/change logging levels
  /memory: print current allocation/heap usage
  /quitquitquit: exit the server
  /ready: print server state, return 200 if LIVE, otherwise return 503
  /reopen_logs: reopen access logs
  /reset_counters: reset all counters to zero
  /runtime: print runtime values
  /runtime_modify: modify runtime values
  /server_info: print server version/status information
  /stats: print server stats
  /stats/prometheus: print server stats in prometheus format
  /stats/recentlookups: Show recent stat-name lookups
  /stats/recentlookups/clear: clear list of stat-name lookups and counter
  /stats/recentlookups/disable: disable recording of reset stat-name lookup names
  /stats/recentlookups/enable: enable recording of reset stat-name lookup names
```

通常会用到`/logging`接口来修改Envoy sidecar的日志级别，输出更详细的Envoy日志，用于问题定位。支持的日志级别为：`trace|debug|info|warning|error|critical|off`，默认是`info`级别。使用如下命令可以将pod `sleep-856d589c9b-5tv8c`的日志级别修改为`trace`：

```shell
# kubectl -n default exec -it -c istio-proxy sleep-856d589c9b-5tv8c -- curl -X POST localhost:15000/logging?level=trace
```

另外两个是`/config_dump`和`/stats`接口。前者可以导出Envoy sidecar的所有配置，后者给出Envoy sidecar记录的连接信息

```shell
# kubectl -n default exec -it -c istio-proxy sleep-856d589c9b-5tv8c -- curl -X POST localhost:15000/config_dump
```

```shell
# kubectl -n default exec -it -c istio-proxy sleep-856d589c9b-5tv8c -- curl -X POST localhost:15000/stats
```



### 问题解决

1. 当Istio的Envoy sidecar出现如下错误时：

   ```shell
   "- - -" 0 UH "-" "-" 0 0 7 - "-" "-" "-" "-" "-" - - 180.101.49.69:443 10.83.0.198:45392 - -
   ```

   对`UH`的解释为`UH: No healthy upstream hosts in upstream cluster in addition to 503 response code`。从上面错误信息可以看到并没有`UPSTREAM_CLUSTER`字段的内容，如果是访问集群内部的服务，可以查看是否存在该服务；如果是访问集群外面的服务，可以查看是否在Istio安装时指定了`--set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY`参数，如果指定了该参数，需要通过`ServiceEntry`来将外部服务注册到Istio的注册中心。

   下面是对baidu.com的服务，一个通过`ServiceEntry`进行注册，可以看到`UPSTREAM_CLUSTER`字段的值为`outbound|443||*.baidu.com`，可以通过`istioctl pc cluster`命令查看到该cluster；另外一个采用passthrough方式访问，可以看到`UPSTREAM_CLUSTER`字段的值为`PassthroughCluster`。

   ```shell
   "- - -" 0 - "-" "-" 780 165931 275 - "-" "-" "-" "-" "180.101.49.69:443" outbound|443||*.baidu.com 10.83.0.198:33954 180.101.49.69:443 10.83.0.198:33952 map.baidu.com -
   ```

   ```shell
   "- - -" 0 - "-" "-" 780 165875 338 - "-" "-" "-" "-" "180.101.49.69:443" PassthroughCluster 10.83.0.198:36314 180.101.49.69:443 10.83.0.198:36312 - -
   ```

   







format": "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPO
RT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"



















