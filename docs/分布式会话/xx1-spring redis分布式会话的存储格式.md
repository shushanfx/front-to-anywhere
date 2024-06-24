# Spring redis 分布式会话的存储格式

本文讨论 Spring Redis 分布式会话的存储格式。接下来我们将从一下几个部分进行展开：

- session key 的确定

- hash key 的确定

- hash value 的确定

我们先看 spring 在创建 redis template 时，代码如下：

```java
 protected RedisTemplate<String, Object> createRedisTemplate() {
  RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
  redisTemplate.setKeySerializer(new StringRedisSerializer());
  redisTemplate.setHashKeySerializer(new StringRedisSerializer());
  if (getDefaultRedisSerializer() != null) {
   redisTemplate.setDefaultSerializer(getDefaultRedisSerializer());
  }
  redisTemplate.setConnectionFactory(getRedisConnectionFactory());
  redisTemplate.setBeanClassLoader(this.classLoader);
  redisTemplate.afterPropertiesSet();
  return redisTemplate;
 }
```

它们定义了 redis 内容的存储形式。

## session key 的确定

当请求首次被处理时，不携带任何 session 信息，此时需要创建一个 session id，这个是后续请求识别 session 的唯一标识。然后，在写入到 redis 时，写入的 key 也不一致。因此，我们需要搞清楚两个问题：

1. session id 的生成规则

2. session redis key 的生成规则

### session id 的生成规则

我们依然来看其生成过程，从 SessionRepositoryFilter 入手，当 session 不存在时，会调用`createSession`方法，代码如下：

```java
public HttpSessionWrapper getSession(boolean create) {
  // 省略
  S session = SessionRepositoryFilter.this.sessionRepository.createSession();
  session.setLastAccessedTime(Instant.now());
  currentSession = new HttpSessionWrapper(session, getServletContext());
  setCurrentSession(currentSession);
  return currentSession;
}
```

进一步看`sessionRepository`.`createSession`方法，代入到 redis session，默认使用`RedisSessionRepository`，其实现如下：

```java
public class RedisSessionRepository {
  public RedisSession createSession() {
    MapSession cached = new MapSession();
    cached.setMaxInactiveInterval(this.defaultMaxInactiveInterval);
    RedisSession session = new RedisSession(cached, true);
    session.flushIfRequired();
    return session;
  }
}
```

我们进一步看`MapSession`的实现，代码如下：

```java
class MapSession {
  public MapSession() {
    this(generateId());
  }
  public MapSession(String id) {
    // 即 session id
    this.id = id;
    this.originalId = id;
  }
  private static String generateId() {
    return UUID.randomUUID().toString();
  }
}
```

因此，session id 的生成规则是`UUID.randomUUID().toString()`。

### session redis key 的生成规则

session id 生成之后，需要将 session 映射到 redis 中，这个映射关系是通过 redis key 来实现的。我们来看`RedisSessionRepository`的实现：

```java
class RedisSessionRepository {
  public void save(RedisSession session) {
    if (!session.isNew) {
    String key = getSessionKey(session.hasChangedSessionId() ? session.originalSessionId : session.getId());
    Boolean sessionExists = this.sessionRedisOperations.hasKey(key);
    if (sessionExists == null || !sessionExists) {
      throw new IllegalStateException("Session was invalidated");
    }
    }
    session.save();
 }
}
```

其中，`getSessionKey`方法的实现如下：

```java
 private String getSessionKey(String sessionId) {
  return this.keyNamespace + "sessions:" + sessionId;
 }
```

keyNamespace 是怎么确定的呢？其默认值为： `DEFAULT_KEY_NAMESPACE + ":"`。其中，`DEFAULT_KEY_NAMESPACE`的值为`"spring:session:"`。但是它可以通过配置文件进行修改，怎么修改的呢 ？

我们看`RedisHttpSessionConfiguration`的实现：

```java
public class RedisHttpSessionConfiguration {
  @Bean
  public RedisSessionRepository sessionRepository() {
    RedisTemplate<String, Object> redisTemplate = createRedisTemplate();
    RedisSessionRepository sessionRepository = new RedisSessionRepository(redisTemplate);
    sessionRepository.setDefaultMaxInactiveInterval(getMaxInactiveInterval());
    if (StringUtils.hasText(getRedisNamespace())) {
      sessionRepository.setRedisKeyNamespace(getRedisNamespace());
    }
    sessionRepository.setFlushMode(getFlushMode());
    sessionRepository.setSaveMode(getSaveMode());
    getSessionRepositoryCustomizers()
      .forEach((sessionRepositoryCustomizer) -> sessionRepositoryCustomizer.customize(sessionRepository));
    return sessionRepository;
 }
  protected String getRedisNamespace() {
    return this.redisNamespace;
  }
}
```

实际上来源于`RedisHttpSessionConfiguration`的 redisNamespace 属性。我们查找它的 setter 方法，发现在 setImportMetadata 方法中，代码如下：

```java
class RedisHttpSessionConfiguration {
 public void setImportMetadata(AnnotationMetadata importMetadata) {
  Map<String, Object> attributeMap = importMetadata
    .getAnnotationAttributes(EnableRedisHttpSession.class.getName());
  AnnotationAttributes attributes = AnnotationAttributes.fromMap(attributeMap);
  if (attributes == null) {
   return;
  }
  setMaxInactiveInterval(Duration.ofSeconds(attributes.<Integer>getNumber("maxInactiveIntervalInSeconds")));
  String redisNamespaceValue = attributes.getString("redisNamespace");
  if (StringUtils.hasText(redisNamespaceValue)) {
   setRedisNamespace(this.embeddedValueResolver.resolveStringValue(redisNamespaceValue));
  }
  setFlushMode(attributes.getEnum("flushMode"));
  setSaveMode(attributes.getEnum("saveMode"));
 }
}
```

说明它可以通过可以`importMetadata`获取，而`importMetadata`是在`@EnableRedisHttpSession`注解中获取的，代码如下：

```java
public @interface EnableRedisHttpSession {
  String redisNamespace() default RedisSessionRepository.DEFAULT_KEY_NAMESPACE;
}
```

因此，我们可以通过`@EnableRedisHttpSession`注解的`redisNamespace`属性来修改`keyNamespace`的值。我们再看`RedisHttpSessionConfiguration`.`sessionRepository`的实现：

```java
public class RedisHttpSessionConfiguration {
  @Bean
  public RedisSessionRepository sessionRepository() {
    RedisTemplate<String, Object> redisTemplate = createRedisTemplate();
    RedisSessionRepository sessionRepository = new RedisSessionRepository(redisTemplate);
    // ...
    if (StringUtils.hasText(getRedisNamespace())) {
      sessionRepository.setRedisKeyNamespace(getRedisNamespace());
    }
    // ...
    getSessionRepositoryCustomizers()
      .forEach((sessionRepositoryCustomizer) -> sessionRepositoryCustomizer.customize(sessionRepository));
    return sessionRepository;
 }
}
```

可以看到它还包括了`sessionRepositoryCustomizers`，这个是什么呢？我们看`RedisHttpSessionConfiguration`的`getSessionRepositoryCustomizers`方法：

```java
 @Autowired(required = false)
 public void setSessionRepositoryCustomizer(
   ObjectProvider<SessionRepositoryCustomizer<T>> sessionRepositoryCustomizers) {
  this.sessionRepositoryCustomizers = sessionRepositoryCustomizers.orderedStream().collect(Collectors.toList());
 }

 protected List<SessionRepositoryCustomizer<T>> getSessionRepositoryCustomizers() {
  return this.sessionRepositoryCustomizers;
 }
```

它是通过`@Autowired`注解获取的，因此我们可以通过 SessionRepositoryCustomizer 注解来修改`keyNamespace`的值。
