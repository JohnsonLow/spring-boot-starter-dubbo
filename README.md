#spring-boot-start-dubbo

spring-boot-start-dubbo，让你可以使用spring-boot的方式开发dubbo程序。使dubbo开发变得如此简单。
> 该项目是由teaey的项目[spring-boot-starter-dubbo](https://github.com/teaey/spring-boot-starter-dubbo)重构而成。 如果有何指正，请联系我 272472809@qq.com

## 如何使用

### 1. clone 代码

```shell
git clone git@github.com:JohnsonLow/spring-boot-starter-dubbo.git
```

### 2. 编译安装（暂时未发布。目前未发现bug）

```
cd spring-boot-starter-dubbo
mvn clean install
```


### 3. 修改maven配置文件(可以参考样例[spring-boot-starter-dubbo-sample](https://github.com/JohnsonLow/spring-boot-starter-dubbo-sample))

##### 在Spring Boot项目的pom.xml修改依赖的版本及编码:
```xml
<properties>
    <java.version>1.7</java.version>
    <spring-boot.version>1.5.1.RELEASE</spring-boot.version>
    <version.compiler-plugin>3.5.1</version.compiler-plugin>
    <version.source-plugin>2.2.1</version.source-plugin>
    <version.javadoc-plugin>2.9.1</version.javadoc-plugin>
    <version.maven-gpg-plugin>1.5</version.maven-gpg-plugin>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
</properties>

 ```
#### 该项目分为了两个模块：
* starter-dubbo-common 该模块主要功能为扫描dubbo组件 
* starter-dubbo-server 依赖 starter-common 便于注册到zookeeper

##### maven插件用于打包成可执行的jar文件,添加以下插件(这里一定要加载需要打包成jar的mudule的pom中)
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```
### 4. 服务接口
```java
package com.lyf.samples.api;

public interface HelloService {
    String code(String device);
}

```
### 5. 发布服务

##### 在pom.xml中添加以下依赖:

根据实际情况依赖最新版本

```xml
 <dependency>
     <groupId>com.oqiji.springboot</groupId>
     <artifactId>starter-dubbo-server</artifactId>
     <version>1.0.0</version>
 </dependency>
 ```

在application.yml添加Dubbo的版本信息和客户端超时信息,如下:
```yml
spring:
  dubbo:
    application:
      name: provider
    registry:
      address: zookeeper://127.0.0.1:20181
    protocol:
      name: dubbo
      prot: 20880

  datasource:
      url: jdbc:mysql://127.0.0.1:3306/qiji?useUnicode=true&characterEncoding=UTF8
      username: qiji
      password: qiji,123
debug: true
```
> spring.dubbo.scan 为要扫描的包。也可使用@DubboScan进行配置

* 编写你的Dubbo服务,只需要添加要发布的服务实现上添加 @Service ,如下
```java
  package com.lyf.ds;
  
  import com.alibaba.dubbo.config.annotation.Service;
  import com.lyf.samples.api.HelloService;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.jdbc.core.JdbcTemplate;
  import org.springframework.util.CollectionUtils;
  import java.util.List;
  
  @Service(version = "1.0")
  public class HelloServiceProvider implements HelloService {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    @Override
    public String code(String device) {
      List<String> res =  jdbcTemplate.queryForList("SELECT code FROM activity WHERE device = ? ", String.class, device);
      if(CollectionUtils.isEmpty(res)){
        return null;
      }
      return res.get(0);
    }
  }

```

* spring boot启动
```java
  package com.lyf.ds;
  
  import com.oqiji.boot.dubbo.DubboScan;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  
  /**
   * Created by 永锋 on 2017/2/22.
   */
  @SpringBootApplication
  @DubboScan("com.lyf.ds")
  public class Server {
    public static void main(String[] args) {
      SpringApplication.run(Server.class, args);
    }
  }
```

### 6. 消费Dubbo服务
* 在application.yml添加Dubbo的版本信息和客户端超时信息,如下:
```yml
spring:
  dubbo:
    application:
      name: consumer
    registry:
      address: zookeeper://127.0.0.1:20181
#    scan: com.lyf.dc
debug: true

```

##### 在pom.xml中添加以下依赖:

根据实际情况依赖最新版本

```xml
 <dependency>
    <groupId>com.oqiji.springboot</groupId>
    <artifactId>starter-dubbo-common</artifactId>
    <version>1.0.0</version>
 </dependency>
 ```
* 引用Dubbo服务,只需要添加要发布的服务实现上添加 @Reference ,如下:
```java
  package com.lyf.dc;
  
  import com.alibaba.dubbo.config.annotation.Reference;
  import com.lyf.samples.api.HelloService;
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  import org.springframework.stereotype.Component;
  
  @Component
  public class HelloServiceReq {
    private static Logger logger = LoggerFactory.getLogger(HelloServiceReq.class);
  
    @Reference(version = "1.0")
    public HelloService helloService;
  
    public void hello() {
      long start = System.currentTimeMillis();
      for (int i = 0; i < 10000; i++) {
        String result = helloService.code("bbk_bbk72_t_jb3_jzusjfbi65wwqosk" + i);
        logger.info("{}. result is {}", i, result);
      }
      logger.info("total spent {} ms", System.currentTimeMillis() - start);
    }
  }

```
* spring boot启动
```java
  package com.lyf.dct;
  
  import com.lyf.dc.HelloServiceReq;
  import com.oqiji.boot.dubbo.DubboScan;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.context.ConfigurableApplicationContext;
  
  @SpringBootApplication
  @DubboScan("com.lyf.dc")
  public class Client {
    public static void main(String[] args) {
      ConfigurableApplicationContext context = SpringApplication.run(Client.class, args);
      HelloServiceReq helloService = context.getBean(HelloServiceReq.class);
      helloService.hello();
    }
  }

```


### 7. 打包

> 可以直接执行Server或者Client启动

> 可以通过mvn clean package 打包成可执行的jar文件
