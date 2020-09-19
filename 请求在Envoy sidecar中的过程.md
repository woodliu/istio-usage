## Envoy 代理中的请求的生命周期

翻译自Envoy[官方文档](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)。

[TOC]

下面描述一个经过Envoy代理的请求的生命周期。首先会描述Envoy如何在请求路径中处理请求，然后描述请求从下游到达Envoy代理之后发生的内部事件。我们将跟踪该请求，直到其被分发到上游和响应路径中。

### 术语

Envoy会在代码和文档中使用如下术语：

- Cluster：逻辑上的服务，包含一系列的endpoints，Envoy会将请求转发到这些Cluster上。
- Downstream：连接到Envoy的实体。可能是一个本地应用(sidecar模型)或网络节点。非sidecar模型下体现为一个远端客户端。
- Endpoints：实现了逻辑服务的网络节点。Endpoints组成了Clusters。一个Cluster中的Endpoints为某个Envoy代理的upstream。
- Filter：连接或请求处理流水线的一个模块，提供了请求处理的某些功能。类似Unix的小型实用程序（过滤器）和Unix管道（过滤器链）的组合。
- Filter chain：一些列Filters。
- Listeners：负责绑定一个IP/端口的Envoy模块，接收新的TCP连接(或UDP数据包)以及对下游的请求进行编排。
- Upstream：Envoy转发请求到一个服务时连接的Endpoint。可能是一个本地应用或网络节点。非sidecar模型下体现为一个远端endpoint。

### 网络拓扑

一个请求是如何通过一个网络组件取决于该网络的模型。Envoy可能会使用大量网络拓扑。下面会重点介绍Envoy的内部运作方式，但在本节中会简要介绍Envoy与网络其余部分的关系。

Envoy起源于服务网格Sidecar代理，用于剥离应用程序的负载平衡，路由，可观察性，安全性和发现服务。在服务网格模型中，请求会经过作为网关的Envoy，或通过ingress或egress监听器到达一个Envoy。

- ingress 监听器会从其他节点接收请求，并转发到本地应用。本地应用的响应会经过Envoy发回下游。
- Egress 监听器会从本地应用接收请求，并转发到网络的其他节点。这些接收请求的节点通常也会运行Envoy，并接收经过它们的ingress 监听器的请求。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213011467-1050558629.png)

Envoy会用到除服务网格使用到的各种配置，例如，它可以作为一个内部的负载均衡器：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213205552-975050703.png)

或作为一个边缘网络的ingress/egress代理：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213253584-1924298712.png)

实际中，通常会在服务网格中混合使用Envoy的特性，在网格边缘作为ingress/egress代理，以及在内部作为负载均衡器。一个请求路径可能会经过多个Envoys。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213520222-1306838362.png)

Envoy可以配置为多层拓扑来实现可伸缩性和可靠性，一个请求会首先经过一个边缘Envoy，然后传递给第二个Envoy层。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911213654205-659093979.png)

以上所有场景中，请求通过下游的TCP，UDP或Unix域套接字到达一个指定的Envoy，然后由该Envoy通过TCP，UDP或UNIX域套接字转发到上游。下面仅关注单个Envoy代理。

### 配置

Envoy是一个可扩展的平台。通过如下条件可以组合成丰富的请求路径：

- L3/4协议，即TCP，UDP，UNIX域套接字
- L7协议，即 HTTP/1, HTTP/2, HTTP/3, gRPC, Thrift, Dubbo, Kafka, Redis 以及各种数据库
- 传输socket，即明文，TLS，ALTS
- 连接路由，即PROXY协议，源地址，动态转发
- 断路器以及异常值检测配置和激活状态
- 与网络相关的配置，HTTP, listener, 访问日志,健康检查, 跟踪和统计信息扩展

本例将涵盖如下内容：

- 在TCP连接上使用HTTP/2(带TLS)的请求的上下游
- 使用[HTTP连接管理器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_connection_management#arch-overview-http-conn-man)作为唯一的[网络过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/network_filters#arch-overview-network-filters)。
- 一个假定的CustomFilter，以及 `router <arch_overview_http_routing>` ([HTTP 过滤器链](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters#arch-overview-http-filters))
- [文件系统访问日志](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/access_logging#arch-overview-access-logs-sinks)
- [Statsd sink](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/metrics/v3/stats.proto#envoy-v3-api-msg-config-metrics-v3-statssink)
- 使用静态endpoints的单个[cluster](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/cluster_manager#arch-overview-cluster-manager)

假设使用如下静态的bootstrap配置文件，该配置仅包含一个listener和一个cluster。在listener中静态指定了路由配置，在cluster静态指定了endpoints。

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
          route_config: # 静态路由配置
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

- [Listener子系统](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners)：处理**下游**请求，同时负责管理下游请求的生命周期以及到客户端的响应路径。同时包含下游HTTP/2的编解码器。
- [Cluster子系统](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/cluster_manager#arch-overview-cluster-manager)：负责选择和配置到上游endpoint的连接，以及Cluster和endpoint的健康检查，负载均衡和连接池。同时包含上游HTTP/2的编解码器。

这两个子系统与HTTP router filter桥接在一起，用于将HTTP请求从下游转发到上游。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200911225519945-621108340.png)

我们使用术语[listener subsystem](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners) 和[cluster subsystem](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/cluster_manager#arch-overview-cluster-manager) 指代模块组以及由高层*ListenerManager* 和*ClusterManager*类创建的实例类。在下面讨论的很多组件都是由这些管理系统在请求前和请求过程中实例化的，如监听器, 过滤器链, 编解码器, 连接池和负载均衡数据结构。

Envoy有一个[基于事件的线程模型](https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310)。主线程负责生命周期、配置处理、统计等。[工作线程](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/threading_model#arch-overview-threading)用于处理请求。所有线程都围绕一个事件循环([libevent](https://libevent.org/))进行操作，任何给定的下游TCP连接(包括其中的所有多路复用流)，在其生命周期内都由一个工作线程进行处理。每个工作线程维护各自到上游endpoints的TCP连接池。UDP处理中会使用`SO_REUSEPORT`，通过内核一致性哈希将源/目标IP:端口元组散列到同一个工作线程。UDP过滤器状态会共享给特定的工作线程，过滤器负责根据需要提供会话语义。这与下面讨论的面向连接的TCP过滤器形成了对比，后者的过滤器状态以每个连接为基础，在HTTP过滤器的情况下，则以每个请求为基础。

工作线程很少会共享状态，且很少会并行运行。 该线程模型可以扩展到core数量非常多的CPU。

### 请求流

#### 总览

使用上面的示例配置简要概述请求和响应的生命周期：

1. 由运行在一个[工作线程](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/threading_model#arch-overview-threading)的Envoy [监听器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners)接收下游TCP连接
2. 创建并运行[监听过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listener_filters#arch-overview-listener-filters)链。该链可以提供SNI以及其他TLS之前的信息。一旦完成，该监听器会匹配到一个网络过滤器链。每个监听器都可能具有多个过滤链，这些filter链会匹配目标IP CIDR范围，SNI，ALPN，源端口等的某种组合。传输套接字（此例为TLS传输套接字）与该过滤器链相关联。
3. 在进行网络读取时，TLS传输套接字会从TCP连接中解密数据，以便后续做进一步的处理。
4. 创建并运行[网络过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/network_filters#arch-overview-network-filters)链。HTTP最重要的过滤器为HTTP连接管理器，它作为network filter链上的最后一个过滤器。
5. [HTTP连接管理器中](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_connection_management#arch-overview-http-conn-man)的HTTP/2编解码器将解密后的数据流从TLS连接上解帧并解复用为多个独立的流。每个流处理一个单独的请求和响应。
6. 对于每个HTTP流，会创建并运行一个[HTTP 过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters#arch-overview-http-filters)链。请求会首先经过CustomFilter，该过滤器可能会读取并修改请求。最重要的HTTP过滤器是路由过滤器，位于HTTP 过滤器链的末尾。当路由过滤器调用`decodeHeaders`时，会选择路由和cluster。数据流中的请求首部会转发到上游cluster对应的endpoint中。[router](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_routing#arch-overview-http-routing) 过滤器会从群集管理器中为匹配的cluster获取HTTP[连接池](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/connection_pooling#arch-overview-conn-pool)。
7. Cluster会指定[负载均衡](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/overview#arch-overview-load-balancing)来查找endpoint。cluster的断路器会检查是否允许一个新的流。如果endpoint的连接池为空或容量不足，则会创建一个到该endpoint的新连接。
8. 上游endpoint连接的HTTP/2编解码器会对请求的流(以及通过单个TCP连接到该上游的其他流)进行多路复用和帧化。
9. 上游endpoint连接的TLS传输socket会加密这些字节并写入到上游连接的TCP socket中。
10. 请求包含首部，可选的消息体和尾部，通过代理到达上游，并通过代理对下游进行响应。响应会以与请求[相反的顺序](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters#arch-overview-http-filters-ordering)通过HTTP过滤器，从路由过滤器开始，然后经过CustomFilter。
11. 完成响应后会销毁流，更新统计信息，写入访问日志并最终确定跟踪范围。

我们将在以下各节中详细介绍每个步骤。

#### 1.Listener TCP连接的接收

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200912151231417-1211519644.png)

*ListenerManager*负责获取[监听器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners)的配置，并实例化绑定到各自IP/端口的多个`Listener`实例。监听器的状态可能为：

- *Warming*：监听器等待配置依赖(即，路配置，动态secret)。此时监听器无法接收TCP连接。
- *Active*：监听器绑定到其IP/端口，可以接收TCP连接。
- *Draining*：监听器不再接收新的TCP连接，现有的TCP连接可以在一段时间内继续使用。

每个[工作线程](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/threading_model#arch-overview-threading)会为每个监听器维护各自的监听器实例。每个监听器可能通过**SO_REUSEPORT** 绑定到相同的端口，或共享绑定到该端口的socket。当接收到一个新的TCP连接，内核会决定哪个工作线程来接收该连接，然后由该工作线程对应的监听器调用`Server::ConnectionHandlerImpl::ActiveTcpListener::onAccept()`。

#### 2.监听过滤链和网络过滤器链的匹配

工作线程的监听器然后会创建并运行[监听过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listener_filters#arch-overview-listener-filters)链。过滤器链是通过每个过滤器的过滤器工厂创建的，该工厂会感知过滤器的配置，并为每个连接或流创建新的过滤器实例。

在TLS 过滤器配置下，监听过滤器链会包含[TLS检查](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/listener_filters/tls_inspector#config-listener-filters-tls-inspector)过滤器(`envoy.filters.listener.tls_inspector`)。该过滤器会检查初始的TLS握手，并抽取server name(SNI)，然后使用SNI进行过滤器链的匹配。尽管`tls_inspector`会明确出现在监听过滤器链配置中，但Envoy还能够在监听过滤器链需要SNI（或ALPN）时自动将其插入。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913152453326-1937005929.png)

TLS检查过滤器实现了 [ListenerFilter](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/network/filter.h)接口。所有的过滤器接口，无论是监听或网络/HTTP过滤器，都需要实现特定连接或流事件的回调方法，*ListenerFilter*中为：

```c++
virtual FilterStatus onAccept(ListenerFilterCallbacks& cb) PURE;
```

`onAccept()`允许在TCP accept处理时运行一个过滤器。回调方法的`FilterStatus`控制监听过滤器链将如何运行。监听过滤器可能会暂停过滤器链，后续再恢复运行，如响应另一个服务进行的RPC请求。

在过滤器链进行匹配时，会抽取监听过滤器和连接的属性，提供给用于处理连接的网络过滤器链和传输socket。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913154118823-389554441.png)

#### 3.TLS传输socket的解密

Envoy通过[TransportSocket](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/include/envoy/network/transport_socket.h)扩展接口提供了插件式的传输socket。传输socket遵循TCP连接的生命周期事件，使用网络buffer进行读写。传输socket需要实现如下关键方法：

```c++
virtual void onConnected() PURE;
virtual IoResult doRead(Buffer::Instance& buffer) PURE;
virtual IoResult doWrite(Buffer::Instance& buffer, bool end_stream) PURE;
virtual void closeSocket(Network::ConnectionEvent event) PURE;
```

当一个TCP连接可以传输数据时，`Network::ConnectionImpl::onReadReady()`会通过`SslSocket::doRead()`调用TLS传输socket。传输socket然后会在TCP连接上进行TLS握手。TLS握手结束后，`SslSocket::doRead()`会给`Network::FilterManagerImpl`实例提供一个解密的字节流，该实例负责管理网络过滤器链。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913154822343-2090496267.png)

需要注意的是，无论是TLS握手还是过滤器处理流程的暂停，都不会真正阻塞任何操作。由于Envoy是基于事件的，因此任何需要额外数据才能进行处理的情况都将导致提前完成事件，并将CPU转移给另一个事件。如当网络提供了更多的可读数据时，该读事件将会触发TLS握手恢复。

#### 4.网络过滤器链的处理

与监听过滤器链相同，Envoy会通过*Network::FilterManagerImpl*，从对应的过滤器工厂实例化一些列网络过滤器。每个新连接的实例都是新的。与传输socket相同，网络过滤器也会遵循TCP的生命周期事件，并在来自传输socket中的数据可用时被唤醒。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913155624620-2072767193.png)

网络过滤器包含一个pipeline，与一个连接一个的传输socket不同，网络过滤器分为三种：

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

HTTP的监听器的最后一个网络过滤器是[HTTP连接管理器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_connection_management#arch-overview-http-conn-man)（HCM）。该过滤器负责创建HTTP/2编解码器并管理HTTP过滤器链。在上面的例子中，它是唯一的网络过滤器。使用多个网络过滤器的网络过滤器链类似：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913160912634-50421199.png)

在响应路径中，网络过滤器执行的顺序与请求路径相反

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913160958137-1025398158.png)

#### 5.HTTP/2编解码器的解码

Envoy的HTTP/2编解码器基于[nghttp2](https://nghttp2.org/)，当TCP连接使用明文字节时(经过网络过滤器链变换后)，会被HCM调用。编解码器将字节流解码为一系列HTTP/2帧，并将连接解复用为多个独立的HTTP流。流复用是HTTP/2的一个关键特性，与HTTP/1相比具有明显的性能优势。每个HTTP流会处理一个单独的请求和响应。

编解码器也负责处理HTTP/2设置帧，以及连接级别的[流控制](https://github.com/envoyproxy/envoy/blob/5d95032baa803f853e9120048b56c8be3dab4b0d/source/docs/flow_control.md)。

编解码器负责抽象HTTP连接的细节，向HTTP连接管理器展示标准视图，并将连接的HTTP过滤器链拆分为多个流，每个流都有请求/响应标头/正文/尾部(无论协议是HTTP/1，HTTP/2还是HTTP/3 )。

#### 6.HTTP过滤器链的处理

对于每个HTTP流，HCM会实例化一个[HTTP过滤器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters#arch-overview-http-filters)链，遵循上面为监听器和网络过滤器链建立的模式。

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

HTTP过滤器遵循HTTP请求的生命周期，而不针对连接缓冲区和事件进行操作，如`decodeHeaders()`使用HTTP首部作为参数，而不是字节buffer。与网络和监听过滤器一样，返回的`FilterStatus`提供了管理过滤器链控制流的功能。

当可以使用HTTP/2编解码器处理HTTP请求首部时，会首先传递给在CustomFilter中的`decodeHeaders()`。如果返回的`FilterHeadersStatus`为`Continue`，则HCM会将首部(可能会被CustomFilter修改)传递给路由过滤器。

解码器和编解码过滤器运行在请求路径上，编码器和编码解码过滤器运行在响应路径上。考虑如下过滤器链：



![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200918162822691-1449911917.png)

请求路径为：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200918162958264-342204392.png)

响应路径为:

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200918163048422-742634825.png)

当在[路由](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_routing#arch-overview-http-routing)过滤器中调用`decodeHeaders()`时，会确定路由选择并挑选cluster。HCM会在HTTP过滤器链执行开始时从`RouteConfiguration`中选择一个路由，该路由被称为缓存路由。过滤器可能会通过要求HCM清除路由缓存并请求HCM重新评估路由选择来修改首部，并导致选择一个新的路由。当调用路由过滤器时，也就确定了路由。显选择的路由会指向一个上游cluster名称。然后路由过滤器会向*ClusterManager* 为该cluster请求一个HTTP[连接池](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/connection_pooling#arch-overview-conn-pool)。该过程涉及负载平衡和连接池，将在下一节中讨论。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200918164329050-451744694.png)

HTTP连接池是用来在router中构建一个*UpstreamRequest*对象，该对象封装了用于处理上游HTTP请求的HTTP编码和解码的回调方法。一旦在HTTP连接池的连接上分配了一个流，则会通过`UpstreamRequest::encoderHeaders()`将请求首部转发到上游endpoint。

路由过滤器负责(从HTTP连接池上分配的流上的)到上游的请求的生命周期管理，同时也负责请求超时，重试和亲和性等。

#### 7.负载均衡

每个cluster都有一个[负载均衡](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/overview#arch-overview-load-balancing)，当接收到一个请求时会选择一个endpoint。Envoy支持多种类型的负载均衡算法，如基于权重的轮询，Maglev，负载最小，随机等算法。负载均衡会从静态的bootstrap配置，DNS，动态xDS以及主动/被动健康检查中获得其需要处理的内容。更多详情参见[官方文档](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/overview#arch-overview-load-balancing)。

一旦选择了endpoint，会使用[连接池](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/connection_pooling#arch-overview-conn-pool)来为该endpoint选择一个连接来转发请求。如果没有到该主机的连接，或所有的连接已经达到了并发流的上线，则会建立一条新的流，并将它放到连接池里(除非触发了群集的最大连接的[断路器](https://github.com/envoyproxy/envoy/blob/3d481305fad358c4e47ee47869590b18d592e893/arch_overview_circuit_breakers))。如果配置了流的最大生命时间，且已经达到了该时间点，那么此时会在连接池中分配一个新的连接，并终止旧的HTTP/2连接。此外还会检查其他断路器，如到一个cluster的最大并发请求等。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200919193428731-886559580.png)

#### 8.HTTP/2 编解码器的编码

连接的HTTP/2的编解码器会对单条TCP连接上的到达相同上游的其他请求流进行多路复用，与[HTTP/2编解码器的解码](HTTP/2编解码器的解码)是相反的

与下游HTTP/2编解码器一样，上游的编解码器负责采用Envoy的HTTP标准抽象，即多个流在单个连接上与请求/响应标头/正文/尾部进行复用，并通过生成一系列HTTP/2帧将其映射到HTTP/2的细节中。

#### 9.TLS传输socket的加密

上游endpoint连接的TLS传输socket会加密来自HTTP/2编解码器的输出，并将其写入到上游连接的TCP socket中。 与[TLS传输套接字的解码](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request#life-of-a-request-tls-decryption)一样，在我们的示例中，群集配置了传输套接字，用来提供TLS传输的安全性。上游和下游传输socket扩展都存在相同的接口。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200919195135252-755646571.png)

#### 10.响应路径和HTTP生命周期

请求包含首部，可选择的主体和尾部，通过代理到上游，并将响应代理到下游。响应会通过以与请求[相反的顺序](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters#arch-overview-http-filters-ordering)经过HTTP和network过滤器。

HTTP过滤器会调用解码器/编码器请求生命周期事件的各种回调，例如 当转发响应尾部或请求主体被流式传输时。类似地，读/写network过滤器还将在数据在请求期间继续在两个方向上流动时调用它们各自的回调。

endpoint的[异常检测](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier#arch-overview-outlier-detection)状态会随着请求的进行而修改。

当上游响应到达流的末端后即完成了一个请求。即接收到尾部或带有最终流集的响应头/主体时。这个流程在`Router::Filter::onUpstreamComplete()`在进行处理。

一个请求有可能提前结束，可能的原因为：

- 请求超时
- 上游endpoint的流被重置
- HTTP过滤器流被重置
- 出发断路器
- 不可用的上游资源，如缺少路由指定的cluster
- 不健康的endpoints
- Dos攻击
- 无效的HTTP协议
- 通过HCM或HTTP过滤器进行了本地回复。如HTTP过滤器可能会因为频率限制而返回429响应。

如果上游响应还没有发送，则Envoy会原因生成一个内部的响应；如果响应首部已经转发到了下游，则会重置流。更多参见Envoy的[调试FAQ](https://www.envoyproxy.io/docs/envoy/latest/faq/overview#faq-overview-debug)。

#### 11.请求后的处理

一旦请求完成，则流会被销毁。发生的事件如下：

- 更新请求后的[统计](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/statistics#arch-overview-statistics)(如时间，活动的请求，更新，检查检查等)。但有些统计会在请求过程中进行更新。此时尚未将统计信息写入统计[接收器](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-field-config-bootstrap-v3-bootstrap-stats-sinks)，它们由主线程定期进行批处理和写入。在上述示例中，这是一个statsd接收器。
- 将[访问日志](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/access_logging#arch-overview-access-logs)写入访问日志接收器，在上述示例中，为一个文件访问日志。
- 确定trace spans。如果上述例子进行了请求追踪，则会生成一个trace span，描述了请求的持续时间和细节，在处理请求首部是HCM会创建trace span，并在请求后处理过程中由HCM进行最终确定。



















