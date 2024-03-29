---
title: k8s 入门到实战--部署应用到 k8s
date: 2023/08/31 22:42:32
categories: 
- k8s
tags: 
- Go
---


![k8s 入门到实战 01.png](https://s2.loli.net/2023/09/04/ymUpcXZrxfNsT91.png)

# 背景

最近这这段时间更新了一些 k8s 相关的博客和视频，也收到了一些反馈；大概分为这几类：
- 公司已经经历过服务化改造了，但还未接触过云原生。
- 公司部分应用进行了云原生改造，但大部分工作是由基础架构和运维部门推动的，自己只是作为开发并不了解其中的细节，甚至 k8s 也接触不到。
- 还处于比较传统的以虚拟机部署的传统运维为主。

其中以第二种占大多数，虽然公司进行了云原生改造，但似乎和纯业务研发同学来说没有太大关系，自己工作也没有什么变化。

恰好我之前正好从业务研发的角度转换到了基础架构部门，两个角色我都接触过，也帮助过一些业务研发了解公司的云原生架构；

为此所以我想系统性的带大家以**研发**的角度对 k8s 进行实践。

因为 k8s 部分功能其实是偏运维的，对研发来说优先级并不太高；
所以我不太会涉及一些 k8s 运维的知识点，比如安装、组件等模块；主要以我们日常开发会使用到的组件讲起。

<!--more-->

# 计划

## 入门
- 部署应用到 k8s
- 跨服务调用
- 集群外部访问
## 进阶
- 如何使用配置
- 服务网格实战

## 运维你的应用
- 应用探针
- 滚动更新与回滚
- 优雅采集日志
- 应用可观测性
	- 指标可视化

## k8s 部署常见中间件
- helm 一键部署
- 编写 Operator 自动化应用生命周期

![image.png](https://s2.loli.net/2023/09/02/BtYcF6jp8u3nzJs.png)
这里我整理了一下目录，每个章节都有博客+视频配合观看，大家可以按照喜好选择。

因为还涉及到了视频，所以只能争取一周两更，在两个月内全部更新完毕。

根据我自己的经验，以上内容都掌握的话对 k8s 的掌握会更进一步。


# 部署应用到 k8s

首先从第一章【部署应用到 k8s】开始，我会用 Go 写一个简单的 Web 应用，然后打包为一个 Docker 镜像，之后部署到 k8s 中，并完成其中的接口调用。

## 编写应用

```go
func main() {  
   http.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {  
      log.Println("ping")  
      fmt.Fprint(w, "pong")  
   })  
  
   http.ListenAndServe(":8081", nil)  
}
```

 应用非常简单就是提供了一个 `ping`  接口，然后返回了一个 `pong`.

## Dockerfile

```dockerfile
# 第一阶段：编译 Go 程序  
FROM golang:1.19 AS dependencies  
ENV GOPROXY=https://goproxy.cn,direct  
WORKDIR /go/src/app  
COPY go.mod .  
#COPY ../../go.sum .  
RUN --mount=type=ssh go mod download  
  
# 第二阶段：构建可执行文件  
FROM golang:1.19 AS builder  
WORKDIR /go/src/app  
COPY . .  
#COPY --from=dependencies /go/pkg /go/pkg  
RUN go build  
  
# 第三阶段：部署  
FROM debian:stable-slim  
RUN apt-get update && apt-get install -y curl  
COPY --from=builder /go/src/app/k8s-combat /go/bin/k8s-combat  
ENV PATH="/go/bin:${PATH}"  
  
# 启动 Go 程序  
CMD ["k8s-combat"]
```

 之后编写了一个 `dockerfile` 用于构建 `docker` 镜像。

```makefile
docker:  
   @echo "Docker Build..."  
   docker build . -t crossoverjie/k8s-combat:v1 && docker image push crossoverjie/k8s-combat:v1
```

使用 `make docker`  会在本地构建镜像并上传到 `dockerhub`

## 编写 deployment
下一步便是整个过程中最重要的环节了，也是唯一和 k8s 打交道的地方，那就是编写 deployment。

<iframe src="//player.bilibili.com/player.html?aid=702346697&bvid=BV1Cm4y1n7yG&cid=1235124452&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

在之前的视频《一分钟了解 k8s》中讲过常见的组件：
![image.png](https://s2.loli.net/2023/09/04/hrOUSVsmP2KkNlC.png)

其中我们最常见的就是 deployment，通常用于部署无状态应用；现在还不太需要了解其他的组件，先看看 deployment 如何编写：
```yaml
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  labels:  
    app: k8s-combat  
  name: k8s-combat  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: k8s-combat  
  template:  
    metadata:  
      labels:  
        app: k8s-combat  
    spec:  
      containers:  
        - name: k8s-combat  
          image: crossoverjie/k8s-combat:v1  
          imagePullPolicy: Always  
          resources:  
            limits:  
              cpu: "1"  
              memory: 300Mi  
            requests:  
              cpu: "0.1"  
              memory: 30Mi
```

开头两行的 `apiVersion`  和 `kind` 可以暂时不要关注，就理解为 deployment 的固定写法即可。

metadata：顾名思义就是定义元数据的地方，告诉 `Pod` 我们这个 `deployment` 叫什么名字，这里定义为：`k8s-combat`

中间的：
```yaml
metadata:  
  labels:  
    app: k8s-combat
```

也很容易理解，就是给这个 `deployment` 打上标签，通常是将这个标签和其他的组件进行关联使用才有意义，不然就只是一个标签而已。
> 标签是键值对的格式，key, value 都可以自定义。

而这里的  `app: k8s-combat` 便是和下面的 spec 下的 selector 选择器匹配，表明都使用  `app: k8s-combat`  进行关联。

而 template 中所定义的标签也是为了让选择器和 template 中的定义的 Pod 进行关联。

> Pod 是 k8s 中相同功能容器的分组，一个 Pod 可以绑定多个容器，这里就只有我们应用容器一个了；后续在讲到 istio 和日志采集时便可以看到其他的容器。

template 中定义的内容就很容易理解了，指定了我们的容器拉取地址，以及所占用的资源(`cpu/ memory`)。

`replicas: 1`：表示只部署一个副本，也就是只有一个节点的意思。

## 部署应用

之后我们使用命令:

```shell
kubectl apply -f deployment/deployment.yaml
```

> 生产环境中往往会使用云厂商所提供的 k8s 环境，我们本地可以使用 [https://minikube.sigs.k8s.io/docs/start/](https://minikube.sigs.k8s.io/docs/start/) minikube 来模拟。

就会应用这个 deployment 同时将容器部署到 k8s 中，之后使用:
```shell
kubectl get pod
```
>  在后台 k8s 会根据我们填写的资源选择一个合适的节点，将当前这个 Pod 部署过去。

 就会列出我们刚才部署的 Pod:
```shell
❯ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
k8s-combat-57f794c59b-7k58n         1/1     Running   0          17h
```

我们使用命令：
```shell
kubectl exec -it k8s-combat-57f794c59b-7k58n  bash
```
就会进入我们的容器，这个和使用 docker 类似。

之后执行 curl 命令便可以访问我们的接口了：
```shell
root@k8s-combat-57f794c59b-7k58n:/# curl http://127.0.0.1:8081/ping
pong
root@k8s-combat-57f794c59b-7k58n:/#
```

这时候我们再开一个终端执行：
```
❯ kubectl logs -f k8s-combat-57f794c59b-7k58n
2023/09/03 09:28:07 ping
```
便可以打印容器中的日志，当然前提是应用的日志是写入到了标准输出中。

# 总结

以上就是这一章节的主要内容，重点就是将我们应用程序员打包为 docker 镜像后上传到镜像仓库，再配置好 deployment 由 k8s 进行调度运行。

下一章主要会涉及服务内部的调用，感兴趣的朋友可以先关注起来。

相关的源码和 yaml 资源文件都存在这里：
[https://github.com/crossoverJie/k8s-combat](https://github.com/crossoverJie/k8s-combat)

#Blog #K8s 