---
title: k8s-服务网格实战-配置 Mesh（灰度发布）
date: 2023/11/07 22:30:46
categories:
  - k8s
tags:
- Istio
---
![istio-02.png](https://s2.loli.net/2023/11/07/rmwdGK6TQDuhAEW.png)


在上一篇 [k8s-服务网格实战-入门Istio](https://crossoverjie.top/2023/10/31/ob/k8s-Istio01/)中分享了如何安装部署 `Istio`，同时可以利用 `Istio` 实现 `gRPC` 的负载均衡。
<!--more-->
今天我们更进一步，深入了解使用 Istio 的功能。
![image.png](https://s2.loli.net/2023/11/07/jKIeEH7ir9uqDUd.png)
从 Istio 的流量模型中可以看出：Istio 支持管理集群的出入口请求（gateway），同时也支持管理集群内的 mesh 流量，也就是集群内服务之间的请求。

本次先讲解集群内部的请求，配合实现以下两个功能：
- 灰度发布（对指定的请求分别路由到不同的 service 中）
- 配置 service 的请求权重

## 灰度发布
在开始之前会部署两个 `deployment` 和一个 `service`，同时这两个 `deployment` 所关联的 `Pod` 分别对应了两个不同的 `label`，由于在灰度的时候进行分组。
![image.png](https://s2.loli.net/2023/11/07/tLOYQiNg5HEe2ry.png)

使用这个 `yaml` 会部署所需要的 `deployment` 和 `service`。
```bash
kubectl apply -f https://raw.githubusercontent.com/crossoverJie/k8s-combat/main/deployment/istio-mesh.yaml 
```

---
首先设想下什么情况下我们需要灰度发布，一般是某个重大功能的测试，只对部分进入内测的用户开放入口。

假设我们做的是一个 `App`，我们可以对拿到了内测包用户的所有请求头中加入一个版本号。

比如 `version=200` 表示新版本，`version=100` 表示老版本。
同时在服务端会将这个版本号打印出来，用于区分请求是否进入了预期的 Pod。

```go
// Client 
version := r.URL.Query().Get("version")  
name := "world"  
ctx, cancel := context.WithTimeout(context.Background(), time.Second)  
md := metadata.New(map[string]string{  
    "version": version,  
})  
ctx = metadata.NewOutgoingContext(ctx, md)  
defer cancel()  
g, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})

// Server
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {  
    md, ok := metadata.FromIncomingContext(ctx)  
    var version string  
    if ok {  
       version = md.Get("version")[0]  
    }    log.Printf("Received: %v, version: %s", in.GetName(), version)  
    name, _ := os.Hostname()  
    return &pb.HelloReply{Message: fmt.Sprintf("hostname:%s, in:%s, version:%s", name, in.Name, version)}, nil  
}
```

### 对 service 分组
进行灰度测试时往往需要新增部署一个灰度服务，这里我们称为 v2（也就是上图中的 Pod2）。

同时需要将 v1 和 v2 分组：

```yaml
apiVersion: networking.istio.io/v1alpha3  
kind: DestinationRule  
metadata:  
  name: k8s-combat-service-ds  
spec:  
  host: k8s-combat-service-istio-mesh  
  subsets:  
    - name: v1  
      labels:  
        app: k8s-combat-service-v1  
    - name: v2  
      labels:  
        app: k8s-combat-service-v2
```
这里我们使用 Istio 的 `DestinationRule` 定义 `subset`，也就是将我们的 `service` 下的 Pod 分为 v1/v2。

> 使用 标签 `app` 进行分组

注意这里的 `host: k8s-combat-service-istio-mesh` 通常配置的是 `service` 名称。

```yaml
apiVersion: v1  
kind: Service  
metadata:  
  name: k8s-combat-service-istio-mesh  
spec:  
  selector:  
    appId: "12345"  
  type: ClusterIP  
  ports:  
    - port: 8081  
      targetPort: 8081  
      name: app  
    - name: grpc  
      port: 50051  
      targetPort: 50051
```
也就是这里 service 的名称，同时也支持配置为 `host: k8s-combat-service-istio-mesh.default.svc.cluster.local`，如果使用的简写`Istio` 会根据当前指定的 `namespace` 进行解析。
> Istio 更推荐使用全限定名替代我们这里的简写，从而避免误操作。

当然我们也可以在 `DestinationRule` 中配置负载均衡的策略，这里我们先略过：

```yaml
apiVersion: networking.istio.io/v1alpha3  
kind: DestinationRule  
metadata:  
  name: k8s-combat-service-ds  
spec:  
  host: k8s-combat-service-istio-mesh 
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN  
```
![image.png](https://s2.loli.net/2023/11/07/TJyEV6eIiCcapSH.png)


---
这样我们就定义好了两个分组：
- v1：app: k8s-combat-service-v1
- v2：app: k8s-combat-service-v2

之后就可以配置路由规则将流量分别指定到两个不同的组中，这里我们使用 `VirtualService` 进行配置。

```yaml
apiVersion: networking.istio.io/v1alpha3  
kind: VirtualService  
metadata:  
  name: k8s-combat-service-vs  
spec:  
  gateways:  
    - mesh  
  hosts:  
    - k8s-combat-service-istio-mesh # match this host
http:  
  - name: v1  
    match:  
      - headers:  
          version:  
            exact: '100'  
    route:  
      - destination:  
          host: k8s-combat-service-istio-mesh  
          subset: v1  
  - name: v2  
    match:  
      - headers:  
          version:  
            exact: '200'  
    route:  
      - destination:  
          host: k8s-combat-service-istio-mesh  
          subset: v2  
  - name: default  
    route:  
      - destination:  
          host: k8s-combat-service-istio-mesh  
          subset: v1
```

这个规则很简单，会检测 http 协议的 `header` 中的 `version` 字段值，如果为 100 这路由到 `subset=v1` 这个分组的 Pod 中，同理为 200 时则路由到 `subset=v2` 这个分组的 Pod 中。

当没有匹配到 header 时则进入默认的 `subset:v1`

>  `gRPC` 也是基于 http 协议，它的 `metadata` 也就对应了 `http` 协议中的 `header`。


### header=100
```bash
Greeting: hostname:k8s-combat-service-v1-5b998dc8c8-hkb72, in:world, version:100istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=100"
Greeting: hostname:k8s-combat-service-v1-5b998dc8c8-hkb72, in:world, version:100istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=100"
Greeting: hostname:k8s-combat-service-v1-5b998dc8c8-hkb72, in:world, version:100istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=100"
Greeting: hostname:k8s-combat-service-v1-5b998dc8c8-hkb72, in:world, version:100istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=100"
```

### header=200
```bash
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
```

根据以上的上面的测试请求来看，只要我们请求头里带上指定的 `version` 就会被路由到指定的 `Pod` 中。

利用这个特性我们就可以在灰度验证的时候单独发一个灰度版本的 `Deployment`，同时配合客户端指定版本就可以实现灰度功能了。

## 配置权重

同样基于 `VirtualService` 我们还可以对不同的 `subset` 分组进行权重配置。

```yaml
apiVersion: networking.istio.io/v1alpha3  
kind: VirtualService  
metadata:  
  name: k8s-combat-service-vs  
spec:  
  gateways:  
    - mesh  
  hosts:  
    - k8s-combat-service-istio-mesh # match this host  
  http:  
    - match:  
        - uri:  
            exact: /helloworld.Greeter/SayHello  
      route:  
        - destination:  
            host: k8s-combat-service-istio-mesh  
            subset: v1  
          weight: 10  
        - destination:  
            host: k8s-combat-service-istio-mesh  
            subset: v2  
          weight: 90  
      timeout: 5000ms
```

这里演示的是针对 `SayHello` 接口进行权重配置（当然还有多种匹配规则），90% 的流量会进入 v2 这个 subset，也就是在 `k8s-combat-service-istio-mesh` service 下的 `app: k8s-combat-service-v2` Pod。

```bash
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-**v1**-5b998dc8c8-hkb72, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$ curl "http://127.0.0.1:8081/grpc_client?name=k8s-combat-service-istio-mesh&version=200"
Greeting: hostname:k8s-combat-service-v2-5db566fb76-xj7j6, in:world, version:200istio-proxy@k8s-combat-service-v1-5b998dc8c8-hkb72:/$
```
经过测试会发现大部分的请求都会按照我们的预期进入 v2 这个分组。

当然除之外之外我们还可以：
- 超时时间
- 故障注入
- 重试
具体的配置可以参考 [Istio](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMatchRequest) 官方文档：
![image.png](https://s2.loli.net/2023/11/07/LBjEtd1MP9VcAgl.png)
当然在一些云平台也提供了可视化的页面，可以更直观的使用。
![image.png](https://s2.loli.net/2023/11/07/2LVTgeiSK9HyxQJ.png)

> 以上是 阿里云的截图

但他们的管理的资源都偏 `kubernetes`，一般是由运维或者是 DevOps 来配置，不方便开发使用，所以还需要一个介于云厂商和开发者之间的管理发布平台，可以由开发者以项目维度管理维护这些功能。

本文的所有源码在这里可以访问：
[https://github.com/crossoverJie/k8s-combat](https://github.com/crossoverJie/k8s-combat)

#Blog #Istio 
