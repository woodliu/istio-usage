# Istio安全-证书管理

[TOC]

*注：本章更新至1.8版本*

## 插入现有CA证书

本节展示了管理员如何使用现有的根证书来授权istio证书，签发证书和密钥，不使用Istio自动生成的证书。

默认情况下，istio的CA会生成一个自签的根证书和密钥，并使用它们签发负载证书。istio的CA也可以使用管理员指定的证书和密钥，以及管理员指定的根证书来签发负载证书。

根CA是网格中所有负载信任的根证书。每个Istio CA会使用一个中间CA来签发密钥和证书，该CA由根CA签发。当一个网格中存在多个Istio CA时，会在CA之间建立起信任层级。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200925155758195-1735784080.png)

本节展示如何将这些证书和密钥插入Istio的CA。

### 插入现有证书和密钥

> 对于生产环境，最好在一台离线机器上执行如下步骤，确保将根密钥暴露给尽可能少的人

1. 创建一个保存证书和密钥的目录

   ```shell
   $ mkdir -p certs
   $ pushd certs
   ```

2. 生成根证书和密钥

   ```shell
   $ make -f ../tools/certs/Makefile.selfsigned.mk root-ca
   ```

   上述命令将生成如下文件：

   - `root-cert.pem`: 根证书
   - `root-key.pem`: 根密钥
   - `root-ca.conf`:  `openssl` 使用该配置来生成根证书
   - `root-cert.csr`: 为根证书生成的CSR

3. 生成中间证书和密钥

   ```shell
   $ make -f ../tools/certs/Makefile.selfsigned.mk cluster1-cacerts
   ```

   执行如上命令会生成一个名为`cluster1`的目录，包含如下文件，`root-cert.pem`签发了`ca-cert.pem`

   - `ca-cert.pem`: 中间证书
   - `ca-key.pem`: 中间密钥
   - `cert-chain.pem`: Istiod使用的证书链
   - `root-cert.pem`: 根证书
   - `intermediate.conf`: `openssl` 使用该配置来生成中间证书
   - `cluster-ca.csr`: 为中间证书生成的CSR

   > 可以将`cluster1`替换为任何字符串。例如`make mycluster-certs`将会生成名为mycluster的目录

   > 为了配置其他Istio CA，可以重复执行上述步骤来生成不同名称的证书目录

4. 创建一个`cacerts` secret，包含如下输入文件`ca-cert.pem`, `ca-key.pem`, `root-cert.pem` 和`cert-chain.pem`。需要注意的是创建出来的secret的名称必须是`cacerts`，这样才能被istio正确挂载。

   ```shell
   $ kubectl create namespace istio-system
   $ kubectl create secret generic cacerts -n istio-system \
         --from-file=cluster1/ca-cert.pem \
         --from-file=cluster1/ca-key.pem \
         --from-file=cluster1/root-cert.pem \
         --from-file=cluster1/cert-chain.pem
   ```

5. 返回Istio安装的顶层目录

   ```shell
   $ popd
   ```

### 部署Istio

使用demo profile，Istio会从挂载的secret文件中读取证书

```shell
$ istioctl install --set profile=demo
```

在下面的例子中，istio的CA证书(`ca-cert.pem`)与根证书(`root-cert.pem`)不同，因此负载无法通过根证书验证工作负载证书，需要使用一个`cert-chain.pem`来指定信任的证书链，该证书链包含负载和根CA之间的所有中间CA，在此例子中，它包含了istio的CA签名证书，因此`cert-chain.pem`和`ca-cert.pem`是**相同**的。注意，如果`ca-cert.pem`与`root-cert.pem`相同，那么`ca-chain.pem`文件应该是空的。

### 配置示例services

1. 部署httpbin和sleep示例services

   ```shell
   $ kubectl create ns foo
   $ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
   $ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
   ```

2. 部署一个策略，使得foo命名空间的负载仅接受mutual TLS流量

   ```shell
   $ kubectl apply -n foo -f - <<EOF
   apiVersion: "security.istio.io/v1beta1"
   kind: "PeerAuthentication"
   metadata:
     name: "default"
   spec:
     mtls:
       mode: STRICT
   EOF
   ```

### 校验证书

本节中会校验插入到CA中的证书是否签发了负载证书。

1. sleep 20s，等待mTLS策略下`httpbin`的证书链生效。由于CA证书是自签的，因此openssl命令会返回`verify error:num=19:self signed certificate in certificate chain`错误。

   ```shell
   $ sleep 20; kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c istio-proxy -n foo -- openssl s_client -showcerts -connect httpbin.foo:8000 > httpbin-proxy-cert.txt
   ```

2. 解析证书链中的证书

   ```shell
   $ sed -n '/-----BEGIN CERTIFICATE-----/{:start /-----END CERTIFICATE-----/!{N;b start};/.*/p}' httpbin-proxy-cert.txt > certs.pem
   $ awk 'BEGIN {counter=0;} /BEGIN CERT/{counter++} { print > "proxy-cert-" counter ".pem"}' < certs.pem
   ```

3. 校验根证书与管理员指定的证书相同

   ```shell
   $ openssl x509 -in samples/certs/root-cert.pem -text -noout > /tmp/root-cert.crt.txt
   $ openssl x509 -in ./proxy-cert-3.pem -text -noout > /tmp/pod-root-cert.crt.txt
   $ diff -s /tmp/root-cert.crt.txt /tmp/pod-root-cert.crt.txt
   Files /tmp/root-cert.crt.txt and /tmp/pod-root-cert.crt.txt are identical
   ```

4. 校验CA证书与管理员指定的相同

   ```shell
   $ openssl x509 -in samples/certs/ca-cert.pem -text -noout > /tmp/ca-cert.crt.txt
   $ openssl x509 -in ./proxy-cert-2.pem -text -noout > /tmp/pod-cert-chain-ca.crt.txt
   $ diff -s /tmp/ca-cert.crt.txt /tmp/pod-cert-chain-ca.crt.txt
   Files /tmp/ca-cert.crt.txt and /tmp/pod-cert-chain-ca.crt.txt are identical
   ```

5. 校验从根证书到负载证书的证书链。下面使用中间证书和根证书组成的证书链来校验其签发了负载证书

   ```shell
   $ openssl verify -CAfile <(cat samples/certs/ca-cert.pem samples/certs/root-cert.pem) ./proxy-cert-1.pem
   ./proxy-cert-1.pem: OK
   ```

### 卸载

卸载证书`cacert`和`foo`以及`istio-system`命名空间

```shell
$ kubectl delete secret cacerts -n istio-system
$ kubectl delete ns foo istio-system
```

## Istio的DNS证书管理

本节展示如何使用 [Chiron](https://istio.io/latest/blog/2019/dns-cert/)提供和管理DNS证书，Chiron是一个与istiod相连的轻量型组件，它使用kubernetes的CA API签发证书，无需管理私钥。有如下优势：

- 与isitod不同，这种方式无需维护签发的私钥，增强了安全性
- 简化了将根证书分发到TLS客户端。客户端不再需要等待istiod生成并分发其CA证书

首先使用`istioctl`安装istio，并配置DNS证书，当istiod启动后会读取该配置：

```shell
$ cat <<EOF > ./istio.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    certificates:
      - secretName: dns.example1-service-account
        dnsNames: [example1.istio-system.svc, example1.istio-system]
      - secretName: dns.example2-service-account
        dnsNames: [example2.istio-system.svc, example2.istio-system]
EOF
$ istioctl install -f ./istio.yaml
```

可以看到在istio-system下生成了两个secret：

```shell
# kubectl get secret -n istio-system |grep dns
dns.example1-service-account                       istio.io/dns-key-and-cert             3      81s
dns.example2-service-account                       istio.io/dns-key-and-cert             3      81s
```

### DNS证书的提供和管理

Istio根据用户的配置为DNS证书提供了DNS名字和secret名称。DNS证书由kubernetes CA签发，并根据配置保存到secret中。istio也管理着DNS证书的生命周期，包括证书滚动和重新生成。

### 配置DNS证书

可以在 `istioctl install` 命令中使用`IstioControlPlane`用户资源对istio进行配置。`dnsNames`字段用于设定证书中的DNS名称，`secretName`字段指定保存证书和密钥的kubernetes secret的名称。

### 检查提供的DNS证书

在配置istio生成DNS证书并保存到secret后，需要校验提供的证书是否能够正确运行。

为了校验istio前面例子中生成的`dns.example1-service-account`的DNS证书，以及校验该证书是否包含配置的DNS名称，需要获取kubernetes的secret，解析并对其解码，查看其具体内容：

```shell
$ kubectl get secret dns.example1-service-account -n istio-system -o jsonpath="{.data['cert-chain\.pem']}" | base64 --decode | openssl x509 -in /dev/stdin -text -noout
```

输出的文本包括：

```
X509v3 Subject Alternative Name:
    DNS:example1.istio-system.svc, DNS:example1.istio-system
```

### 重新生成DNS证书

istio可以在DNS证书被错删的情况下重新生成证书。

1. 删除前面保存的DNS证书

   ```shell
   $ kubectl delete secret dns.example1-service-account -n istio-system
   ```

2. 校验istio重新生成了删除的DNS证书，且证书包含配置的DNS名称。需要从kubernetes获取secret，解析并对其解码，获取其内容：

   ```shell
   $sleep 10; kubectl get secret dns.example1-service-account -n istio-system -o jsonpath="{.data['cert-chain\.pem']}" | base64 --decode | openssl x509 -in /dev/stdin -text -noout
   ```

输出包括

```shell
X509v3 Subject Alternative Name:
    DNS:example1.istio-system.svc, DNS:example1.istio-system
```

### 卸载

```shell
$ kubectl delete ns istio-system
```