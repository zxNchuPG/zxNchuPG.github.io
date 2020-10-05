---
layout:      post
title:       "MessagePack"
subtitle:    "MessagePack Serialize"
author:      "Ekko"
header-img:  "img/bg/bg-Elasticsearch.jpg"
catalog:     true
tags:
  - 学习笔记
  - 序列化
---

> [官网](https://msgpack.org/)、[github-msgpack-jackson](https://github.com/msgpack/msgpack-java/blob/develop/msgpack-jackson/README.md)、[MessagePack：最可能取代JSON的存在](https://cloud.tencent.com/developer/article/1489591)


## MessagePack 简介

**It's like JSON.but fast and small**

MessagePack 是一种有效的二进制序列化格式，可以在多种语言（ 如 JSON ）之间交换数据。但是它更快、更小，适合序列化传输大批量的数据

MessagePack 之所以比 JSON 又快又好，是因为 MessagePack 遵循最优编码

最优编码也叫 Huffman 编码，什么是最优编码呢，通俗点讲就是没有浪费一丁点信息

## 依赖包

主要依赖 

```xml
<dependency>
    <groupId>org.msgpack</groupId>
    <artifactId>msgpack</artifactId>
    <version>${msgpack.version}</version>
</dependency>
```

第三方扩展包 JackSon 

通过 jackson-databind API 可以轻松读取和写入 MessagePack 编码数据，它扩展了标准的 Jackson 流媒体 API（JsonFactory、JsonParser、JsonGenerator），因此可以与所有更高级别的数据抽象（数据绑定、树结构模型和可插入扩展）无缝地协同工作

> [Jackson-annotations](https://github.com/FasterXML/jackson-annotations)

```xml
<dependency>
  <groupId>org.msgpack</groupId>
  <artifactId>jackson-dataformat-msgpack</artifactId>
  <version>(version)</version>
</dependency>
```

版本用的较多的是 0.8.13 ，默认情况下，此库与 msgpack-java V0.6或更早版本不兼容

---

## 基本用法

**POJO 的序列化/反序列化**

使用 MessagePackFactory 作为参数创建 com.fasterxml.jackson.databind.ObjectMapper

```java
// Instantiate ObjectMapper for MessagePack
ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory());

// Serialize a Java object to byte array
ExamplePojo pojo = new ExamplePojo("komamitsu");
byte[] bytes = objectMapper.writeValueAsBytes(pojo);

// Deserialize the byte array to a Java object
ExamplePojo deserialized = objectMapper.readValue(bytes, ExamplePojo.class);
System.out.println(deserialized.getName()); // => komamitsu
```

使用 ObjectMapper 的 configuer 创建对象

```java
ObjectMapper objectMapper = new ObjectMapper().configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,false);
```

> jackson 的 DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES：反序列化时,遇到未知属性(那些没有对应的属性来映射的属性,并且没有任何 setter 或 handler 来处理这样的属性)时是否引起结果失败(通过抛 JsonMappingException 异常).此项设置只对那些已经尝试过所有的处理方法之后并且属性还是未处理( 这里未处理的意思是：最终还是没有一个对应的类属性与此属性进行映射 )的未知属性才有影响

**List 的序列化/反序列化**

```java
// Instantiate ObjectMapper for MessagePack
ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory());

// Serialize a List to byte array
List<Object> list = new ArrayList<>();
list.add("Foo");
list.add("Bar");
list.add(42);
byte[] bytes = objectMapper.writeValueAsBytes(list);

// Deserialize the byte array to a List
List<Object> deserialized = objectMapper.readValue(bytes, new TypeReference<List<Object>>() {});
System.out.println(deserialized); // => [Foo, Bar, 42]
```

**Map 的序列化/反序列化**

```java
// Instantiate ObjectMapper for MessagePack
ObjectMapper objectMapper = new ObjectMapper(new MessagePackFactory());

// Serialize a Map to byte array
Map<String, Object> map = new HashMap<>();
map.put("name", "komamitsu");
map.put("age", 42);
byte[] bytes = objectMapper.writeValueAsBytes(map);

// Deserialize the byte array to a Map
Map<String, Object> deserialized = objectMapper.readValue(bytes, new TypeReference<Map<String, Object>>() {});
System.out.println(deserialized); // => {name=komamitsu, age=42}
```

**ObjectMapper 反序列化两个常用方法：**

objectMapper.readValue

objectMapper.readTree 返回 JsonNode

---

## 自定义 JackSonUtil 工具类

```java
public class JacksonUtil {
    private ObjectMapper objectMapper = new ObjectMapper().configure(
                DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    private static JacksonUtil instance;

    public static JacksonUtil getInstance() {
        newInstance();
        return instance;
    }

    /**
     * 新建单例，线程安全
     */
    private static synchronized void newInstance() {
        if (instance == null) {
            instance = new JacksonUtil();
            ObjectMapper objectMapper = new ObjectMapper().configure(
                DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
            instance.setObjectMapper(objectMapper);    
        }
    } 

    /**
     * 取单例
     */
    public ObjectMapper getMapper() {
        return objectMapper;
    }

    /**
     * MessagePack + jackson serialize
     */
    public byte[] serialize(Object obj) throws JsonProcessingException {
        if (obj == null) {
            return null;
        }
        return getMapper().writeValueAsBytes(obj);
    }

    public JsonNode deserializeToJson(byte[] bytes) throws IOException {
        if (bytes == null) {
            return null;
        }
        return getMapper().readTree(bytes);
    }

    public JsonNode deserializeToJSON(InputStream input) throws IOException {
        if (input == null){
            return null;
        }
        return getMapper().readTree(input);
    }

    public <T> T deserialize(byte[] bytes, TypeReference typeReference) throws IOException {
        if (bytes == null) {
            return null;
        }
        return getMapper().readValue(bytes, typeReference);
    }
}
```

---

## 使用 RedisTemplate 的情况

```java
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
    // 这个可以是自定义的序列化方法
    RedisSerializer<Object> serializer = redisSerializer();
    RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(redisConnectionFactory);

    // 通过设置的方式使用 messagePack 序列
    redisTemplate.setKeySerializer(new StringRedisSerializer());
    redisTemplate.setValueSerializer(serializer);
    redisTemplate.setHashKeySerializer(new StringRedisSerializer());
    redisTemplate.setHashValueSerializer(serializer);
    redisTemplate.afterPropertiesSet();
    return redisTemplate;
}

@Bean
public RedisSerializer<Object> redisSerializer() {
    //创建JSON序列化器
    Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
    //必须设置，否则无法将JSON转化为对象，会转化成Map类型
    objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance,ObjectMapper.DefaultTyping.NON_FINAL);
    serializer.setObjectMapper(objectMapper);
    return serializer;
}
```

Jackson2JsonRedisSerializer 用到的依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```