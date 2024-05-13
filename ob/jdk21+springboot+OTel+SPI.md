---
title: 从一个 JDK21+OpenTelemetry 不兼容的问题讲起
date: 2024/05/13 23:31:40
categories:
  - OB
  - OpenTelemetry
tags:
---

# 背景

前段时间公司领导让我排查一个关于在 JDK21 环境中使用 Spring Boot 配合一个 JDK18 新增的一个 SPI(`java.net.spi.InetAddressResolverProvider`) 不生效的问题。

但这个不生效的前置条件有点多：
- JDK 的版本得在 18+
- SpringBoot3.x
- 还在额外再配合使用 `-javaagent:opentelemetry-javaagent.jar` 使用，也就是 OpenTelemetry 提供的 agent。

才会导致自定义的 `InetAddressResolverProvider` 无法正常工作。

<!--more-->

---

在复现这个问题之前先简单介绍下 `java.net.spi.InetAddressResolverProvider` 这个 SPI；它是在 JDK18 之后才提供的，在这之前我们使用 `InetAddress` 的内置解析器来解析主机名和 IP 地址，但这个解析器之前是不可以自定义的。

在某些场景下会不太方便，比如我们需要请求 `order.service` 这个域名时希望可以请求到某一个具体 IP 地址上，我们可以自己配置 host ，或者使用服务发现机制来实现。

但现在通过 `InetAddressResolverProvider` 就可以定义在请求这个域名的时候返回一个我们预期的 IP 地址。

同时由于它是一个 SPI，所以我们只需要编写一个第三方包，任何项目依赖它之后在发起网络请求时都会按照我们预期的 IP 进行请求。
# 复现

要使用它也很简单，主要是两个类：
- `InetAddressResolverProvider`：这是一个抽象类，我们可以继承它之后重写它的 get 函数返回一个 `InetAddressResolver` 对象
- `InetAddressResolver`：一个接口，主要提供了两个函数；一个用于传入域名返回 IP 地址，另一个反之：传入 IP 地址返回域名。

```java

public class MyAddressResolverProvider extends InetAddressResolverProvider {
    @Override
    public InetAddressResolver get(Configuration configuration) {
        return new MyAddressResolver();
    }
    @Override
    public String name() {
        return "MyAddressResolverProvider Internet Address Resolver Provider";
    }
}

public class MyAddressResolver implements InetAddressResolver {

    public MyAddressResolver() {
        System.out.println("=====MyAddressResolver");
    }

    @Override
    public Stream<InetAddress> lookupByName(String host, LookupPolicy lookupPolicy)
            throws UnknownHostException {
        if (host.equals("fedora")) {
            return Stream.of(InetAddress.getByAddress(new byte[] {127, 127, 10, 1}));
        }
        return Stream.of(InetAddress.getByAddress(new byte[] {127, 0, 0, 1}));
    }
    @Override
    public String lookupByAddress(byte[] addr) {
        System.out.println("++++++" + addr[0] + " " + addr[1] + " " + addr[2] + " " + addr[3]);
        return  "fedora";
    }
}

---

```java
addresses = InetAddress.getAllByName("fedora");
// output: 127 127 10 1
```
这里我简单实现了一个对域名 fedora 的解析，会直接返回 `127.127.10.1`。

如果使用 IP 地址进行查询时：
```java
InetAddress byAddress = InetAddress.getByAddress(new byte[]{127, 127, 10, 1});

System.out.println("+++++" + byAddress.getHostName());
// output: fedora
```

当然要要使得这个 SPI 生效的前提条件是我们需要新建一个文件：
`META-INF/services/java.net.spi.InetAddressResolverProvider`
里面的内容是我们自定义类的全限定名称：
```
com.example.demo.MyAddressResolverProvider
```

这样一个完整的 SPI 就实现完成了。

---

正常情况下我们将应用打包为一个 jar 之后运行：
```shell
java -jar target/demo-0.0.1-SNAPSHOT.jar
```
是可以看到输出结果是符合预期的。

一旦我们使用配合上 spring boot 打包之后，也就是加上以下的依赖：

```xml
<parent>  
  <groupId>org.springframework.boot</groupId>  
  <artifactId>spring-boot-starter-parent</artifactId>  
  <version>3.2.3</version>  
  <relativePath/> <!-- lookup parent from repository -->  
</parent>

<build>  
  <plugins>  
   <plugin>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-maven-plugin</artifactId>  
   </plugin>  
  </plugins>  
</build>
```
再次执行其实也没啥问题，也能按照预期输出结果。

但我们加上 OpenTelemetry 的 agent 时：
```shell
java  -javaagent:opentelemetry-javaagent.jar \
      -jar target/demo-0.0.1-SNAPSHOT.jar
```

就会发现在执行解析的时候抛出了 `java.net.UnknownHostException`异常。

![](https://s2.loli.net/2024/04/08/owZLIF7yzUpSdjn.png)
从结果来看就是没有进入我们自定义的解析器。

# SPI 原理

在讲排查过程之前还是要先预习下关于 Java SPI 的原理以及应用场景。

以前写过一个 http 框架 [cicada](https://github.com/TogetherOS/cicada)，其中有一个可拔插 IOC 容器的功能：

> 就是可以自定义实现自己的 IOC 容器，将自己实现的 IOC 容器打包为一个第三方包加入到依赖中，cicada 框架就会自动使用自定义的 IOC 实现。


要实现这个功能本质上就是要定义一个接口，然后根据依赖的不同实现创建接口的实例对象。

```java
public interface CicadaBeanFactory {

    /**
     * Register into bean Factory
     * @param object
     */
    void register(Object object);

    /**
     * Get bean from bean Factory
     * @param name
     * @return
     * @throws Exception
     */
    Object getBean(String name) throws Exception;

    /**
     * get bean by class type
     * @param clazz
     * @param <T>
     * @return bean
     * @throws Exception
     */
    <T> T getBean(Class<T> clazz) throws Exception;

    /**
     * release all beans
     */
    void releaseBean() ;
}
```

获取具体的示例代码时就只需要使用 JDK 内置的 `ServiceLoader` 进行加载即可：

```java
public static CicadaBeanFactory getCicadaBeanFactory() {  
    ServiceLoader<CicadaBeanFactory> cicadaBeanFactories = ServiceLoader.load(CicadaBeanFactory.class);  
    if (cicadaBeanFactories.iterator().hasNext()){  
        return cicadaBeanFactories.iterator().next() ;  
    }  
    return new CicadaDefaultBean();  
}
```

代码也非常的简洁，和刚才提到的 `InetAddressResolverProvider` 一样我们需要新增一个 `META-INF/services/top.crossoverjie.cicada.base.bean.CicadaBeanFactory` 文件来配置我们的类名称。

```java
private boolean hasNextService() {
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
        	// PREFIX = META-INF/services/
            String fullName = PREFIX + service.getName();
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        pending = parse(service, configs.nextElement());
    }
    nextName = pending.next();
    return true;
}
```

在 ServiceLoader 类中会会去查找 `META-INF/services` 的文件，然后解析其中的内容从而反射生成对应的接口对象。

这里还有一个关键是通常我们的代码都会打包为一个 JAR 包，类加载器需要加载这个  JAR 包，同时需要在这个 JAR 包里找到我们之前定义的那个 spi 文件，如果这里查不到文件那就认为没有定义 SPI。

这个是本次问题的重点，会在后文分析原因的时候用到。

# 排查
因为问题就出现在是否使用 opentelemetry-javaagent.jar 上，所以我需要知道在使用了 agent 之后有什么区别。

从刚才的对 SPI 的原理分析，加上 agent 出现异常，说明理论上就是没有读取到我们配置的文件: `java.net.spi.InetAddressResolverProvider`。

于是我便开始 debug，在 ServiceLoader 加载 jar 包的时候是可以看到具体使用的是什么 `classLoader` 。

这是不配置 agent 的时候使用的 classLoader：
![](https://s2.loli.net/2024/04/10/kgR1hOzKbnGMJUA.png)
使用这个 loader 是可以通过文件路径在 jar 包中查找到我们配置的文件。


而配置上 agent 之后使用的 classLoader:
![](https://s2.loli.net/2024/04/10/45sUKGr6xeVPNXA.png)
却是一个 JarLoader，这样是无法加载到在 springboot 格式下的配置文件的，至于为什么加载不到，那就要提一下 maven 打包后的文件目录和 spring boot 打包后的文件目录的区别了。

![](https://s2.loli.net/2024/04/10/ZtDCc7SvXFHmL9J.png)
这里我截图了同样的一份代码不同的打包方式：
上面的是传统 maven，下图是 spring boot；其实主要的区别就是在 pom 中使用了一个构建插件：

```xml
<build>  
  <plugins>  
   <plugin>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-maven-plugin</artifactId>  
   </plugin>  
  </plugins>  
</build>
```

> 或者使用 `spring-boot` 命令再次打包的效果也是一样的。

会发现 spring boot 打包后会多出一层 `BOOT-INF` 的文件夹，然后会在 `MANIFIST.MF` 文件中定义 `Main-Class` 和 `Start-Class`.

---
通过上面的 debug 其实会发现 JarLoader 只能在加载 maven 打包后的文件，也就是说无法识别 BOOT-INF 这个目录。

正常情况下 spring boot 中会有一个额外的 `java.nio.file.spi.FileSystemProvider` 实现:
![](https://s2.loli.net/2024/04/10/iFus4tA1KXEMYkq.png)
通过这个类的实现可以直接从 JAR 包中加载资源，比如我们自定义的 SPI 资源等。

初步判断使用 `opentelemetry-javaagent.jar`的 agent 之后，它的类加载器优先于了 spring boot ，从而导致后续的加载失败。

## 远程 debug
这里穿插几个 debug 小技巧，其中一个是远程 debug，因为这里我是需要调试 javaagent，正常情况下是无法直接 debug 的。

所以我们可以使用以下命令启动应用：

```shell
java -agentlib:jdwp="transport=dt_socket,server=y,suspend=y,address=5000" -javaagent:opentelemetry-javaagent.jar \
      -jar target/demo-0.0.1-SNAPSHOT.jar
```
![](https://s2.loli.net/2024/04/10/D2z4krNyHAanSlC.png)

然后在 idea 中配置一个 remote 启动。
> 注意这里的端口得和命令行中的保持一致。

当应用启动之后便可以在 idea 中启动这个 remote 了，这样便可以正常 debug 了。
## 条件断点
第二个是条件断点也非常有用，有时候我们需要调试一个公共函数，调用的地方非常多。

而我们只需要关心某一类行为的调用，此时就可以对这个函数中的变量进行判断，当他们满足某些条件时再进入断点，这样可以极大的提高我们的调试效率：
![](https://s2.loli.net/2024/04/10/L9PkNyZprCql6Wd.png)

配置也很简单，只需要在断点上右键就可以编辑条件了。

# 社区咨询

虽然我根据现象初步可以猜测下原因，但依然不确定如何调整才能解决这个问题，于是便去社区提了一个 [issue](https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/10921)。

![](https://s2.loli.net/2024/04/10/YHiIOfvxu1EUMpj.png)
最后在社区大佬的帮助下发现我们需要禁用掉 OpenTelemetry agent 中的一个 resource 就可以了。

![](https://s2.loli.net/2024/04/10/EiX3mD9k6cwjMUf.png)
这个 resource 是由 agent 触发的，它优先于 spring boot 之前进行 SPI 的加载。
目的是为了给 metric 和 trace 新增两个属性：
![](https://s2.loli.net/2024/04/10/I39iXt4JfdwVn8S.png)

![](https://s2.loli.net/2024/04/10/bH3wfUeCk4K9PJ5.png)
加载的核心代码在这里，只要禁用掉之后就不会再加载了。


禁用前：
![](https://s2.loli.net/2024/04/10/7ZIo2VaqesFXL53.png)

禁用后：
![](https://s2.loli.net/2024/04/10/k2yQBPxzMHFENjd.png)

当我们禁用掉之后就不会存在这两个属性了，不过我们目前并没有使用这两个属性，所以为了使得 SPI 生效就只有先禁用掉了，后续再看看社区还有没有其他的方案。

想要复现 debug 的可以在这里尝试：
[https://github.com/crossoverJie/demo](https://github.com/crossoverJie/demo)




参考连接：
- https://github.com/TogetherOS/cicada
- https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/htmlsingle/#packaging.repackage-goal
- https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/10921
- https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/resources/library/README.md#host
