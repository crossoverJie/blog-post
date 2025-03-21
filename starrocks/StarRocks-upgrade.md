---
title: StarRocks 升级注意事项
date: 2025/03/14 17:16:35
categories:
  - OB
tags:
  - StarRocks
---
前段时间升级了生产环境的 `StarRocks`，从 3.3.3 升级到了 3.3.9，期间还是踩了不少坑所以在这里记录下。

![](https://s2.loli.net/2025/03/17/uGyo2HULqzQ1j7W.png)

 因为我们的集群使用的是存算分离的版本，也是使用官方提供的 operator 部署在 kubernetes 里的，所以没法按照官方的流程进入虚拟机手动启停对应的服务。

只能使用 operator 提供的方案手动修改对应组件的镜像版本，后续的升级操作交给 operator 去完成。

<!--more-->

![](https://s2.loli.net/2025/03/17/7YwRaKzbPo4dAEs.png)

理论上这个升级流程没什么问题，修改镜像版本之后只需要安静等待他滚动更新即可。
# 元数据备份与恢复

但考虑到之前在社区看到有存算分离集群升级失败导致数据丢失的案例，我们的全量业务已经切换到 StarRocks，如果数据丢失那需要花几天时间进行数据同步，这在业务上是无法接受的，所以我们最好是可以在升级前备份数据，即便是升级失败数据依然还在。

![](https://s2.loli.net/2025/03/17/g1q7NbX6H9t5oce.png)

原本官方社区是有提供数据备份与恢复能力的，但是我们使用的存算分离集群不支持😂，而想要获得社区版的支持应该还要等一段时间，即便是支持了我们升级到那个版本依然是需要备份的。

![](https://s2.loli.net/2025/03/17/l6Dfcs4JYV52pQE.png)

> 好消息，在最新的 3.4.1 版本中已经支持了快照备份了，只是作为一个新 feature，稳定性还有待观察。

所以我们的计划是在当前这个版本（3.3.3）能否自己备份数据，由于我们是存算分离的版本，所以数据主要分为两部分：
- 存储在所有 FE 节点里的 meta 元数据
- 存储在云存储里的业务数据

备份的时候自然就需要备份这两部分的数据。


## 备份元数据

在元数据里存放了所有的数据库、表、视图等信息，具体在磁盘的结构如下：

```shell
|-- bdb
|   |-- 00000000.jdb
|   |-- je.config.csv
|   |-- je.info.0
|   |-- je.info.0.lck
|   |-- je.lck
|   `-- je.stat.csv
|-- image
|   |-- ROLE
|   |-- VERSION
|   |-- image.327375
|   |-- starmgr
|   |   `-- image.390
|   `-- v2
|       |-- checksum.327375
|       `-- image.327375
```

bdb 目录主要是用于 leader 选举的，理论上并不需要备份，真正需要的是 `image` 目录下的 `image.327375` 等元数据文件。

![](https://s2.loli.net/2025/03/17/KpVCBqJGjXQctah.png)

![](https://s2.loli.net/2025/03/17/CMAiILF3ZHaouY6.png)

里面是用 JSON 存储的各种类型的元数据，FE 在启动的时候会读取该文件，然后根据不同的类型取不同的偏移量读取其中的元数据加载到内存里。

我们的 FE 一共有三个节点，需要找到其中的 leader 节点（理论上只需要备份 leader 节点即可，其他节点会在 leader 启动后同步过去），直接将这个 meta 目录备份到本地即可：

在开始之前需要停掉所有的写入任务，暂停所有的物化视图刷新。

```sql

# inactive 所有的物化视图
SELECT CONCAT('ALTER MATERIALIZED VIEW ', TABLE_NAME, ' INACTIVE;') FROM information_schema.materialized_views;

# 手动创建镜像
ALTER SYSTEM CREATE IMAGE;

# 找到 leader 节点
SHOW FRONTENDS;
```

然后进入 leader 节点备份元数据：
```shell
k exec -it kube-starrocks-fe-0-n sr -- bash

tar -zcvf meta.tar.gz meta/

# 下载备份元数据到本地
k cp starrocks-fe-0:/opt/starrocks/fe/meta/image.tar.gz image.tar.gz -n starrocks -c fe --retries=5
```

## 备份云存储数据

云存储的备份就需要结合你使用的云厂商来备份了，通常他们都有提供对应的备份能力。

要注意的是我们再备份的时候需要记录在存储桶里的目录名称，之后还原的时候名称得保持一致才行。
## 恢复元数据

当出现极端情况升级失败的时候，我们需要把元数据覆盖回去；但由于我们的应用运行在容器里，不可以在应用启动之后再替换元数据。

只能在应用启动之前将之前备份的元数据覆盖回去，这里可以使用 kubernetes 中的 `initContainers` 提前将数据复制到应用容器里。

在开始之前我们需要先把备份的元数据打包为一个镜像。

```dockerfile
FROM busybox  
ADD meta.tar.gz /temp
```

然后我们需要手动修改 FE 的 `statefulset` 的资源，创建一个 initContainers。

```yaml
initContainers:  
  - name: copy-file-init  
    image: meta:0.0.1  
    command: ["/bin/sh", "-c"]  
    args: ["rm -rf /meta-target/* && cp -r /temp/meta/. /meta-target"]  
    volumeMounts:  
      - name: fe-meta  
        mountPath: "/meta-target"
```

原理就是在 initContainers 中挂载原本 FE 的元数据目录，这样就可以直接将之前备份的元数据覆盖过去。
> 当然也可以直接使用 k8s 的 go client 用代码的方式来修改，会更容易维护。

还原的时候需要先将云存储里的数据先还原之后再还原元数据。
# 物化视图刷新策略


真正升级的时候倒是没有碰到升级失败的情况，所以没有走恢复流程；但是却碰到了一个更麻烦的事情。
## 物化视图作为基表


我们在升级前将所有的物化视图设置为了 `INACTIVE`，升级成功后需要将他们都改为 `ACTIVE`。

第一个问题是如果某个物化视图 `MV1` 的基表也是一个物化视图 `MV-base`，这样会导致 `MV1` 的全量刷新。

我之前在这个 [PR](https://github.com/StarRocks/starrocks/pull/50926) 里新增了一个参数：`excluded_refresh_tables` 可以用于排除基表发生变化的时候刷新物化视图，但是忘记了基表也是物化视图的场景。

![](https://s2.loli.net/2025/03/18/nf3QioRgc96ze2p.png)


所以在这个 [PR](https://github.com/StarRocks/starrocks/pull/56428) 中修复了该问题，现在基表是物化视图的时候也可以使用了。

## 物化视图手动 ACTIVE

前面提到在升级之前需要将所有的物化视图设置为 `INACTIVE`，升级成功后再手动设置为 ACTIVE。

我们在手动 ACTIVE 之后发现这些物化视图又在做全量刷新了，于是我们检查了代码。

![](https://s2.loli.net/2025/03/18/DMNQsxH5ZqaKpFb.png)

发现在使用 `ALTER MATERIALIZED VIEW order_mv ACTIVE;` 修改视图状态的时候会强制刷新物化视图的所有分区。

![](https://s2.loli.net/2025/03/18/GP6Bzl7Zxq29FoD.png)

> force: true 的时候会直接跳过基表的分区检查，导致分区的全量刷新。


![](https://s2.loli.net/2025/03/18/PG8WVKrpfoTz61I.png)

同时会在 ACTIVE 的时候将视图基表的 `baseTableVisibleVersionMap` 版本号缓存清空，FE 需要在刷新的时候判断当前需要刷新的分区是否存在与缓存中，如果存在的话说明不需要刷新，现在被清空后就一定会被刷新。

所以我提了一个 PR 可以在 `ACTIVE` 物化视图的时候人工判断是否需要刷新:
```sql
alter materialized view mv_test1 ACTIVE WITH NO_VALIDATION
```

这样带上 `NO_VALIDATION` 参数后就 `force=false` 也就不会全量刷新了。

如果在 ACTIVE 物化视图的时候碰到类似场景，可以在这个 `PR` 发布之后加上 `NO_VALIDATION` 来跳过刷新。

参考链接：
- https://github.com/StarRocks/starrocks/pull/50926
- https://github.com/StarRocks/starrocks/pull/56428
- https://github.com/StarRocks/starrocks/pull/56864