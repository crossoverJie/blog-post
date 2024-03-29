---
title: 模板方法实践
date: 2022/12/27 08:08:08
categories: 
- 设计模式
tags: 
- Go
- Java
---

![](https://s2.loli.net/2023/01/12/TG3RQnjKDOc7vaF.png)

# 前言

最近不出意外的阳了，加上刚入职新公司不久，所以也没怎么更新；这两天好些后分享一篇前段时间的一个案例：

最近在设计一个对某个中间件的测试方案，这个测试方案需要包含不同的测试逻辑，但相同的是需要对各个环节进行记录；比如统计耗时、调用通知 API 等相同的逻辑。

如果每个测试都单独写这些逻辑那无疑是做了许多重复工作了。

<!--more-->

基于以上的特征很容易能想到**模板方法**这个设计模式。

这是一种有上层定义框架，下层提供不同实现的设计模式。

比如装修房子的时候业主可以按照自己的喜好对不同的房间进行装修，但是整体的户型图不能做修改，比如承重墙是肯定不能打的。

而这些固定好的条条框框就是上层框架给的约束，下层不同的实现就有业主自己决定；所以对于整栋楼来说框架都是固定好的，让业主在有限的范围内自由发挥也方便物业的管理。

# 具体实现

以我这个案例的背景为例，首先需要定义出上层框架：

## Java

`Event` 接口：

```java
public interface Event {

    /**
     * 新增一个任务
     */
    void addJob();

    /**
     * 单个任务执行完毕
     *
     * @param jobName    任务名称
     * @param finishCost 任务完成耗时
     */
    void finishOne(String jobName, String finishCost);

    /**单个任务执行异常
     * @param jobDefine 任务
     * @param e 异常
     */
    void oneException(AbstractJobDefine jobDefine, Exception e);

    /**
     * 所有任务执行完毕
     */
    void finishAll();
}
```

```java
    public void start() {
        event.addJob();
        try {
            CompletableFuture.runAsync(() -> {
                StopWatch watch = new StopWatch();
                try {
                    watch.start(jobName);
                    // 不同的子业务实现
                    run(client);
                } catch (Exception e) {
                    event.oneException(this, e);
                } finally {
                    watch.stop();
                    event.finishOne(jobName, StrUtil.format("cost: {}s", watch.getTotalTimeSeconds()));
                }
            }, TestCase.EXECUTOR).get(timeout, TimeUnit.SECONDS);
        } catch (Exception e) {
            event.oneException(this, e);
        }
    }

    /** Run busy code
     * @param client
     * @throws Exception e
     */
    public abstract void run(Client client) throws Exception;    
```

其中最核心的就是 run 函数，它是一个抽象函数，具体实现交由子类完成；这样不同的测试用例之间也互不干扰，同时整体的流程完全相同：
- 记录任务数量
- 统计耗时
- 异常记录

等流程。

----

接下来看看如何使用：

```java
        AbstractJobDefine job1 = new Test1(event, "测试1", client, 10);
        CompletableFuture<Void> c1 = CompletableFuture.runAsync(job1::start, EXECUTOR);

        AbstractJobDefine job2 = new Test2(event, "测试2", client, 10);
        CompletableFuture<Void> c2 = CompletableFuture.runAsync(job2::start, EXECUTOR);

        AbstractJobDefine job3 = new Test3(event, "测试3", client, 20);
        CompletableFuture<Void> c3 = CompletableFuture.runAsync(job3::start, EXECUTOR);

        CompletableFuture<Void> all = CompletableFuture.allOf(c1, c2, c3);
        all.whenComplete((___, __) -> {
            event.finishAll();
            client.close();
        }).get();
```

显而易见 `Test1~3` 都继承了 `AbstractJobDefine` 同时实现了其中的 `run` 函数，使用的时候只需要创建不同的实例等待他们都执行完成即可。

以前在 Java 中也有不同的应用：
![](https://s2.loli.net/2023/01/12/dRl4DEIXj1BfNZ2.png)

[https://crossoverjie.top/2019/03/01/algorithm/consistent-hash/?highlight=%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95#%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95](https://crossoverjie.top/2019/03/01/algorithm/consistent-hash/?highlight=%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95#%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95)

## Go

同样的示例用 Go 自然也可以实现：

![](https://s2.loli.net/2023/01/12/Eu6OUrb7jGtLozN.png)

```go
func TestJobDefine_start(t *testing.T) {
	event := NewEvent()
	j1 := &JobDefine{
		Event:   event,
		Run:     &run1{},
		JobName: "job1",
		Param1:  "p1",
		Param2:  "p2",
	}
	j2 := &JobDefine{
		Event:   event,
		Run:     &run2{},
		JobName: "job2",
		Param1:  "p11",
		Param2:  "p22",
	}
	j1.Start()
	j2.Start()
	for _, ch := range event.GetChan() {
		<-ch
	}
	log.Println("finish all")

}

func (r *run2) Run(param1, param2 string) error {
	log.Printf("run3 param1:%s, param2:%s", param1, param2)
	time.Sleep(time.Second * 3)
	return errors.New("test err")
}

func (r *run1) Run(param1, param2 string) error {
	log.Printf("run1 param1:%s, param2:%s", param1, param2)
	return nil
}
```

使用起来也与 Java 类似，创建不同的实例；最后等待所有的任务执行完毕。

# 总结

设计模式往往是对某些共性能力的抽象，但也没有一个设计模式可以适用于所有的场景；需要对不同的需求选择不同的设计模式。

至于在工作中如何进行正确的选择，那就需要自己日常的积累了；比如多去了解不同的设计模式对于的场景，或者多去阅读优秀的代码，Java 中的 `InputStream/Reader/Writer` 这类 IO 相关的类都有具体的应用。

