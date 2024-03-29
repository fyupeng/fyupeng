---
title: SpringBoot整合rpc-netty-framework
top: false
cover: false
toc: true
mathjax: false
date: 2022-10-20 17:15:08
author:
img:
coverImg:
password:
summary: SpringBoot如何整合RPC框架，解决端口占用问题、注入动态启动、服务远程依赖问题
tags:
- RPC
categories:
- Java笔记
---
一个分布式微服务RPC框架 | [返回](README.CN.md)

## 使用效果：

1. 用户访问客户端：GET http://localhost:8081/user/hello?name="张三来访"

![image-20221020170500139](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/image-20221020170500139.png)

1. 浏览器访问客户端：

![image-20221020170622580](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/image-20221020170622580.png)

服务端接收情况：

![image-20221020170428236](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/image-20221020170428236.png)

服务端负载注册服务：

![image-20221020170833644](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/image-20221020170833644.png)

上面的实现就好比客户端只拿到服务端的api接口，加上配置中心地址即可调用远程服务！

## 1. 创建工程

创建两个工程，一个作为服务端`SpringBoot`、一个作为客户端`SpringBoot`，同时作为后端接口服务

创建Maven工程的时候推荐使用父子工程依赖，而且要注意子模块之间的相互依赖关系，其中：

父模块（`root`项目）：负责管理`SpringBoot`版本、统一版本、`JDK`版本、日志依赖

```xml
<profiles>
    <profile>
        <id>jdk1.8</id>
        <activation>
            <activeByDefault>true</activeByDefault>
            <jdk>1.8</jdk>
        </activation>
        <properties>
            <maven-compiler-source>1.8</maven-compiler-source>
            <maven-compiler-target>1.8</maven-compiler-target>
            <maven-copiler-compilerVersion>1.8</maven-copiler-compilerVersion>
        </properties>
    </profile>
</profiles>

<parent>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>2.1.2.RELEASE</version>
<relativePath/>
</parent>
<dependencies>
<!-- 与 logbakc 整合 -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.10</version>
</dependency>
<!-- 日志框架 -->
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.3</version>
</dependency>
</dependencies>
```

主模块（启动类所在模块）应配置`maven`打包插件，`SpringBootStarterWeb`，`rpc-core`和依赖`Service`/`Controller`模块，客户端只有`Service`模块，服务端只有`Controller`模块

```xml
<dependencies>

    <dependency>
        <groupId>cn.fyupeng</groupId>
        <artifactId>springboot-rpc-service</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>

    <dependency>
        <groupId>cn.fyupeng</groupId>
        <artifactId>rpc-core</artifactId>
        <version>2.0.0</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<build>
<plugins>
    <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
</plugins>
</build>
```

`Service`/`Controller`模块：依赖`api`模块和`common`模块，客户端请求服务端调用服务，当然没有`Service`模块，让主模块依赖`Controller`模块，`Controller`模块还要与主模块一样依赖`SpringBootStarterWeb`和`rpc-core`

```xml
<dependencies>
    <dependency>
        <groupId>cn.fyupeng</groupId>
        <artifactId>springboot-rpc-common</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>cn.fyupeng</groupId>
        <artifactId>springboot-rpc-api</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
</dependencies>
```

`Common`模块，依赖`rpc-common`模块

```xml
<dependencies>
    <dependency>
        <groupId>cn.fyupeng</groupId>
        <artifactId>rpc-common</artifactId>
        <version>2.0.0</version>
    </dependency>
</dependencies>
```

项目在准备测试服务端的自动发现服务功能时，要保证`cn.fyupeng.@Service`注解类能够被扫描，使用`cn.fyupeng.util.ReflectUtil`类即可。

```java
public class Test {
    public static void main(String[] args) throws IOException {
        Set<Class<?>> classes = ReflectUtil.getClasses("cn.fyupeng");
        for (Class<?> aClass : classes) {
            System.out.println(aClass);
        }
    }
}
```

## 2. 客户端

### 2.1 编写启动器

新建`cn.fyupeng`包，包下新建启动器类

```java
@SpringBootApplication
@ComponentScan(basePackages = {"cn.fyupeng","org.utils"})
public class SpringBootClientStarter {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootClientStarter.class, args);
    }
}
```

### 2.2 编写配置文件

```properties
# 单机模式
cn.fyupeng.nacos.register-addr=192.168.10.1:8848
# 集群模式
cn.fyupeng.nacos.cluster.use=true
cn.fyupeng.nacos.cluster.load-balancer=round
cn.fyupeng.nacos.cluster.nodes=192.168.10.1:8847|192.168.10.1:8848;192.168.10.1:8849
```

### 2.3 编写自定义配置类

### 2.4 编写api

注意与客户端包名完全相同

```java
package cn.fyupeng.service;

/**
 * @Auther: fyp
 * @Date: 2022/10/19
 * @Description: HelloWorld接口
 * @Package: cn.fyupeng.cn.fyupeng.controller
 * @Version: 1.0
 */
public interface HelloService {
    String sayHello(String name);
}
```

### 2.5 编写控制器

`@PostConstruct`注解不要与`@Autowire`公用，因为`@Autowire`是本地依赖的，而我`@PostConstruct`会在该变量使用前调用，不过需要自行去实现，我的实现是远程依赖。

而`@Reference`没有依赖注入的功能，只有在超时重试才需要标记上！

`@PostContruct`与`@Autowire`在`51cto`博客中有所讲解，请自行到我的`github`主页`get`

```java
package cn.fyupeng.controller;

import cn.fyupeng.anotion.Reference;
import cn.fyupeng.loadbalancer.RandomLoadBalancer;
import cn.fyupeng.net.netty.client.NettyClient;
import cn.fyupeng.proxy.RpcClientProxy;
import cn.fyupeng.serializer.CommonSerializer;
import cn.fyupeng.service.HelloService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.utils.JSONResult;

import javax.annotation.PostConstruct;


/**
 * @Auther: fyp
 * @Date: 2022/10/19
 * @Description: HelloWorld控制器
 * @Package: cn.fyupeng.cn.fyupeng.controller
 * @Version: 1.0
 */

@RequestMapping("/user")
@RestController
public class HelloController {

    //@Autowired
    //private HelloService helloService;

    private static final RandomLoadBalancer randomLoadBalancer = new RandomLoadBalancer();
    private static final NettyClient nettyClient = new NettyClient(randomLoadBalancer, CommonSerializer.KRYO_SERIALIZER);
    private static final RpcClientProxy rpcClientProxy = new RpcClientProxy(nettyClient);

    @Reference(retries = 5, timeout = 600, asyncTime = 3000)
    //@Autowired
    private HelloService helloService;

    @PostConstruct
    public void init() {
        helloService = rpcClientProxy.getProxy(HelloService.class, HelloController.class);
    }

    @GetMapping("/hello")
    public JSONResult sayHello(String name) {
        return JSONResult.ok(helloService.sayHello(name));
    }

}

```

## 3. 服务端

### 3.1 编写配置类

新建`cn.fyupeng.config`包，在包下新建资源配置类，用于注入绑定端口

```java
@Configuration
@ConfigurationProperties(prefix="cn.fyupeng.config")
//不使用默认配置文件application.properties和application.yml
@PropertySource("classpath:resource.properties")
public class ResourceConfig {
    private int serverPort;

    public int getServerPort() {
        return serverPort;
    }

    public void setServerPort(int serverPort) {
        this.serverPort = serverPort;
    }
}
```

### 3.2 编写启动器

```java
@Slf4j
@ServiceScan
@SpringBootApplication
@ComponentScan(basePackages = {"cn.fyupeng", "org.utils"})
public class RpcServerStarter implements CommandLineRunner {

    @Autowired
    private ResourceConfig resourceConfig;

    @PostConstruct
    public void init() {
        Map<String, String> resourceLoaders = ResourceLoadUtils.load("resource.properties");
        if (resourceLoaders != null) {
            String serverPort = resourceLoaders.get("cn.fyupeng.config.serverPort");
            resourceConfig.setServerPort(Integer.parseInt(serverPort));
        }
    }

    public static void main(String[] args) {
        SpringApplication.run(RpcServerStarter.class,args);
    }

    @Override
    public void run(String... args) throws Exception {
        //这里也可以添加一些业务处理方法，比如一些初始化参数等
        while(true){
            NettyServer nettyServer = null;
            try {
                nettyServer = new NettyServer("192.168.2.185", resourceConfig.getServerPort(), SerializerCode.KRYO.getCode());
            } catch (RpcException e) {
                e.printStackTrace();
            }
            log.info("Service bind in port with "+ resourceConfig.getServerPort() +" and start successfully!");
            nettyServer.start();
            log.error("RegisterAndLoginService is died，Service is restarting....");
        }
    }

}
```

### 3.3 编写配置文件

注意`config/resource.properties`与资源目录下的`resource.properties`不能同时公用，前者优先级高于后者

```properties
# 用于启动 jar 包端口
cn.fyupeng.config.serverPort=8082

# 用于配置中心单机
cn.fyupeng.nacos.register-addr=192.168.2.185:8848
# 用于配置中心集群
cn.fyupeng.nacos.cluster.use=true
cn.fyupeng.nacos.cluster.load-balancer=round
cn.fyupeng.nacos.cluster.nodes=192.168.2.185:8847|192.168.2.185:8848;192.168.2.185:8849
```

### 3.4 编写api

注意与客户端包名完全相同

```java
package cn.fyupeng.service;

/**
 * @Auther: fyp
 * @Date: 2022/10/19
 * @Description: HelloWorld接口
 * @Package: cn.fyupeng.cn.fyupeng.controller
 * @Version: 1.0
 */
public interface HelloService {
    String sayHello(String name);
}
```

### 3.5 编写业务

注意`Service`注解为`cn.fyupeng.service.HelloService`

```java
package cn.fyupeng.service.impl;


import cn.fyupeng.anotion.Service;
import cn.fyupeng.service.HelloService;

/**
 * @Auther: fyp
 * @Date: 2022/10/19
 * @Description: HelloWorld实现类
 * @Package: cn.fyupeng.cn.fyupeng.controller.impl
 * @Version: 1.0
 */

@Service
public class HelloServiceImpl implements HelloService {
    @Override
    public String sayHello(String name) {
        return "hello, my name is " + name;
    }
}

```



为了使用配置文件注入来启动服务对应的端口