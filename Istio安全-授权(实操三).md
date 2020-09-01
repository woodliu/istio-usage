# Istio安全-授权

[TOC]

## 授权HTTP流量

本节展示如何在**istio网格中**授权HTTP流量。

部署[Bookinfo](https://istio.io/latest/docs/examples/bookinfo/#deploying-the-application)。由于下例在策略中使用了principal和namespace，因此需要启用mutual TLS。

### 为使用HTTP流量的负载配置访问控制

本任务展示了如何使用istio的授权设置访问控制。首先，使用简单的`deny-all`策略拒绝所有到负载的请求，然后增量地授权到负载的访问。

> 下面使用kubernetes的service account授权istio网格中的HTTP访问

1. 在`default`命名空间中创建`deny-all`策略。该策略没有`selector`字段，将会应用到default命名空间中的所有负载上。`sepc:`字段为空`{}`，意味着禁止所有的流量。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: deny-all
     namespace: default #禁止任何对default命名空间中的负载的访问
   spec:
     {}
   EOF
   ```

   刷新Boofinfo `productpage`的页面，可以看到错误`RBAC: access denied`，即`deny-all`策略已经生效，且istio没有其他规则允许流量访问网格中的负载。

2. 执行如下命令创建一个 `productpage-viewer` 允许使用`GET`方法访问`productpage`负载。该策略并没有设置from字段，意味着允许所有用户和工作负载进行访问：

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: "security.istio.io/v1beta1"
   kind: "AuthorizationPolicy"
   metadata:
     name: "productpage-viewer"
     namespace: default
   spec:
     selector:
       matchLabels:
         app: productpage #允许对default命名空间中的负载productpage的GET访问
     rules:
     - to:
       - operation:
           methods: ["GET"]
   EOF
   ```

   再次刷新Boofinfo `productpage`的页面，此时可以看到`Bookinfo Sample`页面，但该页面同时也显示了如下错误：

   - `Error fetching product details`
   - `Error fetching product reviews`

   这些错误符合预期，因为并没给`productpage`负载授权访问`details`和`reviews`负载。下面，需要配置一个策略来授权访问这些负载。

3. 执行如下命令创建`details-viewer`策略来允许`productpage`负载使用 `cluster.local/ns/default/sa/bookinfo-productpage`  service account发起`GET`请求访问`details`：

   > 在`productpage`的deployment中可以看到它使用了service account `bookinfo-productpage`

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: "security.istio.io/v1beta1"
   kind: "AuthorizationPolicy"
   metadata:
     name: "details-viewer"
     namespace: default
   spec:
     selector:
       matchLabels:
         app: details
     rules:
     - from:
       - source:
           principals: ["cluster.local/ns/default/sa/bookinfo-productpage"] #允许使用default命名空间中的sa bookinfo-productpage访问details
       to:
       - operation:
           methods: ["GET"]
   EOF
   ```

4. 执行如下命令创建`reviews-viewer`策略来允许`productpage`负载使用 `cluster.local/ns/default/sa/bookinfo-productpage`  service account发起`GET`请求访问`reviews `：

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: "security.istio.io/v1beta1"
   kind: "AuthorizationPolicy"
   metadata:
     name: "reviews-viewer"
     namespace: default
   spec:
     selector:
       matchLabels:
         app: reviews
     rules:
     - from:
       - source:
           principals: ["cluster.local/ns/default/sa/bookinfo-productpage"] #允许使用default命名空间中的sa bookinfo-productpage访问reviews,信任域为cluster.local
       to:
       - operation:
           methods: ["GET"]
   EOF
   ```

   刷新Bookinfo `productpage`，可以在`Bookinfo Sample`页面的左边看到`Book Details`，并在页面右侧看到`Book Reviews`，但在`Book Reviews`一栏可以看到错误`Ratings service currently unavailable`。

   这是因为`reviews`负载没有权限访问`ratings`负载。为了解决这个问题， 需要授权`reviews`负载访问`ratings`负载。下面配置一个策略来授权`reviews`负载进行访问。

5. 运行下面命令创建策略`ratings-viewer`来允许`reviews`负载使用 `cluster.local/ns/default/sa/bookinfo-reviews`  service account发起`GET`请求访问`ratings`：

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: "security.istio.io/v1beta1"
   kind: "AuthorizationPolicy"
   metadata:
     name: "ratings-viewer"
     namespace: default
   spec:
     selector:
       matchLabels:
         app: ratings
     rules:
     - from:
       - source:
           principals: ["cluster.local/ns/default/sa/bookinfo-reviews"]
       to:
       - operation:
           methods: ["GET"]
   EOF
   ```

   刷新Bookinfo `productpage`页面，可以在`Book Reviews`一栏中看到黑色和红色的ratings信息。

### 卸载

```shell
$ kubectl delete authorizationpolicy.security.istio.io/deny-all
$ kubectl delete authorizationpolicy.security.istio.io/productpage-viewer
$ kubectl delete authorizationpolicy.security.istio.io/details-viewer
$ kubectl delete authorizationpolicy.security.istio.io/reviews-viewer
$ kubectl delete authorizationpolicy.security.istio.io/ratings-viewer
```

## 授权TCP流量

本节展示如何授权istio网格中的TCP流量。

### 部署

- 在同一个命名空间`foo`中部署两个名为`sleep`和`tcp-echo`的负载，两个负载都允许了Envoy代理。`tcp-echo`负载监听端口9000，9001和9002，并回显收到的所有带有前缀hello的流量。例如，如果发送"world"到`tcp-echo`，则会响应"hello world"。`tcp-echo` kubernetes service对象仅声明端口9000和9001，并忽略端口9002。pass-through过滤器将处理端口9002的流量。使用如下命令部署命名空间和负载：

  > openshift注意创建NetworkAttachmentDefinition

  ```shell
  $ kubectl create ns foo
  $ kubectl apply -f <(istioctl kube-inject -f samples/tcp-echo/tcp-echo.yaml) -n foo
  $ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
  ```

- 使用如下命令校验`sleep`可以通过9000和9001端口连接`tcp-echo`

  ```shell
  # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c 'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
  hello port 9000
  connection succeeded
  ```

  ```shell
  # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c 'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
  hello port 9001
  connection succeeded
  ```

- 校验`sleep`可以连接`tcp-echo`的9002端口。此时需要通过tcp-echo的pod IP发送流量，这是因为tcp-echo的kubernetes service对象中并没有定义9002端口。获取pod IP并使用如下命令发送请求：

  ```shell
  # TCP_ECHO_IP=$(kubectl get pod "$(kubectl get pod -l app=tcp-echo -n foo -o jsonpath={.items..metadata.name})" -n foo -o jsonpath="{.status.podIP}")
  # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c "echo \"port 9002\" | nc $TCP_ECHO_IP 9002" | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
  [root@bastion istio-1.7.0]# kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c "echo \"port 9002\" | nc $TCP_ECHO_IP 9002" | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
  hello port 9002
  connection succeeded
  ```

  可以看到tcp-echo的k8s service仅暴露了9000和9001端口：

  ```shell
  # oc get svc -n foo
  NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
  sleep      ClusterIP   10.84.92.66    <none>        80/TCP              4m31s
  tcp-echo   ClusterIP   10.84.85.246   <none>        9000/TCP,9001/TCP   4m32s
  ```

### 配置TCP负载的访问控制

1. 为`foo`命名空间中的`tcp-echo`负载创建`tcp-policy`授权策略，运行如下命令创建一个授权策略，允许到9000和9001的请求：

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: tcp-policy
     namespace: foo
   spec:
     selector:
       matchLabels:
         app: tcp-echo #对匹配标签的负载允许到端口9000/9001的访问
     action: ALLOW
     rules:
     - to:
       - operation:
          ports: ["9000", "9001"]
   EOF
   ```

2. 使用如下命令校验允许到9000的访问

   > 此时即使没有上面的策略，到9000/9001的访问都是允许的。因为istio默认使用宽容模式，区别是如果该服务暴露的不止9000/9001端口，那么其他端口是无法访问的。

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c 'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   hello port 9000
   connection succeeded
   ```

3. 使用如下命令校验允许到9001的访问

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c 'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   hello port 9001
   connection succeeded
   ```

4. 校验到9002的请求是拒绝的。这是由授权策略执行的，该策略也适用于pass through过滤器链，即使`tcp-echo` kubernetes service对象没有明确声明该port。

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c "echo \"port 9002\" | nc $TCP_ECHO_IP 9002" | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   command terminated with exit code 1
   connection rejected
   ```

5. 使用如下命令将策略更新为仅在9000端口上允许HTTP的GET方法

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: tcp-policy
     namespace: foo
   spec:
     selector:
       matchLabels:
         app: tcp-echo
     action: ALLOW
     rules:
     - to:
       - operation:
           methods: ["GET"]
           ports: ["9000"]
   EOF
   ```

6. 校验到9000的端口的请求被拒绝了。这是因为此时规则仅允许HTTP格式的TCP流量。istio会忽略无效的ALLOW规则。最终结果是由于请求不匹配任何ALLOW规则，而被被拒绝。执行如下命令进行校验：

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c 'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   connection rejected
   ```

7. 校验到9001的请求被拒绝了。原因同样是因为请求并没有匹配任何ALLOW规则

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c 'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   connection rejected
   ```

8. 使用如下命令将策略更新为DENY，拒绝到9000的请求(当然此时其他端口会走默认的宽容模式)

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: tcp-policy
     namespace: foo
   spec:
     selector:
       matchLabels:
         app: tcp-echo
     action: DENY
     rules:
     - to:
       - operation:
           methods: ["GET"]
           ports: ["9000"]
   EOF
   ```

9. 校验到9000端口的请求被拒绝。它与上面无效的ALLOW规则(istio忽略了整个规则)不同，istio忽略了仅支持HTTP的字段`methods`，但使用了`ports`，导致匹配到这个端口的请求被拒绝：

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c 'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   connection rejected
   ```

10. 校验到端口9001的请求是允许的，这是因为请求并不匹配DENY策略的`ports`字段

    ```shell
    # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- sh -c 'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
    hello port 9001
    connection succeeded
    ```

### 卸载

```shell
$ kubectl delete namespace foo
```

> action字段中的默认值为ALLOW，且ALLOW中的规则的关系是`AND`，而DENY中的规则的关系是`OR`

## 使用JWT进行授权

本任务展示如何设置istio授权策略来执行基于JSON Web Token(JWT)的访问。Istio授权策略同时支持字符串类型和字符串列表类型的JWT claims。

### 部署

在`foo`命名空间中部署部署两个负载：`httpbin`和`sleep`。两个负载都运行了Envoy代理。使用如下命令进行部署：

```shell
$ kubectl create ns foo
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
$ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
```

校验可以使用`sleep`连接`httpbin`：

```shell
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
200
```

### 使用有效的JWT和列表类型的claims允许请求

1. 使用如下命令在`foo`命名空间中为`httpbin`负载创建`jwt-example`请求认证策略。该策略会让`httpbin`负载接受一个由`testing@secure.istio.io`发布的JWT。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: "security.istio.io/v1beta1"
   kind: "RequestAuthentication"
   metadata:
     name: "jwt-example"
     namespace: foo
   spec:
     selector:
       matchLabels:
         app: httpbin
     jwtRules:
     - issuer: "testing@secure.istio.io"
       jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.7/security/tools/jwt/samples/jwks.json"
   EOF
   ```

   >jwksUri为开放公钥的接口地址，用于获取公钥，进而对token进行校验

2. 校验带无效JWT的请求被拒绝了：

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/headers" -s -o /dev/null -H "Authorization: Bearer invalidToken" -w "%{http_code}\n"
   401
   ```

3. 校验不带JWT的请求是允许的，因为此时没有配置授权策略

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/headers" -s -o /dev/null -w "%{http_code}\n"
   200
   ```

4. 下面命令会给`foo`命名空间中的`httpbin`负载创建`require-jwt`授权策略。该策略会要求所有的请求都必须包含一个有效的JWT(`requestPrincipal`为`testing@secure.istio.io/testing@secure.istio.io`)。Istio通过将JWT token的`iss`和`sub`与一个/分隔符组合来构造`requestPrincipal`：

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: require-jwt
     namespace: foo
   spec:
     selector:
       matchLabels:
         app: httpbin
     action: ALLOW
     rules:
     - from:
       - source:
          requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
   EOF
   ```

5. 获取key为`iss`和`sub`，且值(`testing@secure.istio.io`)相同的JWT。这会导致istio生成属性`requestPrincipal`，对应值为 `testing@secure.istio.io/testing@secure.istio.io`:

   ```shell
   # TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.7/security/tools/jwt/samples/demo.jwt -s) && echo "$TOKEN" | cut -d '.' -f2 - | base64 --decode -
   {"exp":4685989700,"foo":"bar","iat":1532389700,"iss":"testing@secure.istio.io","sub":"testing@secure.istio.io"}
   ```

6. 校验带有效JWT的请求是允许的：

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/headers" -s -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
   200
   ```

7. 校验带无效JWT的请求被拒绝：

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/headers" -s -o /dev/null -w "%{http_code}\n"
   403
   ```

8. 如下命令会更新`require-jwt`授权策略，要求JWT具有一个名为`groups`的claim，且置为`group1`：

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: require-jwt
     namespace: foo
   spec:
     selector:
       matchLabels:
         app: httpbin
     action: ALLOW
     rules:
     - from:
       - source:
          requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
       when:
       - key: request.auth.claims[groups]
         values: ["group1"]
   EOF
   ```

9. 获取将 `groups` claim设置为字符串列表的JWT： `group1` 和`group2`:

   ```shell
   # TOKEN_GROUP=$(curl https://raw.githubusercontent.com/istio/istio/release-1.7/security/tools/jwt/samples/groups-scope.jwt -s) && echo "$TOKEN_GROUP" | cut -d '.' -f2 - | base64 --decode -
   {"exp":3537391104,"groups":["group1","group2"],"iat":1537391104,"iss":"testing@secure.istio.io","scope":["scope1","scope2"],"sub":"testing@secure.istio.io"}
   ```

10. 校验在请求的JWT的`groups` claim中带`group1`是允许的：

    ```shell
    # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/headers" -s -o /dev/null -H "Authorization: Bearer $TOKEN_GROUP" -w "%{http_code}\n"
    200
    ```

11. 校验请求的JWT中没有`groups` claim是被拒绝的

    ```shell
    # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/headers" -s -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
    403
    ```

### 卸载

```shell
$ kubectl delete namespace foo
```

### 总结

[RFC 7519](https://tools.ietf.org/html/rfc7519).定义了JWT token的格式。Istio的JWT认证流程使用了[OAuth 2.0](https://tools.ietf.org/html/rfc6749) 和[OIDC 1.0](http://openid.net/connect)，可以使用[jwks](https://auth0.com/docs/tokens/json-web-tokens/json-web-key-sets)字段或`jwksUri`字段标识公钥的[提供方](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderMetadata)。

`AuthorizationPolicy` rule 规则中与 JWT 相关的字段包括：

| field                                                        | sub field                | JWT claims   |
| ------------------------------------------------------------ | ------------------------ | ------------ |
| from.[source](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#source) | requestPrincipals        | iss/sub      |
| from.[source](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#source) | notRequestPrincipals     | iss/sub      |
| when.key                                                     | request.auth.principal   | iss/sub      |
| when.key                                                     | request.auth.audiences   | aud          |
| when.key                                                     | request.auth.presenter   | azp          |
| when.key                                                     | request.auth.claims[key] | JWT 全部属性 |

### 参考

1. [基于OIDC实现istio来源身份验证](https://www.jianshu.com/p/8f98098b7599)
2. [JWT 授权](https://www.servicemesher.com/istio-handbook/practice/jwt-authorization.html)
3. [JWT Claims](https://tools.ietf.org/html/rfc7519#section-4)

## 使用deny action的授权策略

本节将展示如何授权istio授权策略来拒绝istio网格中的HTTP流量。

### 部署

部署sleep应用并校验连通性：

```shell
$ kubectl create ns foo
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
$ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
```

```shell
# kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
200
```

### 明确拒绝一个请求

1. 下面命令会为`foo`命名空间中的`httpbin`负载创建`deny-method-get`授权策略。策略会对满足`rules`字段中的条件的请求执行`DENY` `action`。下面例子会拒绝使用`GET`方法的请求

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: deny-method-get
     namespace: foo
   spec:
     selector:
       matchLabels:
         app: httpbin
     action: DENY
     rules:
     - to:
       - operation:
           methods: ["GET"]
   EOF
   ```

2. 校验GET请求被拒绝：

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/get" -X GET -s -o /dev/null -w "%{http_code}\n"
   403
   ```

3. 校验允许`POST`请求

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/post" -X POST -s -o /dev/null -w "%{http_code}\n"
   200
   ```

4. 更新`deny-method-get`授权策略，只有当HTTP首部`x-token`的值不为`admin`时才会拒绝`GET`请求。下面例子将`notValues`字段设置为`["admin"]`来拒绝首部字段不为`admin`的请求。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: deny-method-get
     namespace: foo
   spec:
     selector:
       matchLabels:
         app: httpbin
     action: DENY
     rules:
     - to:
       - operation:
           methods: ["GET"]
       when:
       - key: request.headers[x-token]
         notValues: ["admin"]
   EOF
   ```

5. 校验当HTTP首部字段为`x-token: admin`时是允许的

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/get" -X GET -H "x-token: admin" -s -o /dev/null -w "%{http_code}\n"
   200
   ```

6. 校验当HTTP首部字段为`x-token: guest`时是拒绝的

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/get" -X GET -H "x-token: guest" -s -o /dev/null -w "%{http_code}\n"
   403
   ```

7. 下面命令会创建`allow-path-ip`授权策略来允许请求通过`/ip`路径访问`httpbin`负载。通过在`action`字段设置`ALLOW`实现

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: allow-path-ip
     namespace: foo
   spec:
     selector:
       matchLabels:
         app: httpbin
     action: ALLOW
     rules:
     - to:
       - operation:
           paths: ["/ip"]
   EOF
   ```

8. 校验HTTP首部为`x-token: guest`且路径为`/ip`的GET请求仍然被`deny-method-get`策略拒绝

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -X GET -H "x-token: guest" -s -o /dev/null -w "%{http_code}\n"
   403
   ```

9. 校验HTTP首部为`x-token: admin `且路径为`/ip`的GET请求仍然被`allow-path-ip`策略允许

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -X GET -H "x-token: admin" -s -o /dev/null -w "%{http_code}\n"
   200
   ```

10. 校验HTTP首部为`x-token: admin `且路径为`/get`的GET请求仍然被拒绝，因为不匹配 `allow-path-ip`策略

    ```shell
    # kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl "http://httpbin.foo:8000/get" -X GET -H "x-token: admin" -s -o /dev/null -w "%{http_code}\n"
    403
    ```

### 总结

本节主要展示了授权中的`action`字段的用法：`DENY`总是优先于`ALLOW`，且`ALLOW`中的规则的关系是`AND`，而`DENY`中的规则的关系是`OR` 。

### 卸载

```shell
$ kubectl delete namespace foo
```

## 授权ingress Gateway

本节展示如何在istio ingress网关上使用授权策略配置访问控制。

istio授权策略支持基于IP的allow/deny列表，以及先前由Mixer策略支持的基于属性的allow/deny列表。Mixer策略已经在1.5版本中被废弃，不建议生产使用。

### 部署

在`foo`命名空间中部署一个`httpbin`负载，使用istio ingress网关暴露该服务：

```shell
$ kubectl create ns foo
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin-gateway.yaml) -n foo
```

通过设置`externalTrafficPolicy：local`来更新ingress网关，使用以下命令在ingress网关上保留原始客户端的源IP。更多参见 [Source IP for Services with `Type=NodePort`](https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-nodeport)

> 使用`externalTrafficPolicy：local`时，kube-proxy仅会将请求代理到本地endpoint，不会跨节点。如果没有匹配的endpoint，则该会丢弃该报文，此时会保留报文的原始源IP地址(跨节点会使用SNAT将报文原始源IP地址修改为本节点地址)。

```shell
$ kubectl patch svc istio-ingressgateway -n istio-system -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

获取`INGRESS_HOST` 和 `INGRESS_PORT`

```shell
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
$ export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
```

校验可以通过ingress网关访问httbin负载

```shell
# curl "$INGRESS_HOST":"$INGRESS_PORT"/headers -s -o /dev/null -w "%{http_code}\n"
200
```

使用如下命令的输出来保证ingress网关接收到了原始客户端的源IP地址，该地址将会在授权策略中使用：

```shell
# CLIENT_IP=$(curl "$INGRESS_HOST":"$INGRESS_PORT"/ip -s | grep "origin" | cut -d'"' -f 4) && echo "$CLIENT_IP"
172.20.127.78
```

### 基于IP的allow列表和deny列表

1. 下面命令会为istio ingress网关创建授权策略`ingress-policy`。下面策略中的`action`字段为`ALLOW`，允许ipBlocks字段指定的IP地址访问ingress网关。不在该列表中的IP地址会被拒绝。ipBlocks支持单IP地址和CIDR：

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: ingress-policy
     namespace: istio-system
   spec:
     selector:
       matchLabels:
         app: istio-ingressgateway
     action: ALLOW
     rules:
     - from:
       - source:
          ipBlocks: ["1.2.3.4", "5.6.7.0/24"] #允许访问网关的源地址IP列表
   EOF
   ```

2. 校验到ingress网关的请求被拒绝了

   ```shell
   # curl "$INGRESS_HOST":"$INGRESS_PORT"/headers -s -o /dev/null -w "%{http_code}\n"
   403
   ```

3. 更新`ingress-policy`，在ALLOW IP地址列表中包含客户端的IP地址

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: ingress-policy
     namespace: istio-system
   spec:
     selector:
       matchLabels:
         app: istio-ingressgateway
     action: ALLOW
     rules:
     - from:
       - source:
          ipBlocks: ["1.2.3.4", "5.6.7.0/24", "$CLIENT_IP"]
   EOF
   ```

4. 校验到ingress网关的请求变为允许

   ```shell
   # curl "$INGRESS_HOST":"$INGRESS_PORT"/headers -s -o /dev/null -w "%{http_code}\n"
   200
   ```

5. 更新 `ingress-policy`授权策略，将`action`设置为`DENY`，将客户端地址设置到`ipBlocks`字段中，此时对ingress的源地址为`$CLIENT_IP`的访问将会被拒绝：

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: ingress-policy
     namespace: istio-system
   spec:
     selector:
       matchLabels:
         app: istio-ingressgateway
     action: DENY
     rules:
     - from:
       - source:
          ipBlocks: ["$CLIENT_IP"]
   EOF
   ```

6. 校验到ingress网关的请求被拒绝

   ```shell
   # curl "$INGRESS_HOST":"$INGRESS_PORT"/headers -s -o /dev/null -w "%{http_code}\n"
   403
   ```

### 卸载

```shell
$ kubectl delete namespace foo
$ kubectl delete authorizationpolicy ingress-policy -n istio-system
```

### 总结

从上面可以看出，在用法上，对ingress网关的授权策略和对其他istio网关内部的服务的授权策略并没有什么不同。

## 授权策略信任域迁移

本节展示如何在不修改授权策略的前提下进行信任域的迁移。

在istio 1.4中，引入了一个alpha特性来支持对授权策略的信任域的迁移，即如果一个istio网格需要改变其信任域时，则不需要手动修改授权策略。在istio中，如果一个负载运行在`foo`命名空间中，使用的service account为`bar`，系统的信任域为`my-td`，则负载的身份标识为 `spiffe://my-td/ns/foo/sa/bar`。默认情况下，istio网格的信任域为`cluster.local`(除非在安装时指定了其他域)。

### 部署

1. 使用用户信任域安装istio，并启用mutual TLS

   ```shell
   # istioctl install -f cni-annotations.yaml --set values.global.istioNamespace=istio-system --set values.gateways.istio-egressgateway.enabled=true  --set meshConfig.accessLogFile="/dev/stdout"  --set values.global.trustDomain=old-td
   ```

2. 在`default`命名空间中部署httpbin，并在`default`和`sleep-allow`命名空间中部署sleep

   ```shell
   $ kubectl label namespace default istio-injection=enabled
   $ kubectl apply -f samples/httpbin/httpbin.yaml
   $ kubectl apply -f samples/sleep/sleep.yaml
   $ kubectl create namespace sleep-allow
   $ kubectl label namespace sleep-allow istio-injection=enabled
   $ kubectl apply -f samples/sleep/sleep.yaml -n sleep-allow
   ```

3. 配置如下授权策略，拒绝除`sleep-allow`命名空间中的`sleep`外的对`httpbin`的请求。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: service-httpbin.default.svc.cluster.local
     namespace: default
   spec:
     rules:
     - from:
       - source:
           principals:
           - old-td/ns/sleep-allow/sa/sleep #只有sleep-allow命名空间中的sleep才能访问httpbin服务
       to:
       - operation:
           methods:
           - GET
     selector:
       matchLabels:
         app: httpbin
   ---
   EOF
   ```

校验`default`中的`sleep`到`httpbin`的请求，该请求被拒绝

```shell
# kubectl exec "$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -- curl http://httpbin.default:8000/ip -s -o /dev/null -w "%{http_code}\n"
403
```

校验`sleep-allow`中的`sleep`到`httpbin`的请求，该请求被允许

```shell
# kubectl exec "$(kubectl -n sleep-allow get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -n sleep-allow -- curl http://httpbin.default:8000/ip -s -o /dev/null -w "%{http_code}\n"
200
```

### 不使用信任域别名迁移信任域

1. 使用新的信任域安装istio，现在istio网格的信任域为`new-td`

   ```shell
   # istioctl install -f cni-annotations.yaml --set values.global.istioNamespace=istio-system --set values.gateways.istio-egressgateway.enabled=true  --set meshConfig.accessLogFile="/dev/stdout"  --set values.global.trustDomain=new-td
   ```

2. 重新部署httpbin和sleep，使其接收来自新的istio控制面的变更

   ```shell
   $ kubectl delete pod --all
   $ kubectl delete pod --all -n sleep-allow
   ```

3. 校验`default`和`sleep-allow`命名空间的`sleep`到`httpbin`的请求都被拒绝了

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -- curl http://httpbin.default:8000/ip -s -o /dev/null -w "%{http_code}\n"
   403
   # kubectl exec "$(kubectl -n sleep-allow get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -n sleep-allow -- curl http://httpbin.default:8000/ip -s -o /dev/null -w "%{http_code}\n"
   403
   ```

   这是因为在授权策略中拒绝除使用`old-td/ns/sleep-allow/sa/sleep`身份标识的所有请求，即`sleep-allow`命名空间中的`sleep`应用使用的老标识。当迁移到一个新的信任域`new-td`之后，该sleep应用的标识变为了`new-td/ns/sleep-allow/sa/sleep`，与授权策略不匹配。因此`sleep-allow`命名空间中的`sleep`到`httpbin`就被拒绝了。在istio 1.4之前需要手动修改授权策略来使之正常工作，现在有了更加方便的方式。

   迁移信任域，不使用信任域别名

### 使用信任域别名迁移信任域

1. 使用新的信任域和信任域别名安装istio

   ```yaml
   # cat cni-annotations.yaml
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   spec:
     components:
       cni:
         enabled: true
         namespace: kube-system
     values:
       meshConfig:
         trustDomain: new-td
         trustDomainAliases:
           - old-td
         certificates:
           - secretName: dns.example1-service-account
             dnsNames: [example1.istio-system.svc, example1.istio-system]
           - secretName: dns.example2-service-account
             dnsNames: [example2.istio-system.svc, example2.istio-system]
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
   ```

2. 不修改任何授权策略，校验到`httpbin`的请求

   default命名空间中的sleep到httpbin的请求被拒绝

   ```shell
   # kubectl exec "$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -- curl http://httpbin.default:8000/ip -s -o /dev/null -w "%{http_code}\n"
   403
   ```

   sleep-allow命名空间中的sleep到httpbin的请求被允许

   ```shell
   # kubectl exec "$(kubectl -n sleep-allow get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -n sleep-allow -- curl http://httpbin.default:8000/ip -s -o /dev/null -w "%{http_code}\n"
   200
   ```

### 最佳实践

从istio 1.4开始，当编写授权策略时，需要使用`cluster.local`作为策略的信任域，例如`cluster.local/ns/sleep-allow/sa/sleep`。注意，上面情况下，`cluster.local`并不是istio网格的信任域(信任域为`old-td`)。但是在授权策略中，`cluster.local`是一个指向当前信任域的指针，即`old-td` (或后面的 `new-td`)。通过在授权策略中使用`cluster.local`，当迁移到一个新的信任域时，istio会探测并将新的信任域与就的信任域一视同仁，而无需使用别名。

> 按照上面的说法，将创建的授权策略的principals字段修改为`cluster.local/ns/sleep-allow/sa/sleep`，重新测试连通性，可以得到与使用别名相同的结果。

### 卸载

```shell
$ kubectl delete authorizationpolicy service-httpbin.default.svc.cluster.local
$ kubectl delete deploy httpbin; kubectl delete service httpbin; kubectl delete serviceaccount httpbin
$ kubectl delete deploy sleep; kubectl delete service sleep; kubectl delete serviceaccount sleep
$ kubectl delete namespace sleep-allow
$ istioctl manifest generate --set profile=demo -f td-installation.yaml | kubectl delete --ignore-not-found=true -f -
$ rm ./td-installation.yaml
```