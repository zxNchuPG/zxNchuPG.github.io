---
layout:      post
title:       "MessagePack"
subtitle:    "Java 中的 MessagePack 序列化、Jackson 集成与 Redis 使用说明"
author:      "Ekko"
header-img:  "img/bg/bg-Elasticsearch.jpg"
catalog:     true
tags:
  - 学习笔记
  - 序列化
  - Java
  - Redis
---

> 这篇笔记的目标是把 MessagePack 放到 Java 实际开发语境里重新梳理一遍：它是什么、为什么会比 JSON 更紧凑，以及在 `Jackson`、`ObjectMapper` 和 `RedisTemplate` 里应该怎么使用。

> 重点放在可直接落地的序列化方案和常见误区上，尤其是依赖选择、`ObjectMapper` 的正确配置，以及 “看起来用了 MessagePack，实际上还是 JSON 序列化” 这种很容易混淆的场景。

> 参考资料：
>
> [MessagePack Official Site](https://msgpack.org/)
>
> [MessagePack Specification](https://github.com/msgpack/msgpack/blob/master/spec.md)
>
> [msgpack-java README](https://github.com/msgpack/msgpack-java/blob/main/README.md)
>
> [msgpack-jackson README](https://github.com/msgpack/msgpack-java/blob/main/msgpack-jackson/README.md)

[TOC]

---

## 1. MessagePack 是什么

MessagePack 的官方描述很直接：

**It's like JSON, but fast and small.**

可以把它概括为一种面向跨语言交换的二进制序列化格式。它和 JSON 一样，都可以表达对象、数组、字符串、数字、布尔值和空值；不同点在于，JSON 是文本格式，而 MessagePack 是二进制格式。

这意味着它在很多场景下会更适合传输和存储：

- 网络传输时，字节体积通常更小
- 编码和解码过程通常更直接
- 跨语言兼容性较好，适合服务间通信
- 对缓存、消息队列、二进制协议封装更友好

需要注意的是，MessagePack 更小并不是因为它用了 Huffman 编码。更准确地说，它的紧凑性来自规范中定义的一组二进制格式族，例如：

- 小整数可以直接编码进单字节
- 短字符串只需要额外的长度前缀
- 数组、Map、二进制数据都有针对长度区间设计的格式

所以它的优势是“按类型和长度选择更紧凑的二进制表示”，而不是通用压缩算法本身。

## 2. Java 里怎么选依赖

在 Java 生态里，MessagePack 常见有两种使用方式：

- 直接使用底层库，自己做 pack/unpack
- 通过 Jackson 扩展，把 MessagePack 当成一种数据格式接入 `ObjectMapper`

如果只是要接入 `Jackson` 的对象映射能力，通常直接引入 `jackson-dataformat-msgpack` 即可，它会依赖 `msgpack-core`。

```xml
<dependency>
    <groupId>org.msgpack</groupId>
    <artifactId>jackson-dataformat-msgpack</artifactId>
    <version>${msgpack.version}</version>
</dependency>
```

如果项目里需要直接操作底层二进制编解码，也可以显式引入核心库：

```xml
<dependency>
    <groupId>org.msgpack</groupId>
    <artifactId>msgpack-core</artifactId>
    <version>${msgpack.version}</version>
</dependency>
```

这里有一个容易踩坑的点：很多旧文章里还会写 `org.msgpack:msgpack`。这个坐标对应的是更早期的旧实现，现代项目更常见的是 `msgpack-core` 和 `jackson-dataformat-msgpack` 这一套。

另外，`jackson-dataformat-msgpack` 在默认的 POJO 序列化/反序列化语义上，与 `msgpack-java v0.6` 及更早版本并不兼容；如果项目里有非常老的历史数据，需要额外评估兼容策略。

## 3. 基本用法

### 3.1 POJO 的序列化与反序列化

最常见的写法，是把 `MessagePackFactory` 交给 `ObjectMapper`：

```java
ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory());

ExamplePojo pojo = new ExamplePojo("komamitsu");
byte[] bytes = objectMapper.writeValueAsBytes(pojo);

ExamplePojo deserialized = objectMapper.readValue(bytes, ExamplePojo.class);
System.out.println(deserialized.getName()); // komamitsu
```

如果项目里已经深度依赖 Jackson，这种方式几乎没有额外学习成本，因为使用方式和 JSON 的 `ObjectMapper` 很接近，只是底层 `Factory` 换成了 MessagePack。

有些版本里也可以直接使用更简洁的写法：

```java
ObjectMapper objectMapper = new MessagePackMapper();
```

### 3.2 反序列化时忽略未知字段

在服务升级、字段新增、灰度发布这些场景里，反序列化时经常不希望因为多了一个字段就直接失败。这时可以显式关闭 `FAIL_ON_UNKNOWN_PROPERTIES`：

```java
ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory())
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```

这项配置的含义是：当输入数据中出现目标类没有声明、也没有对应处理逻辑的字段时，反序列化不抛异常，而是忽略这些字段。

### 3.3 List 的序列化与反序列化

```java
ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory());

List<Object> list = new ArrayList<>();
list.add("Foo");
list.add("Bar");
list.add(42);

byte[] bytes = objectMapper.writeValueAsBytes(list);

List<Object> deserialized = objectMapper.readValue(
        bytes,
        new TypeReference<List<Object>>() {}
);

System.out.println(deserialized); // [Foo, Bar, 42]
```

### 3.4 Map 的序列化与反序列化

```java
ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory());

Map<String, Object> map = new HashMap<>();
map.put("name", "komamitsu");
map.put("age", 42);

byte[] bytes = objectMapper.writeValueAsBytes(map);

Map<String, Object> deserialized = objectMapper.readValue(
        bytes,
        new TypeReference<Map<String, Object>>() {}
);

System.out.println(deserialized); // {name=komamitsu, age=42}
```

### 3.5 两个常用读取入口

如果已经明确目标类型，通常使用：

```java
objectMapper.readValue(bytes, ExamplePojo.class);
```

如果只是想先读取成树结构，再按需处理字段，可以使用：

```java
JsonNode jsonNode = objectMapper.readTree(bytes);
```

前者适合“有稳定模型类”的场景，后者适合“先观察结构，再决定怎么解析”的场景。

## 4. 封装一个可复用的工具类

如果项目里有多个模块都要处理 MessagePack，一个简单、无状态的工具类会比手写单例更清晰，也更不容易出错。

```java
public final class MessagePackUtil {

    private static final ObjectMapper MAPPER = new ObjectMapper(new MessagePackFactory())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    private MessagePackUtil() {
    }

    public static byte[] serialize(Object value) throws JsonProcessingException {
        if (value == null) {
            return null;
        }
        return MAPPER.writeValueAsBytes(value);
    }

    public static <T> T deserialize(byte[] bytes, Class<T> clazz) throws IOException {
        if (bytes == null) {
            return null;
        }
        return MAPPER.readValue(bytes, clazz);
    }

    public static <T> T deserialize(byte[] bytes, TypeReference<T> typeReference) throws IOException {
        if (bytes == null) {
            return null;
        }
        return MAPPER.readValue(bytes, typeReference);
    }

    public static JsonNode deserializeToTree(byte[] bytes) throws IOException {
        if (bytes == null) {
            return null;
        }
        return MAPPER.readTree(bytes);
    }
}
```

这一版和常见旧代码相比，主要做了几件事：

- `ObjectMapper` 明确绑定到 `MessagePackFactory`
- 去掉了不必要的手写单例逻辑
- 泛型方法补上 `TypeReference<T>` 的类型参数
- 工具类职责只保留“序列化 / 反序列化”，不混入额外状态

## 5. 放到 RedisTemplate 里时要注意什么

原理上，`RedisTemplate` 完全可以搭配 MessagePack 使用，因为 Redis 最终存的就是字节数组。

但这里最容易出现一个误区：**只要用了 Jackson 的序列化器，并不等于已经用了 MessagePack。**

例如下面这种写法：

```java
Jackson2JsonRedisSerializer<Object> serializer =
        new Jackson2JsonRedisSerializer<>(Object.class);
```

它序列化出来的仍然是 JSON，而不是 MessagePack。也就是说，`RedisTemplate` 里如果挂的是 `Jackson2JsonRedisSerializer`，那本质上还是 JSON 方案。

如果确实想让 Redis 的 value 使用 MessagePack，可以自定义一个 `RedisSerializer`，内部把 `ObjectMapper` 换成 MessagePack 版本：

```java
public class MessagePackRedisSerializer<T> implements RedisSerializer<T> {

    private final ObjectMapper objectMapper;
    private final JavaType javaType;

    public MessagePackRedisSerializer(ObjectMapper objectMapper, Class<T> type) {
        this.objectMapper = objectMapper;
        this.javaType = objectMapper.getTypeFactory().constructType(type);
    }

    @Override
    public byte[] serialize(T value) throws SerializationException {
        if (value == null) {
            return null;
        }
        try {
            return objectMapper.writeValueAsBytes(value);
        } catch (IOException e) {
            throw new SerializationException("MessagePack serialize error", e);
        }
    }

    @Override
    public T deserialize(byte[] bytes) throws SerializationException {
        if (bytes == null || bytes.length == 0) {
            return null;
        }
        try {
            return objectMapper.readValue(bytes, javaType);
        } catch (IOException e) {
            throw new SerializationException("MessagePack deserialize error", e);
        }
    }
}
```

然后再把它接到 `RedisTemplate`：

```java
@Bean
public RedisTemplate<String, UserCacheDTO> redisTemplate(
        RedisConnectionFactory redisConnectionFactory) {

    ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    RedisSerializer<UserCacheDTO> valueSerializer =
            new MessagePackRedisSerializer<>(objectMapper, UserCacheDTO.class);

    RedisTemplate<String, UserCacheDTO> redisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(redisConnectionFactory);
    redisTemplate.setKeySerializer(new StringRedisSerializer());
    redisTemplate.setValueSerializer(valueSerializer);
    redisTemplate.setHashKeySerializer(new StringRedisSerializer());
    redisTemplate.setHashValueSerializer(valueSerializer);
    redisTemplate.afterPropertiesSet();
    return redisTemplate;
}
```

对应依赖一般至少需要：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>org.msgpack</groupId>
    <artifactId>jackson-dataformat-msgpack</artifactId>
    <version>${msgpack.version}</version>
</dependency>
```

这里还有一个工程上的取舍：

- 如果缓存对象类型固定，且比较关注体积和编解码效率，MessagePack 很合适
- 如果缓存对象类型很多、还希望直接在 Redis CLI 里肉眼排查，JSON 往往更省心

所以在 Redis 场景里，MessagePack 不一定是默认答案，但它确实是一个很实用的二进制序列化选项。

## 6. 小结

把 MessagePack 放到 Java 项目里看，核心其实只有三件事：

- 先分清楚自己要的是底层二进制编解码，还是 Jackson 风格的数据绑定
- 新项目优先使用 `msgpack-core` 和 `jackson-dataformat-msgpack`
- 在 `RedisTemplate` 里真正想用 MessagePack，就要替换成基于 `MessagePackFactory` 的序列化器，而不是继续使用 JSON serializer

如果只是把它理解成 “二进制版 JSON”，这个理解并不算错；但真正落地时，重点不在概念类比，而在对象映射链路、依赖选型和序列化器配置是否一致。
