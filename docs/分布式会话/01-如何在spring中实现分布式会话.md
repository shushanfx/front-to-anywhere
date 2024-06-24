# 如何在 Spring 中实现分布式会话

Spring Boot 提供一套完善的解决方案，可以很方便的实现分布式会话。

Spring Boot + Redis 实现分布式会话是一个常用的搭配，下面我们将阐述如何接入。

## 怎么做？

### 1. 引入依赖

修改项目的`pom.xml`文件，引入如下依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

引入两个关键的依赖：

- spring-boot-starter-data-redis，表示使用 Redis 作为存储介质。

- spring-session-data-redis，包含启用使用 Redis 作为存储介质的配置。

### 2. 配置 Redis

在`application.yml`文件中配置 Redis 的连接信息：

```yaml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    database: 2
```

该步骤主要是为了配置 redis 链接属性。

### 3. 开启 Spring Session

在 Spring Boot 启动类上添加`@EnableRedisHttpSession`注解：

```java
@EnableRedisHttpSession
@SpringBootApplication
class Main {
    public static void main(String[] args) {
        SpringApplication.run(Main.class, args);
    }
}
```

`@EnableRedisHttpSession`注解表示开启 Spring Session 支持，Spring Boot 会自动配置 Spring Session。

通过上述三个步骤，我们就可以很方便的实现分布式会话。

## 小结

`Spring Boot` + `Redis` 实现分布式会话是一个常用的搭配，通过`Spring Boot`提供的`@EnableRedisHttpSession`注解，我们可以很方便的实现分布式会话。Spring 做了很多封装，让使用变得很简单。

那么，它们是怎么实现的呢？在接下里的章节中，我们将解析其原理。
