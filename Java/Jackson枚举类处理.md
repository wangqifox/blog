---
title: Jackson枚举类处理
date: 2018/05/21 11:28:00
---

在[MyBatis探究(五)——枚举类处理][1]一文中，我们知道了如何在mybatis中使用枚举类型。这之后我们还需要正确处理与前端的交互，即正确处理枚举类型的序列化与反序列化操作。

本文将讨论如何在Jackson中处理枚举类型的序列化和反序列化。

<!-- more -->

默认情况下，Jackson会将`Sex.MALE`序列化成`MALE`，`Sex.FEMALE`序列化成`FEMALE`；反序列化时`Sex.MALE`只能接受`MALE`，`Sex.FEMALE`只能接受`FEMALE`。这不符合我们的需要，因此我们需要自定义序列化器和反序列化器。

## 序列化器

序列化器继承`StdSerializer`，重写`serialize`方法，向`JsonGenerator`中写入序列化后的值。

```java
public class BaseCodeEnumSerializer extends StdSerializer<BaseCodeEnum> {
    public BaseCodeEnumSerializer() {
        this(null);
    }

    public BaseCodeEnumSerializer(Class<BaseCodeEnum> t) {
        super(t);
    }

    @Override
    public void serialize(BaseCodeEnum value, JsonGenerator gen, SerializerProvider provider) throws IOException {
        int code = value.getCode();
        gen.writeNumber(code);
    }
}
```

## 反序列化器

反序列化继承`StdDeserializer`，重写`deserialize`方法，根据`JsonParser`中的值返回反序列化后的值。

```java
public class BaseCodeEnumDeserializer<T extends BaseCodeEnum> extends StdDeserializer<T> {

    private Class<T> type;

    public BaseCodeEnumDeserializer() {
        this(null);
    }

    public BaseCodeEnumDeserializer(Class<T> vc) {
        super(vc);
        type = vc;
    }


    @Override
    public T deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {
        for (T t : type.getEnumConstants()) {
            if (t.getCode() == Integer.valueOf(p.getText())) {
                return t;
            }
        }
        return null;
    }
}
```

## 注册序列化器和反序列化器

```java
ObjectMapper mapper = new ObjectMapper();
SimpleModule module = new SimpleModule();
module.addSerializer(BaseCodeEnum.class, new BaseCodeEnumSerializer());
module.addDeserializer(BaseCodeEnum.class, new BaseCodeEnumDeserializer<>());
mapper.registerModule(module);
```

新建`SimpleModule`，在`SimpleModule`中注册我们自定义的序列化和反序列化器，然后向`ObjectMapper`中注册module。

经过以上步骤，Jackson会将`Sex.MALE`序列化成`1`，`Sex.FEMALE`序列化成`0`；反序列化时`Sex.MALE`接受`1`，`Sex.FEMALE`接受`0`。结果符合我们的预期。

[1]: /articles/mybatis/MyBatis探究(五)——枚举类处理.html

