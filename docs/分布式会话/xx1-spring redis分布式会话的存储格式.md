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

它们定义了 redis 内容的存储形式。那么它的具体细节是怎样的呢 ？

## session key 的确定

当请求首次被处理时，不携带任何 session 信息，此时需要创建一个`session id`，它是后续请求识别 session 的唯一标识。然后，在写入到 redis 时，写入的 key 也不一致。因此，我们需要搞清楚两个问题：

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

上述代码使用了`MapSession`的无参构造函数。我们进一步看`MapSession`的实现，代码如下：

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

**因此，session id 的生成规则是`UUID.randomUUID().toString()`**。

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

keyNamespace 是怎么确定的呢？其默认值为： `DEFAULT_KEY_NAMESPACE + ":"`。其中，`DEFAULT_KEY_NAMESPACE`的值为`"spring:session:"`。因此默认的 key 为`spring:session:sessions:${sessionId}`。

### 如何修改默认的 keyNamespace

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

它是通过`@Autowired`注解注入的，因此我们可以通过新建`SessionRepositoryCustomizer`的相关 Bean 来修改`keyNamespace`的值。

SpringBoot 有没有提供已经实现的`SessionRepositoryCustomizer`呢？答案是有的。 我们看`RedisSessionConfiguration`的代码：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ RedisTemplate.class, RedisIndexedSessionRepository.class })
@ConditionalOnMissingBean(SessionRepository.class)
@ConditionalOnBean(RedisConnectionFactory.class)
@EnableConfigurationProperties(RedisSessionProperties.class)
class RedisSessionConfiguration {
 @Configuration(proxyBeanMethods = false)
 @ConditionalOnProperty(prefix = "spring.session.redis", name = "repository-type", havingValue = "default",
   matchIfMissing = true)
 @Import(RedisHttpSessionConfiguration.class)
 static class DefaultRedisSessionConfiguration {
  @Bean
  SessionRepositoryCustomizer<RedisSessionRepository> springBootSessionRepositoryCustomizer(
    SessionProperties sessionProperties, RedisSessionProperties redisSessionProperties,
    ServerProperties serverProperties) {
   // ...
   return (sessionRepository) -> {
    PropertyMapper map = PropertyMapper.get().alwaysApplyingWhenNonNull();
    map.from(sessionProperties
     .determineTimeout(() -> serverProperties.getServlet().getSession().getTimeout()))
     .to(sessionRepository::setDefaultMaxInactiveInterval);
    map.from(redisSessionProperties::getNamespace).to(sessionRepository::setRedisKeyNamespace);
    map.from(redisSessionProperties::getFlushMode).to(sessionRepository::setFlushMode);
    map.from(redisSessionProperties::getSaveMode).to(sessionRepository::setSaveMode);
   };
  }
 }
 @Configuration(proxyBeanMethods = false)
 @ConditionalOnProperty(prefix = "spring.session.redis", name = "repository-type", havingValue = "indexed")
 @Import(RedisIndexedHttpSessionConfiguration.class)
 static class IndexedRedisSessionConfiguration {
  @Bean
  SessionRepositoryCustomizer<RedisIndexedSessionRepository> springBootSessionRepositoryCustomizer(
    SessionProperties sessionProperties, RedisSessionProperties redisSessionProperties,
    ServerProperties serverProperties) {
   return (sessionRepository) -> {
    PropertyMapper map = PropertyMapper.get().alwaysApplyingWhenNonNull();
    map.from(sessionProperties
     .determineTimeout(() -> serverProperties.getServlet().getSession().getTimeout()))
     .to(sessionRepository::setDefaultMaxInactiveInterval);
    map.from(redisSessionProperties::getNamespace).to(sessionRepository::setRedisKeyNamespace);
    map.from(redisSessionProperties::getFlushMode).to(sessionRepository::setFlushMode);
    map.from(redisSessionProperties::getSaveMode).to(sessionRepository::setSaveMode);
    map.from(redisSessionProperties::getCleanupCron).to(sessionRepository::setCleanupCron);
   };
  }
 }
}
```

我们看到两个内部类，分别是`DefaultRedisSessionConfiguration`和`IndexedRedisSessionConfiguration`，它们都实现了`SessionRepositoryCustomizer`接口。`map.from(redisSessionProperties::getNamespace).to(sessionRepository::setRedisKeyNamespace);`这一行，即从`redisSessionProperties`复制对应的 sessionRepository。

那么 RedisSessionProperties 是怎么来的呢？我们看`RedisSessionConfiguration`的实现：

```java
@ConfigurationProperties(prefix = "spring.session.redis")
public class RedisSessionProperties {
  // ...
}
```

由此我们可以看出，`RedisSessionProperties`通过配置文件加载注入属性，前缀满足`spring.session.redis`的属性。因此，我们可以通过配置`spring.session.redis.redisKeyNamespace`属性来修改`keyNamespace`的值。

### 小结

通过上述描述，我们总结三种方法来控制`keyNamespace`：

- 通过`@EnableRedisHttpSession`注解的`redisNamespace`属性来修改`keyNamespace`的值；

- 通过增加`SessionRepositoryCustomizer`类型的 Bean 来实现修改；

- 通过配置文件修改，即`application.properties`或`application.yml`文件中配置`spring.session.redis.redisKeyNamespace`属性。

最终生成的 session key 为`keyNamespace + "sessions:" + sessionId`。`sessionId`按照`UUID.randomUUID().toString()`生成。

## hash key 的确定

redis 存储的 session 内容为一个 hash 值，key 使用上面介绍的 key。既然是一个 hash 值，那么它的 hash key 是怎么确定的呢 ？我们依然从其初始化脚本入手：

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

从字面意思分析使用`StringRedisSerializer`作为 hash key 的序列化器，实际上确实如此。 我们看`StringRedisSerializer`的实现：

```java
public class StringRedisSerializer implements RedisSerializer<String> {
 private final Charset charset;
 public StringRedisSerializer() {
  this(StandardCharsets.UTF_8);
 }
 public StringRedisSerializer(Charset charset) {
  Assert.notNull(charset, "Charset must not be null");
  this.charset = charset;
 }
 @Override
 public String deserialize(@Nullable byte[] bytes) {
  return (bytes == null ? null : new String(bytes, charset));
 }
 @Override
 public byte[] serialize(@Nullable String string) {
  return (string == null ? null : string.getBytes(charset));
 }
 @Override
 public Class<?> getTargetType() {
  return String.class;
 }
}
```

我们看它如何存储的？

```java
class AbstractOperations {
  public void putAll(K key, Map<? extends HK, ? extends HV> m) {
  if (m.isEmpty()) {
   return;
  }

  byte[] rawKey = rawKey(key);

  Map<byte[], byte[]> hashes = new LinkedHashMap<>(m.size());

  for (Map.Entry<? extends HK, ? extends HV> entry : m.entrySet()) {
   hashes.put(rawHashKey(entry.getKey()), rawHashValue(entry.getValue()));
  }

  execute(connection -> {
   connection.hMSet(rawKey, hashes);
   return null;
  });
 }
 <HK> byte[] rawHashKey(HK hashKey) {
  Assert.notNull(hashKey, "non null hash key required");
  if (hashKeySerializer() == null && hashKey instanceof byte[]) {
   return (byte[]) hashKey;
  }
  return hashKeySerializer().serialize(hashKey);
 }
  RedisSerializer hashKeySerializer() {
  return template.getHashKeySerializer();
 }
}
```

由此可见，它的 key 通过 template 的 hashKeySerializer 来序列化。代入到我们的案例中，使用了 StringRedisSerializer，因此 hash key 是通过`StringRedisSerializer`来序列化的。即，使用了 UTF-8 编码进行序列化。

那么存储了哪些内容呢？我们从`RedisSessionMapper`看出，包括如下：

- creationTime，创建时间

- lastAccessedTime，最后访问时间

- maxInactiveInterval，最大不活动时间

- 属性值，以`sessionAttr:`开头

## hash value 的确定

我们按照跟 hash key 一样的思路来看，hash value 使用

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

其使用了`DefaultRedisSerializer`作为 hash value 的序列化器。我们看默认的序列器是：

```java
class AbstractOperations {
  public void putAll(K key, Map<? extends HK, ? extends HV> m) {
  if (m.isEmpty()) {
   return;
  }

  byte[] rawKey = rawKey(key);

  Map<byte[], byte[]> hashes = new LinkedHashMap<>(m.size());

  for (Map.Entry<? extends HK, ? extends HV> entry : m.entrySet()) {
   hashes.put(rawHashKey(entry.getKey()), rawHashValue(entry.getValue()));
  }

  execute(connection -> {
   connection.hMSet(rawKey, hashes);
   return null;
  });
 }
 private byte[] rawValue(Object value) {
  if (valueSerializer == null && value instanceof byte[]) {
   return (byte[]) value;
  }
  return valueSerializer.serialize(value);
 }
}
```

其中 valueSerializer 默认值为 `defaultSerializer`， 即：

```java
  if (defaultSerializer == null) {
   defaultSerializer = new JdkSerializationRedisSerializer(
     classLoader != null ? classLoader : this.getClass().getClassLoader());
  }
```

因此默认的序列化器是`JdkSerializationRedisSerializer`。我们看它的实现：

```java
public class JdkSerializationRedisSerializer implements RedisSerializer<Object> {
   public byte[] serialize(@Nullable Object object) {
  if (object == null) {
   return SerializationUtils.EMPTY_ARRAY;
  }
  try {
   return serializer.convert(object);
  } catch (Exception ex) {
   throw new SerializationException("Cannot serialize", ex);
  }
 }
}
```

serialzer 是一个`SerializingConverter`，它的 convert 方法如下：

```java
public class SerializingConverter implements Converter<Object, byte[]> {
 public byte[] convert(Object source) {
  try  {
   return this.serializer.serializeToByteArray(source);
  }
  catch (Throwable ex) {
   throw new SerializationFailedException("Failed to serialize object using " +
     this.serializer.getClass().getSimpleName(), ex);
  }
 }
}
```

serialzer 是一个`DefaultSerializer`，它的 serializeToByteArray 方法如下：

```java
public class DefaultSerializer implements Serializer<Object> {
 default byte[] serializeToByteArray(T object) throws IOException {
  ByteArrayOutputStream out = new ByteArrayOutputStream(1024);
  serialize(object, out);
  return out.toByteArray();
 }
 @Override
 public void serialize(Object object, OutputStream outputStream) throws IOException {
  if (!(object instanceof Serializable)) {
   throw new IllegalArgumentException(getClass().getSimpleName() + " requires a Serializable payload " +
     "but received an object of type [" + object.getClass().getName() + "]");
  }
  ObjectOutputStream objectOutputStream = new ObjectOutputStream(outputStream);
  objectOutputStream.writeObject(object);
  objectOutputStream.flush();
 }
}
```

因此，hash value 是通过`JdkSerializationRedisSerializer`来序列化的，即将对象通过 ObjectOutputStream 输出为字节数组。故所有的 value 必须实现`Serializable`接口。

其最终存储的内容包括样例：

```bash
  hgetall spring:session:sessions:c89dde1e-db1a-46d6-ae0f-330d821a411b
    1) "maxInactiveInterval"
    2) "\xac\xed\x00\x05sr\x00\x11java.lang.Integer\x12\xe2\xa0\xa4\xf7\x81\x878\x02\x00\x01I\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\a\b"
    3) "sessionAttr:iCount"
    4) "\xac\xed\x00\x05sr\x00\x11java.lang.Integer\x12\xe2\xa0\xa4\xf7\x81\x878\x02\x00\x01I\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x00\x04"
    5) "creationTime"
    6) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01\x908\x81\xda\xda"
    7) "lastAccessedTime"
    8) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01\x908\x9aH\xe6"
```

session key 按照上述`keyNamespace`+ "sessions:" + sessionId 生成，key 包括 maxInactiveInterval、sessionAttr:iCount、creationTime、lastAccessedTime 等属性，value 的内容是二进制的序列化对象。

### 覆盖默认序列化器

能否覆盖默认的序列化器呢？答案是肯定的。我们看`RedisHttpSessionConfiguration`的实现：

```java
public class RedisHttpSessionConfiguration {
  @Bean
  public RedisSessionRepository sessionRepository() {
    RedisTemplate<String, Object> redisTemplate = createRedisTemplate();
    RedisSessionRepository sessionRepository = new RedisSessionRepository(redisTemplate);
    // ...
    if (getDefaultRedisSerializer() != null) {
      redisTemplate.setDefaultSerializer(getDefaultRedisSerializer());
    }
    // ...
    return sessionRepository;
 }
}
```

我们可以看到可以通过`getDefaultRedisSerializer`方法来覆盖默认的序列化器。我们再看`getDefaultRedisSerializer`的实现

```java
public abstract class AbstractRedisHttpSessionConfiguration<T extends SessionRepository<? extends Session>> implements BeanClassLoaderAware {
  private RedisSerializer<Object> defaultRedisSerializer;
 @Autowired(required = false)
 @Qualifier("springSessionDefaultRedisSerializer")
 public void setDefaultRedisSerializer(RedisSerializer<Object> defaultRedisSerializer) {
  this.defaultRedisSerializer = defaultRedisSerializer;
 }
   protected RedisSerializer<Object> getDefaultRedisSerializer() {
  return this.defaultRedisSerializer;
 }
}
```

我们看到`getDefaultRedisSerializer`实际上获取的是属性`defaultRedisSerializer`，它可以通过@Autowired 注解进行了注入，并且是可选的和限定了名称为`springSessionDefaultRedisSerializer`。因此我们可以通过如下方式来进行覆盖：

```java
@SpringBootApplication
class Main {
  @Bean("springSessionDefaultRedisSerializer")
  public RedisSerializer<Object> getSpringSessionDefaultRedisSerializer() {
    return new GenericJackson2JsonRedisSerializer();
  }
}
```

即使用限定名称为`springSessionDefaultRedisSerializer`的 bean 来覆盖默认的序列化器。

## 总结

本文讨论了 Spring Redis 分布式会话的存储格式。我们从 session key、hash key、hash value 三个方面进行了展开。我们发现 session key 是通过`UUID.randomUUID().toString()`生成的，hash key 是通过`StringRedisSerializer`序列化的，hash value 是通过`JdkSerializationRedisSerializer`序列化的。
