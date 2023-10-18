---
title: 使用 Helm 管理应用的一些 Tips
date: 2023/10/07 20:36:14
categories:
  - Helm
tags:
- CloudNative
---
![Helm tips.png](https://s2.loli.net/2023/10/08/9HKS1lNqyGMson5.png)

# 背景
`Helm` 是一个 **Kubernetes** 的包管理工具，有点类似于 `Mac` 上的 `brew`，`Python` 中的 `PIP`；可以很方便的帮我们直接在 `kubernetes` 中安装某个应用。

比如我们可以直接使用以下命令方便的在 k8s 集群安装和卸载 `MySQL`：
```bash
helm install my-sql oci://registry-1.docker.io/bitnamicharts/mysql -n mysql

helm uninstall my-mysql -n mysql
```

<!--more-->
对于一些复杂的应用使用 Helm 一键安装会更简单，以 Pulsar 举例：
![image.png](https://s2.loli.net/2023/10/08/ig4koZIFlUT5Bt1.png)
它有着多个组件，比如 bookkeeper、zookeeper、broker、proxy 等，各个组件还有着依赖关系。

如果我们手动安装流程会比较繁琐，而使用 Helm 时便非常简单：
```bash
helm repo add apache https://pulsar.apache.org/charts

helm install my-pulsar apache/pulsar --version 3.0.0 -n pulsar
```

> 当然他也只是帮我们生成了部署所需要的 yaml 文件，也没有太多黑科技。

# 升级

看似简单的工具我在实际线上使用的时候也踩过一个坑，最大的一个问题就是某次升级 Pulsar 的时候生成的 yaml 文件是空的，导致整个集群被删除了😭。

还好最后使用 `helm  rollback version` 将集群恢复过来了，我们的持久化数据也还在。

而出现这个问题的原因是我执行了下面这个命令：
```bash
helm upgrade pulsar ./charts/pulsar --version 2.9.2 -f charts/pulsar/values-2.10.3.yaml -n pulsar
```

我们是将 `pulsar` 的 `Helm-Chart` 源码下载到本地，然后修改 `value.yaml` 的方式执行升级的。

当时执行命令的时候没有注意，在一个没有 `values-2.10.3.yaml` 文件的目录下执行的，导致生成的 `yaml` 文件是空的，也就导致 k8s 在 `pulsar` 这个 `namespace` 下删除了所有的资源。

## 模拟升级
为了避免今后再次出现类似的问题，需要在升级前先模拟升级：
```shell
helm upgrade pulsar ./charts/pulsar --version 2.9.2 -f charts/pulsar/values-2.10.3.yaml -n pulsar --dry-run --debug > debug.yaml
```

其中关键的 `dry-run` 和 `debug` 参数可以指定模拟升级和输出详细的内容。

这样我们就可以在升级前先查看 `debug.yaml` 里的内容是不是符合我们的预期。
# 对比升级

但这样并不能直观的看出哪些地方是我们修改的，还好社区已经有了相关的插件，可以帮我们高亮显示修改的地方。

```shell
helm plugin install https://github.com/databus23/helm-diff
```
我们先安装好这个 helm 插件。

然后在升级前先使用该插件：
```shell
helm diff upgrade pulsar ./charts/pulsar --version 2.9.2 -f charts/pulsar/values-2.10.3.yaml -n pulsar
```

![image.png](https://s2.loli.net/2023/10/08/V1k5gdhLASfq9JR.png)

这样就可以高亮显示出修改的内容。

> 不用担心这个命令会直接升级，它会自动加上 --dry-run --debug 参数。

更多命令可以参考官方文档：
[https://github.com/databus23/helm-diff](https://github.com/databus23/helm-diff)

Helm 功能很强，在操作生产环境的时候必须得谨慎，都是血淋淋的教训啊。