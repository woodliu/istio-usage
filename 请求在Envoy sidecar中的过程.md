## Envoy 代理中的请求的生命周期

翻译自Envoy[官方文档](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)。

[TOC]

下面描述一个经过Envoy代理的请求的生命周期。首先会描述Envoy如何使用请求路径来处理请求，然后描述请求从下游到达Envoy代理之后发生的内部事件。我们将跟踪该请求，直到请求被分发到上游并在响应路径中进行处理。

### 术语

Envoy会在代码和文档中使用如下术语：

- Cluster：逻辑上的服务，包含一系列的endpoints，Envoy会将请求转发到这些Cluster上
- Downstream：连接到Envoy的实体。可能是一个本地应用(sidecar模型)或网络节点。非sidecar模型下体现为一个远端客户端。
- Endpoints：实现了逻辑服务的网络节点。Endpoints组成了Clusters。一个Cluster中的Endpoints为某个Envoy代理的upstream。
- Filter：连接或请求处理流水线的一个模块，提供了请求处理的某些功能。类似Unix的小型实用程序（过滤器）和Unix管道（过滤器链）的组合。
- Filter chain：一些列的Filters。
- Listeners：负责绑定一个IP/端口的Envoy模块，接收新的TCP连接(或UDP数据包)以及对下游的请求进行编排。
- Upstream：Envoy转发请求到一个服务时连接的Endpoint。可能是一个本地应用或网络节点。非sidecar模型下体现为一个远端后端。

### 网络拓扑

一个请求是如何通过一个网络组件取决于该网络的模型。Envoy可能会使用大量网络拓扑。下面会重点介绍Envoy的内部操作，但在本节中将简要介绍Envoy与网络其余部分的关系。

Envoy起源于服务网格Sidecar代理，用于剥离应用程序的负载平衡，路由，可观察性，安全性和发现服务。在服务网格模型中，请求会经过作为网关的Envoy。请求会通过ingress或egress监听器到达一个Envoy。

- ingress 监听器会从其他节点接收请求，并转发到本地应用。然后本地应用的响应会经过Envoy发回下游。
- Egress 监听器会从本地应用接收请求，并转发到网络的其他节点。这些接收请求的节点通常也会运行Envoy,并接收经过它们的ingress 监听器的请求。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213011467-1050558629.png)

Envoy会用到除服务网格使用到的各种配置，例如，它可以作为一个内部的负载均衡器：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213205552-975050703.png)

或作为一个边缘网络的ingress/egress代理：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213253584-1924298712.png)

实际中，通常会在服务网格中混合使用Envoy的这些特性，在网格边缘作为ingress/egress代理，以及在内部作为负载均衡器。一个请求路径可能会经过多个Envoys。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213520222-1306838362.png)

Envoy可以配置为多层拓扑来实现可伸缩性和可靠性，一个请求会首先经过一个边缘Envoy，然后传递给第二个Envoy层。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213654205-659093979.png)

以上所有场景中，请求通过下游的TCP，UDP或Unix域套接字到达一个指定的Envoy，然后由该Envoy通过TCP，UDP或UNIX域套接字转发到上游。下面仅关注单个Envoy代理。

### 配置

Envoy是一个非常可扩展的平台。通过如下因素可以组合成丰富的请求路径：

- L3/4协议，即TCP，UDP，UNIX域套接字
- L7协议，即 HTTP/1, HTTP/2, HTTP/3, gRPC, Thrift, Dubbo, Kafka, Redis 以及各种数据库
- 传输socket，即明文，TLS，ALTS
- 连接路由，即PROXY协议，源地址，动态转发
- 断路器以及异常值检测配置和激活状态
- 很多与网络相关的配置，HTTP, listener, 访问日志,健康检查, 跟踪和统计信息扩展

本例将涵盖如下内容：

- 在TCP连接上使用HTTP/2(带TLS)的上下游
- [HTTP连接管理器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_connection_management#arch-overview-http-conn-man)是唯一的[网络过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/network_filters#arch-overview-network-filters)。
- 假定的CustomFilter，以及 `router <arch_overview_http_routing>` ([HTTP 过滤器链](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters#arch-overview-http-filters))
- [文件系统访问日志](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/access_logging#arch-overview-access-logs-sinks)
- [Statsd sink](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/metrics/v3/stats.proto#envoy-v3-api-msg-config-metrics-v3-statssink)
- 使用静态endpoints的单[cluster](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/cluster_manager#arch-overview-cluster-manager)

假设使用如下静态的bootstrap配置文件：

```yaml
static_resources:
  listeners:
  # There is a single listener bound to port 443.
  - name: listener_https
    address:
      socket_address:
        protocol: TCP
        address: 0.0.0.0
        port_value: 443
    # A single listener filter exists for TLS inspector.
    listener_filters:
    - name: "envoy.filters.listener.tls_inspector"
      typed_config: {}
    # On the listener, there is a single filter chain that matches SNI for acme.com.
    filter_chains:
    - filter_chain_match:
        # This will match the SNI extracted by the TLS Inspector filter.
        server_names: ["acme.com"]
      # Downstream TLS configuration.
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_certificates:
            - certificate_chain: { filename: "certs/servercert.pem" }
              private_key: { filename: "certs/serverkey.pem" }
      filters:
      # The HTTP connection manager is the only network filter.
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          use_remote_address: true
          http2_protocol_options:
            max_concurrent_streams: 100
          # File system based access logging.
          access_log:
            - name: envoy.access_loggers.file
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
                path: "/var/log/envoy/access.log"
          # The route table, mapping /foo to some_service.
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["acme.com"]
              routes:
              - match:
                  path: "/foo"
                route:
                  cluster: some_service
      # CustomFilter and the HTTP router filter are the HTTP filter chain.
      http_filters:
          - name: some.customer.filter
          - name: envoy.filters.http.router
  clusters:
  - name: some_service
    connect_timeout: 5s
    # Upstream TLS configuration.
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
    load_assignment:
      cluster_name: some_service
      # Static endpoint assignment.
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 10.1.2.10
                port_value: 10002
        - endpoint:
            address:
              socket_address:
                address: 10.1.2.11
                port_value: 10002
    http2_protocol_options:
      max_concurrent_streams: 100
  - name: some_statsd_sink
    connect_timeout: 5s
    # The rest of the configuration for statsd sink cluster.
# statsd sink.
stats_sinks:
   - name: envoy.stat_sinks.statsd
     typed_config:
       "@type": type.googleapis.com/envoy.config.metrics.v3.StatsdSink
       tcp_cluster_name: some_statsd_cluster
```

### 高层架构

Envoy中的请求处理主要包含两大部分：

- [Listener子系统](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners)：处理下游请求，同时负责管理下游请求的生命周期以及到客户端的响应路径。下游HTTP/2的编解码器也位于此处。
- [Cluster子系统](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/cluster_manager#arch-overview-cluster-manager)：负责选择和配置到上游endpoint的连接，以及Cluster和endpoint的健康检查，负载均衡和连接池。上游HTTP/2的编解码器也位于此处。

这两个子系统与HTTP路由器过滤器桥接在一起，用于将HTTP请求从下游转发到上游。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911225519945-621108340.png)

我们使用术语[listener subsystem](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners) 和[cluster subsystem](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/cluster_manager#arch-overview-cluster-manager) 指代模块组以及由高层*ListenerManager* 和*ClusterManager*类创建的实例类。在下面讨论的许多组件都是由这些管理系统在请求之前和请求过程中实例化的，如监听器, 过滤器链, 编解码器, 连接池和负载均衡数据结构。

Envoy有一个[基于事件的线程模型](https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310)。主线程负责服务的生命周期、配置处理、统计等。一些[工作线程](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/threading_model#arch-overview-threading)用于处理请求。所有线程都围绕一个事件循环([libevent](https://libevent.org/))进行操作，任何给定的下游TCP连接(包括其中的所有多路复用流)在其生命周期内都将由一个工作线程进行处理。每个工作线程维护各自到上游endpoints的TCP连接池。UDP处理中会使用`SO_REUSEPORT`，通过内核一致性哈希将源/目标IP:端口元组散列到同一个工作线程。UDP过滤器状态会共享给特定的工作线程，过滤器负责根据需要提供会话语义。这与下面讨论的面向连接的TCP过滤器形成了对比，后者的过滤器状态以每个连接为基础，在HTTP过滤器的情况下，则以每个请求为基础。

工作线程很少会共享状态，并且很少会并行运行。 该线程模型可以扩展到core数量非常多的CPU。

### 请求流

#### 总览

使用上面的示例配置简要概述请求和响应的生命周期：

1. 由运行在一个`工作线程`的Envoy [监听器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners)接收下游TCP连接
2. 创建并运行 [监听过滤器(https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listener_filters#arch-overview-listener-filters)链。该链可以提供SNI以及其他TLS之前的信息。一旦完成，该监听器会匹配到一个网络过滤器链。每个监听器可能具有多个过滤链，这些filter链会匹配目标IP CIDR范围，SNI，ALPN，源端口等的某种组合。传输套接字（此例为TLS传输套接字）与此过滤器链相关联。
3. 在网络读取时，TLS传输套接字会从TCP连接中解密数据，以便后续做进一步的处理
4. 创建并运行[网络过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/network_filters#arch-overview-network-filters)链。用于HTTP的最重要的filter为HTTP连接管理器，作为network filter链上的最后一个filter
5. [HTTP连接管理器中](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_connection_management#arch-overview-http-conn-man)的HTTP/2编解码器将解密后的数据流从TLS连接上解帧并解复用为多个独立的流。每个流处理一个单独的请求和响应。
6. 对于每个HTTP流，会创建并运行一个[HTTP 过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters#arch-overview-http-filters)链。请求首先会经过CustomFilter，可能会读取并修改该请求。最重要的HTTP过滤器是路由过滤器，位于HTTP 过滤器链的末尾。当路由过滤器调用`decodeHeaders`时，会选择路由和cluster。数据流中的请求首部会转发到上游cluster的endpoint中。`router` 过滤器会从群集管理器中为匹配的cluster获取HTTP[连接池](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/connection_pooling#arch-overview-conn-pool)。
7. Cluster会指定[负载均衡](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/overview#arch-overview-load-balancing)来查找endpoint。cluster的断路器会检查是否允许一个新的流。如果endpoint的连接池为空或容量不足，则会创建一个到该endpoint的新连接。
8. 上游endpoint连接的HTTP/2编解码器会对请求的流(以及通过单个TCP连接到该上游的其他流)进行多路复用和帧化。
9. 上游endpoint连接的TLS传输socket会加密这些字节并写入到上游连接的TCP socket中。
10. 请求包含首部，可选的消息体和尾部，通过代理到达上游，并通过代理对下游进行响应。响应会以与请求[相反的顺序](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters#arch-overview-http-filters-ordering)通过HTTP过滤器，在发送到下游前从路由过滤器开始，然后经过CustomFilter。
11. 当完成响应之后，会销毁流。请求后的处理将更新统计信息，写入访问日志并最终确定跟踪范围。



我们将在以下各节中详细介绍每个步骤。

#### 1.Listener TCP连接的接收

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200912151231417-1211519644.png)

*ListenerManager*负责获取监听](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners)的配置，并实例化绑定到各自IP/端口的多个`Listener`实例。监听器的状态可能为：

- *Warming*：监听器等待配置依赖(即，路配置，动态secret)。此时监听器无法接收TCP连接。
- *Active*：监听器绑定到其IP/端口，可以接收TCP连接。
- *Draining*：监听器不再接收新的TCP连接，现有的TCP连接可以在一段时间内继续使用。

每个[工作线程](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/threading_model#arch-overview-threading)会为每个配置的监听器维护各自的监听器实例。每个监听器可能通过**SO_REUSEPORT** 绑定到相同的端口，或共享绑定到该端口的socket。当接收到一个新的TCP连接，内核会决定哪个工作线程来接收该连接，然后该工作线程对应的监听器会调用`Server::ConnectionHandlerImpl::ActiveTcpListener::onAccept()`。

#### 2.监听过滤链和网络过滤器链的匹配

工作线程的监听器然后会创建并运行[监听过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listener_filters#arch-overview-listener-filters)链。过滤器链是通过每个过滤器的过滤器工厂创建的，该工厂会感知过滤器的配置，并为每个连接或流创建新的过滤器实例。

在TLS 过滤器配置下，监听过滤器链会包含[TLS检查](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/listener_filters/tls_inspector#config-listener-filters-tls-inspector)过滤器(`envoy.filters.listener.tls_inspector`)。该过滤器会检查初始的TLS握手，并抽取server name(SNI)，然后使用SNI进行过滤器链的匹配。尽管TLS的检查明确出现在监听过滤器链配置中，但Envoy还能够在监听过滤器链中需要SNI（或ALPN）时自动将其插入。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913152453326-1937005929.png)

TLS检查filter实现了 [ListenerFilter](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/network/filter.h)接口。所有的filter接口，无论是监听或网络/HTTP，都需要过滤器实现特定连接或流时间的回调方法，*ListenerFilter*中为：

```c++
virtual FilterStatus onAccept(ListenerFilterCallbacks& cb) PURE;
```

`onAccept()`允许在TCP accept处理时运行一个过滤器。回调方法的`FilterStatus`控制监听过滤器链将如何继续。监听过滤器可能会暂停过滤器链，后续再恢复运行，如响应另一个服务进行的RPC请求。

在过滤器链进行匹配时，会抽取监听过滤器和连接的属性，提供用于处理连接的网络过滤器链和传输socket。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913154118823-389554441.png)

#### 3.TLS传输socket的解密

Envoy通过[TransportSocket](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/network/transport_socket.h)扩展接口提供了插件式的传输socket。传输socket遵循TCP连接的生命周期事件，使用网络buffer进行读写。传输socket需要实现如下关键方法：

```c++
virtual void onConnected() PURE;
virtual IoResult doRead(Buffer::Instance& buffer) PURE;
virtual IoResult doWrite(Buffer::Instance& buffer, bool end_stream) PURE;
virtual void closeSocket(Network::ConnectionEvent event) PURE;
```

当一个TCP连接可以传输数据时，`Network::ConnectionImpl::onReadReady()`会通过`SslSocket::doRead()`调用TLS传输socket。传输socket然后会在TCP连接上进行TLS握手。TLS握手结束后，`SslSocket::doRead()`会给`Network::FilterManagerImpl`实例提供一个解密的字节流，该实力负责管理网络过滤器链。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913154822343-2090496267.png)

需要注意的是，无论是TLS握手还是过滤器pipeline暂停，都不会真正阻塞任何操作。由于Envoy是基于事件的，因此任何需要额外数据才能进行处理的情况都将导致提前完成事件，并将CPU转移给另一个事件。如当网络提供了更多的可读数据时，该读事件将会触发TLS握手恢复。

#### 4.网络过滤器链的处理

与监听过滤器链相同，Envoy会通过*Network::FilterManagerImpl*，从对应的过滤器工厂实例化一些列网络过滤器。每个新连接的实例都是新的。与传输socket相同，网络过滤器也会遵循TCP的生命周期事件，并在来自传输socket中的数据可用时被唤醒。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913155624620-2072767193.png)

网络过滤器包含一个pipeline，与每个连接一个的传输socket不通，网络过滤器分为三种：

- [ReadFilter](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/network/filter.h)：实现了`onData()`,当连接中的数据可用时(由于某些请求)被调用
- [WriteFilter](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/network/filter.h)：实现了`onWrite()`, 当给连接写入数据时(由于某些响应)被调用
- [Filter](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/network/filter.h)：实现了*ReadFilter*和*WriteFilter*.

关键过滤器方法的方法签名为：

```c++
virtual FilterStatus onNewConnection() PURE;
virtual FilterStatus onData(Buffer::Instance& data, bool end_stream) PURE;
virtual FilterStatus onWrite(Buffer::Instance& data, bool end_stream) PURE;
```

与监听过滤器类似，`FilterStatus`允许过滤器暂停过滤器链的执行。例如，如果需要查询限速服务，限速网络过滤器将会从`onData()`中返回`Network::FilterStatus::StopIteration`，并在请求结束后调用`continueReading()`。

用于处理HTTP的侦听器的最后一个网络过滤器是[HTTP连接管理器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_connection_management#arch-overview-http-conn-man)（HCM）。该过滤器负责创建HTTP/2编解码器并管理HTTP过滤器链。在上面的例子中，它是唯一的网络过滤器。使用多个网络过滤器的网络过滤器链类似：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913160912634-50421199.png)

在响应路径中，网络过滤器执行的顺序与请求路径相反

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913160958137-1025398158.png)

#### 5.HTTP/2编解码器的解码

Envoy的HTTP/2编解码器基于[nghttp2](https://nghttp2.org/)，当TCP连接使用明文字节时(经过网络过滤器链变换后)，会被HCM调用。编解码器将字节流解码为一系列HTTP/2帧，并将连接解复用为多个独立的HTTP流。流复用是HTTP/2的一个关键特性，与HTTP/1相比具有明显的性能优势。每个HTTP流会处理一个单独的请求和响应。

编解码器也负责处理HTTP/2设置帧，以及连接级别的[流控制](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/source/docs/flow_control.md)。

编解码器负责抽象HTTP连接的细节，向HTTP连接管理器展示标准视图，并将连接的HTTP过滤器链拆分为多个流，每个流都有请求/响应标头/正文/尾部(无论协议是HTTP/1，HTTP/2还是HTTP/3 )。

#### 6.HTTP过滤器链的处理

对于每个HTTP流，HCM会实例化一个HTTP过滤器链，遵循上面为监听器和网络过滤器链建立的模式。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913162549808-708437824.png)

HTTP过滤器接口有三种类型：

- [StreamDecoderFilter](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/http/filter.h)：包含请求处理的回调方法
- [StreamEncoderFilter](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/http/filter.h)：包含响应处理的回调方法
- [StreamFilter](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/http/filter.h)：同时实现了*StreamDecoderFilter*和*StreamEncoderFilter*.

查看解码器过滤器接口：

```C++
virtual FilterHeadersStatus decodeHeaders(RequestHeaderMap& headers, bool end_stream) PURE;
virtual FilterDataStatus decodeData(Buffer::Instance& data, bool end_stream) PURE;
virtual FilterTrailersStatus decodeTrailers(RequestTrailerMap& trailers) PURE;
```























































