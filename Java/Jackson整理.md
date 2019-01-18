---
title: Jackson整理
date: 2018/05/08 18:28:00
---

Jackson是最流行的JSON API之一，它提供多种不同的方式来处理JSON。

Jackson包含2个不同的JSON解析器：

- ObjectMapper
- JsonParser

包含2个不同的JSON生成器：

- ObjectMapper
- JsonGenerator

Jackson包含一个核心JAR文件和2个另外的JAR文件：

- Jackson Core
- Jackson Annotations
- Jackson Databind

Jackson Annotation使用Jackson Core，Jackson Databind使用Jackson Annotaion。

如果使用maven来管理依赖的话，只需要加入`jackson-databind`，它会自动依赖另外的两个包。

<!-- more -->

# Jackson ObjectMapper

在Jackson中，`ObjectMapper`是最简单也是最常用的类。`ObjectMapper`可以用于从字符串、流、文件等处将JSON解析成Java对象或者对象图(object graph)。这个过程称为反序列化。

`ObjectMapper`也可以从Java对象中创建JSON。这个过程称为序列化。

Jackson的`ObjectMapper`可以将JSON反序列化到自定义的类对象中，或者反序列化到内建的JSON数模型中。

## Jackson ObjectMapper的工作原理

为了正确地将JSON反序列到Java对象中，了解Jackson是如何将JSON对象的字段映射到Java对象的字段是非常重要的。

默认情况下，Jackson通过匹配JSON对象中的字段和Java对象中的getter和setter方法来对JSON对象的字段和Java对象的字段进行映射。Jackson将getter和setter方法名的get和set部分，然后把第一个字母转成小写。

比如，JSON字段`brand`匹配`getBrand()`和`setBrand()`方法。`engineNumber`匹配`getEngineNumber`和`setEngineNumber`方法。

## 从JSON字符串中读取对象

```java
ObjectMapper objectMapper = new ObjectMapper();

String carJson =
    "{ \"brand\" : \"Mercedes\", \"doors\" : 5 }";

Car car = objectMapper.readValue(carJson, Car.class);
```

## 从Reader中读取对象

```java
ObjectMapper objectMapper = new ObjectMapper();

String carJson =
        "{ \"brand\" : \"Mercedes\", \"doors\" : 4 }";
Reader reader = new StringReader(carJson);

Car car = objectMapper.readValue(reader, Car.class);
```

## 从File中读取对象

可以从FileReader、StringReader或者File中读取对象

```java
ObjectMapper objectMapper = new ObjectMapper();

File file = new File("data/car.json");

Car car = objectMapper.readValue(file, Car.class);
```

### 从URL中读取对象

```java
ObjectMapper objectMapper = new ObjectMapper();

URL url = new URL("file:data/car.json");

Car car = objectMapper.readValue(url, Car.class);
```

## 从InputStream中读取对象

```java
ObjectMapper objectMapper = new ObjectMapper();

InputStream input = new FileInputStream("data/car.json");

Car car = objectMapper.readValue(input, Car.class);
```

## 从字节数组中读取对象

```java
ObjectMapper objectMapper = new ObjectMapper();

String carJson =
        "{ \"brand\" : \"Mercedes\", \"doors\" : 5 }";

byte[] bytes = carJson.getBytes("UTF-8");

Car car = objectMapper.readValue(bytes, Car.class);
```

## 从JSON数组字符串中读取对象数组

```java
String jsonArray = "[{\"brand\":\"ford\"}, {\"brand\":\"Fiat\"}]";

ObjectMapper objectMapper = new ObjectMapper();

Car[] cars2 = objectMapper.readValue(jsonArray, Car[].class);
```

## 从JSON数组字符串中读取对象列表

```java
String jsonArray = "[{\"brand\":\"ford\"}, {\"brand\":\"Fiat\"}]";

ObjectMapper objectMapper = new ObjectMapper();

List<Car> cars1 = objectMapper.readValue(jsonArray, new TypeReference<List<Car>>(){});
```

## 忽略未知的JSON字段

有时候JSON中的字段会比反序列化的目标Java对象字段多。默认情况下Jackson会抛出错误，告诉你JSON中的某个字段在Java对象中找不到。

但是，有时候需要允许JSON字段比Java对象多。Jackson允许你忽略这些多余的字段：

```java
objectMapper.configure(
    DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```

## 自定义反序列化

有时候可能想要以自己的方式来反序列化JSON字符串，而不是ObjectMapper默认的方式。你为ObjectMapper可以自定义一个反序列化器，这个反序列化器可以以你需要的方式工作。

下面的代码展示了如何注册并使用自定义的反序列化器：

```java
String json = "{ \"brand\" : \"Ford\", \"doors\" : 6 }";

SimpleModule module =
        new SimpleModule("CarDeserializer", new Version(3, 1, 8, null, null, null));
module.addDeserializer(Car.class, new CarDeserializer(Car.class));

ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(module);

Car car = mapper.readValue(json, Car.class);
```

以下代码是`CarDeserializer`:

```java
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.core.JsonToken;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.deser.std.StdDeserializer;

import java.io.IOException;

public class CarDeserializer extends StdDeserializer<Car> {

    public CarDeserializer(Class<?> vc) {
        super(vc);
    }

    @Override
    public Car deserialize(JsonParser parser, DeserializationContext deserializer) throws IOException {
        Car car = new Car();
        while(!parser.isClosed()){
            JsonToken jsonToken = parser.nextToken();

            if(JsonToken.FIELD_NAME.equals(jsonToken)){
                String fieldName = parser.getCurrentName();
                System.out.println(fieldName);

                jsonToken = parser.nextToken();

                if("brand".equals(fieldName)){
                    car.setBrand(parser.getValueAsString());
                } else if ("doors".equals(fieldName)){
                    car.setDoors(parser.getValueAsInt());
                }
            }
        }
        return car;
    }
}
```

## 序列化对象

`ObjectMapper`也可以用于从对象生成JSON。使用以下方法中的一个：

- writeValue()
- writeValueAsString()
- writeValueAsBytes()

以下是将`Car`对象序列化为JSON的例子：

```java
ObjectMapper objectMapper = new ObjectMapper();

Car car = new Car();
car.brand = "BMW";
car.doors = 4;

objectMapper.writeValue(
    new FileOutputStream("data/output-2.json"), car);
```

上面的例子首先新建一个`ObjectMappper`，然后新建一个`Car`实例，最后调用`ObjectMapper`的`writeValue()`方法将`Car`对象序列化成JSON，写到给定的`FileOutputStream`中。

`ObjectMapper`的`writeValueAsString()`和`writeValueAsByte()`方法都可以生成JSON，返回字符串或者字节数组。下面的例子展示了如何调用`writeValueAsString()`：

```java
ObjectMapper objectMapper = new ObjectMapper();

Car car = new Car();
car.brand = "BMW";
car.doors = 4;

String json = objectMapper.writeValueAsString(car);
System.out.println(json);
```

上面的例子输出如下JSON：

```java
{"brand":"BMW","doors":4}
```

## 自定义序列化器

有时候你想要生成一个不同于默认规则的JSON。比如，使用不用于Java对象的字段名称，或者忽略某些字段。

Jackson可以让你自定义ObjectMapper的序列化器。序列化器注册到类上，每次ObjectMapper序列化这个类时都会被调用。下面的例子展示了如何给`Car`注册一个自定义的序列化器。

```java
CarSerializer carSerializer = new CarSerializer(Car.class);
ObjectMapper objectMapper = new ObjectMapper();

SimpleModule module =
        new SimpleModule("CarSerializer", new Version(2, 1, 3, null, null, null));
module.addSerializer(Car.class, carSerializer);

objectMapper.registerModule(module);

Car car = new Car();
car.setBrand("Mercedes");
car.setDoors(5);

String carJson = objectMapper.writeValueAsString(car);
```

这个自定义的序列化器输出如下JSON：

```java
{"producer":"Mercedes","doorCount":5}
```

`CarSerializer`如下所示：

```java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.databind.ser.std.StdSerializer;

import java.io.IOException;

public class CarSerializer extends StdSerializer<Car> {

    protected CarSerializer(Class<Car> t) {
        super(t);
    }

    public void serialize(Car car, JsonGenerator jsonGenerator,
                          SerializerProvider serializerProvider)
            throws IOException {

        jsonGenerator.writeStartObject();
        jsonGenerator.writeStringField("producer", car.getBrand());
        jsonGenerator.writeNumberField("doorCount", car.getDoors());
        jsonGenerator.writeEndObject();
    }
}
```

注意，传递到`serialize()`方法的第二个参数是Jackson JsonGenerator实例。你可以使用这个实例来序列化对象。

## 日期格式

默认情况下，Jackson会将`java.util.Date`对象转化为`long`值，即距离1970年1月1日的毫秒值。然而Jackson也支持将日期格式化为字符串。

### 日期转化为long

首先展示一下默认情况下Jackson将`Date`转化为距离1970年1月1日的毫秒值。

```java
public class Transaction {
    private String type = null;
    private Date date = null;

    public Transaction() {
    }

    public Transaction(String type, Date date) {
        this.type = type;
        this.date = date;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public Date getDate() {
        return date;
    }

    public void setDate(Date date) {
        this.date = date;
    }
}
```

序列化`Transaction`对象和之前序列化其他对象是一样的。

```java
Transaction transaction = new Transaction("transfer", new Date());

ObjectMapper objectMapper = new ObjectMapper();
String output = objectMapper.writeValueAsString(transaction);

System.out.println(output);
```

输出和以下字符串类似的JSON：

```java
{"type":"transfer","date":1516442298301}
```

注意`date`字段的格式，它是一个long整型。

### 日期转换为字符串

`Date`转化为long整型非常不适合人类阅读。因此Jackson支持更加适合阅读的格式。你可以给`ObjectMapper`设置`SimpleDateFormat`来给`Date`类型指定格式。下面是这样的一个例子：

```java
SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
objectMapper2.setDateFormat(dateFormat);

String output2 = objectMapper2.writeValueAsString(transaction);
System.out.println(output2);
```

输出类似如下的字符串：

```java
{"type":"transfer","date":"2018-01-20"}
```

## JSON树模型

Jackson内建有一个可用于表示JSON对象的树模型。如果你不知道需要接收的JSON数据是怎样的，或者你无法创建一个类来表示，Jackson的树模型则非常有用。

Jackson的树模型用`JsonNode`类来表示。你可以以之前的方法，使用`ObjectMapper`来将JSON转化为`JsonNode`。

### Jackson树模型示例

```java
String carJson =
        "{ \"brand\" : \"Mercedes\", \"doors\" : 5 }";

ObjectMapper objectMapper = new ObjectMapper();

try {

    JsonNode node = objectMapper.readValue(carJson, JsonNode.class);

} catch (IOException e) {
    e.printStackTrace();
}
```

如上例所示，只要简单地将传递到`readValue()`方法的第二个参数改为`JsonNode.class`，JSON就可以被转化成`JsonNode`对象。

### JsonNode类

下面的例子展示了如何通过JsonNode来访问JSON字段、数组、嵌套对象：

```java
String carJson =
        "{ \"brand\" : \"Mercedes\", \"doors\" : 5," +
        "  \"owners\" : [\"John\", \"Jack\", \"Jill\"]," +
        "  \"nestedObject\" : { \"field\" : \"value\" } }";

ObjectMapper objectMapper = new ObjectMapper();


try {

    JsonNode node = objectMapper.readValue(carJson, JsonNode.class);

    JsonNode brandNode = node.get("brand");
    String brand = brandNode.asText();
    System.out.println("brand = " + brand);

    JsonNode doorsNode = node.get("doors");
    int doors = doorsNode.asInt();
    System.out.println("doors = " + doors);

    JsonNode array = node.get("owners");
    JsonNode jsonNode = array.get(0);
    String john = jsonNode.asText();
    System.out.println("john  = " + john);

    JsonNode child = node.get("nestedObject");
    JsonNode childField = child.get("field");
    String field = childField.asText();
    System.out.println("field = " + field);

} catch (IOException e) {
    e.printStackTrace();
}
```

注意，JSON字符串包含一个数组字段`owners`和一个嵌套对象字段`nestedObject`。

不管是访问一个字段、数组、嵌套对象，都可以使用`get()`方法。通过在`get()`方法中传递一个字符串作为参数，可以访问`JsonNode`的一个字段。如果`JsonNode`代表数组，你需要在`get()`方法中传递一个索引值，索引值表示你想要获取的元素的位置。

# JsonParser

`JsonParser`是一个底层的JSON解析器。它比`ObjectMapper`要更加底层，因此比`ObjectMapper`更快，但是也比`ObjectMapper`更笨重。

## 创建JsonParser

首先需要创建`JsonFactory`。`JsonFactory`用于创建`JsonParser`实例。`JsonFactory`包含多个`createParser()`方法，使用不同的JSON源作为参数。

以下是创建`JsonParser`的示例：

```java
String carJson =
        "{ \"brand\" : \"Mercedes\", \"doors\" : 5 }";

JsonFactory factory = new JsonFactory();
JsonParser  parser  = factory.createParser(carJson);
```

你也可以将`Reader`、`InputStream`、`URL`、字节数组、字符数组传入`createParser()`方法。

## 使用JsonParser解析JSON

创建完`JsonParser`之后，你可以使用它来解析JSON。`JsonParser`的工作原理就是将JSON分解成一系列的token，你可以一个个迭代它们。

以下示例简单展示了迭代tokens，将它们打印到标准输出。这不是一个非常有用的例子，但是它展示了JSON中的tokens被分解，以及如何迭代tokens。

```java
String carJson =
        "{ \"brand\" : \"Mercedes\", \"doors\" : 5 }";

JsonFactory factory = new JsonFactory();
JsonParser  parser  = factory.createParser(carJson);

while(!parser.isClosed()){
    JsonToken jsonToken = parser.nextToken();

    System.out.println("jsonToken = " + jsonToken);
}
```

只要`JsonParser`的`isClosed()`方法返回false，则表示JSON中还有没被访问的token。

你可以使用`JsonParser`的`nextToken()`方法来获得`JsonToken`。你可以使用`JsonToken`实例来检查给定的token。Token类型由`JsonToken`中一系列的常量来表示。这些常量如下所示：

```java
START_OBJECT
END_OBJECT
START_ARRAY
END_ARRAY
FIELD_NAME
VALUE_EMBEDDED_OBJECT
VALUE_FALSE
VALUE_TRUE
VALUE_NULL
VALUE_STRING
VALUE_NUMBER_INT
VALUE_NUMBER_FLOAT
```

你可以使用这些常量找出当前的`JsonToken`是什么类型，通过常量的`equals()`方法。

```java
String carJson =
        "{ \"brand\" : \"Mercedes\", \"doors\" : 5 }";

JsonFactory factory = new JsonFactory();
JsonParser  parser  = factory.createParser(carJson);

Car car = new Car();
while(!parser.isClosed()){
    JsonToken jsonToken = parser.nextToken();

    if(JsonToken.FIELD_NAME.equals(jsonToken)){
        String fieldName = parser.getCurrentName();
        System.out.println(fieldName);

        jsonToken = parser.nextToken();

        if("brand".equals(fieldName)){
            car.brand = parser.getValueAsString();
        } else if ("doors".equals(fieldName)){
            car.doors = parser.getValueAsInt();
        }
    }
}

System.out.println("car.brand = " + car.brand);
System.out.println("car.doors = " + car.doors);
```

如果token指向的是字段名称，`JsonParser`的`getCurrentName()`方法返回当前字段的名称。

如果token指向的是字符串字段值，`getValueAsString()`方法以字符串的形式返回当前token的值。如果token指向的是整型字段值，`getValueAsInt()`方法以`int`的形式返回当前token的值。`JsonParser`还有更多相似的方法以不同类型获取当前token的值。

# JsonGenerator

`JsonGenerator`用于从Java对象中生成JSON。

## 创建JsonGenerator

首先要创建`JsonFactory`实例。以下代码展示了如何创建`JsonFactory`：

```java
JsonFactory factory = new JsonFactory();
```

创建`JsonFactory`之后，你可以使用`JsonFactory`的`createGenerator()`方法来创建`JsonGenerator`。以下代码展示了如何创建`JsonGenerator`：

```java
JsonFactory factory = new JsonFactory();

JsonGenerator generator = factory.createGenerator(
    new File("data/output.json"), JsonEncoding.UTF8);
```

`createGenerator()`方法的第一个参数是生成JSON的目的地。上面例子的参数是一个`File`对象。这意味着生成的JSON对象会被写到给定的文件中。`createGenerator()`方法是重载的，可以插入不同的参数来指定生成的JSON写到何处。

`createGenerator()`方法的第二个参数是生成JSON时使用的字符编码。上面的例子中使用了UTF-8.

## 使用JsonGenerator来生成JSON

创建`JsonGenerator`之后可以开始生成JSON。`JsonGenerator`包含一系列`write...()`方法，你可以用来输出JSON对象的一部分。以下是一个使用`JsonGenerator`来生成JSON的例子：

```java
JsonFactory factory = new JsonFactory();

JsonGenerator generator = factory.createGenerator(
    new File("data/output.json"), JsonEncoding.UTF8);

generator.writeStartObject();
generator.writeStringField("brand", "Mercedes");
generator.writeNumberField("doors", 5);
generator.writeEndObject();

generator.close();
```

这个例子首先调用`writeStartObject()`方法输出`{`。然后调用`writeStringField()`方法输出`brand`字段的名称和值。`writeNumberField()`方法输出`doors`字段的名称和值。最后调用`writeEndObject()`方法输出"}"。

`JsonGenerator`有很多其他用于输出的方法。这个例子只展示了一部分。

## 关闭JsonGenerator

JSON输出完成后，你应该调用`close()`方法关闭`JsonGenerator`：

```java
generator.close();
```

关闭`JsonGenerator`也会关闭文件或者`OutputStream`等。

# 注释

Jackson包含一系列的注释，用于影响JSON的序列化或者反序列化过程。

## 读+写 注释

Jackson包含一系列会同时影响序列化和反序列化过程的注释。我们把这类注释称为"读+写 注释"。

### @JsonIgnore

注释`@JsonIgnore`用于告诉Jackson忽略Java对象中的某个属性(字段)。该属性在序列化和反序列化的过程中都会被忽略。实例如下：

```java
import com.fasterxml.jackson.annotation.JsonIgnore;

public class PersonIgnore {

    @JsonIgnore
    public long    personId = 0;

    public String  name = null;
}
```

上面的类中，属性`personId`在序列化和反序列化时都会被忽略。

### @JsonIgnoreProperties

注释`@JsonIgnoreProperties`用于指定一个列表，处于这个列表中的类属性都会被忽略。注释`@JsonIgnoreProperties`放在类声明之上而不是忽略字段之上。示例如下：

```java
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties({"firstName", "lastName"})
public class PersonIgnoreProperties {

    public long    personId = 0;

    public String  firstName = null;
    public String  lastName  = null;

}
```

在这个例子中，属性`firstName`和`lastName`都会被忽略，因为它们都处于`@JsonIgnoreProperties`注释之中。

### @JsonIgnoreType

注释`@JsonIgnoreType`用于标记整个类，无论这个类在哪被使用，它都会被忽略。示例如下：

```java
import com.fasterxml.jackson.annotation.JsonIgnoreType;

public class PersonIgnoreType {

    @JsonIgnoreType
    public static class Address {
        public String streetName  = null;
        public String houseNumber = null;
        public String zipCode     = null;
        public String city        = null;
        public String country     = null;
    }

    public long    personId = 0;

    public String  name = null;

    public Address address = null;
}
```

以上例子中，所有的`Address`实例都会被忽略。

### @JsonAutoDetect

注释`@JsonAutoDetect`用于告诉Jackson在序列化和反序列化过程中包含非public的属性。示例如下：

```java
import com.fasterxml.jackson.annotation.JsonAutoDetect;

@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.ANY )
public class PersonAutoDetect {

    private long  personId = 123;
    public String name     = null;

}
```

`JsonAutoDetect.Visibility`类包含了匹配可见等级的常量：`ANY`、`DEFAULT`、`NON_PRIVATE`、`NONE`、`PROTECTED_AND_PRIVATE`、`PUBLIC_ONLY`

## 读 注释

Jackson包含一系列只影响反序列化过程的的注释。我们称之为"读 注释"。

### @JsonSetter

注释`@JsonSetter`用于告诉Jackson这个setter方法匹配的属性名称。当JSON中的字段名称和Java对象中的字段名称不同时，这个注释就很有用。

如下所示，`Person`类使用`personId`作为它的id属性：

```java
public class Person {

    private long   personId = 0;
    private String name     = null;

    public long getPersonId() { return this.personId; }
    public void setPersonId(long personId) { this.personId = personId; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

但是在JSON中使用`id`字段而不是`personId`：

```java
{
  "id"   : 1234,
  "name" : "John"
}
```

默认情况下，Jackson无法将JSON中的`id`属性映射到Java类中的`personId`属性。

`@JsonSetter`注释告诉Jackson针对给定的JSON字段使用它所注释的setter方法。我们的例子中，在`setPersonId()`方法上加上`@JsonSetter`注释。

```java
public class Person {

    private long   personId = 0;
    private String name     = null;

    public long getPersonId() { return this.personId; }
    @JsonSetter("id")
    public void setPersonId(long personId) { this.personId = personId; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

`@JsonSetter`注释中指定的值是匹配JSON字段名的值。在这个例子中，这个值是`id`，因为我们要将`id`字段映射到`setPersonId()`方法。

### @JsonAnySetter

`@JsonAnySetter`注释告诉Jackson如果在JSON中遇到不能识别的字段，调用同一个setter方法。不能识别的字段指的是那些不能映射到Java对象的setter方法的字段。

```java
public class Bag {

    private Map<String, Object> properties = new HashMap<>();

    public void set(String fieldName, Object value){
        this.properties.put(fieldName, value);
    }

    public Object get(String fieldName){
        return this.properties.get(fieldName);
    }
}
```

JSON对象如下：

```java
{
  "id"   : 1234,
  "name" : "John"
}
```

Jackson无法直接将JSON对象中的`id`和`name`映射到`Bag`类中，因为`Bag`类不包含相应的公共字段或者setter方法。

你可以使用`@JsonAnySetter`注释，对所有无法识别的字段都调用`set()`方法。

```java
public class Bag {

    private Map<String, Object> properties = new HashMap<>();

    @JsonAnySetter
    public void set(String fieldName, Object value){
        this.properties.put(fieldName, value);
    }

    public Object get(String fieldName){
        return this.properties.get(fieldName);
    }
}
```

现在如果遇到不能识别的字段，都会调用`set()`方法。

注意，这个注释只会在无法识别的字段上生效。比如你在`Bag`类中增加了一个公共的`name`属性或者是一个`setName(String)`方法，JSON中的`name`字段就会映射到这个属性或者setter方法。

### @JsonCreator

`@JsonCreator`注释用于告诉Jackson Java对象拥有一个构造器，JSON中的字段可以映射到这个构造器中的字段。

当无法使用`@JsonSetter`注释时，`@JsonCreator`就能发挥作用了。比如，某些不可变的对象没有任何setter方法，需要将初始值都注入到构造器中。示例如下：

```java
public class PersonImmutable {

    private long   id   = 0;
    private String name = null;

    public PersonImmutable(long id, String name) {
        this.id = id;
        this.name = name;
    }

    public long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

}
```

为了告诉Jackson调用`PersonImmutable`的构造函数，我们必须在构造函数上加上`@JsonCreator`注释。不过这样还不够，我们必须在构造函数的参数中加上注释，告诉Jackson JSON对象的字段映射到构造方法的哪个参数中。示例如下：

```java
public class PersonImmutable {

    private long   id   = 0;
    private String name = null;

    @JsonCreator
    public PersonImmutable(
            @JsonProperty("id")  long id,
            @JsonProperty("name") String name  ) {

        this.id = id;
        this.name = name;
    }

    public long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

}
```

注意构造函数上的注释和构造函数参数上的注释。现在可以从以下JSON对象中反序列化出`PersonImmutable`对象：

```java
{
  "id"   : 1234,
  "name" : "John"
}
```

### @JacksonInject

`@JacksonInject`注释用于在经过解析的对象中注入值，而不是从JSON中读取这些值。举例来说，你从多个不同的源中下载person JSON对象，想要知道person对象的来源。因为来源信息并没有包含在JSON中，你可以将来源信息注入到生成的Java对象中。

为了将Java类的某个字段标记为用于注入，在该字段上加上`@JacksonInject`注释。示例如下：

```java
public class PersonInject {
    public long   id   = 0;
    public String name = null;

    @JacksonInject
    public String source = null;
}
```

为了在`source`字段中注入值，创建`ObjectMapper`时就需要做一些额外的工作。示例如下：

```java
InjectableValues inject = new InjectableValues.Std().addValue(String.class, "jenkov.com");
PersonInject personInject = new ObjectMapper().reader(inject)
                        .forType(PersonInject.class)
                        .readValue(new File("data/person.json"));
```

我们注意到，为了在`source`字段注入值，需要在`InjectableValues addValue()`方法中设置。这个值只能绑定在`String`类型上，而不是特定的字段名。由`@JacksonInject`注释来指定注入到哪个字段中。

如果你是从多个源下载person JSON对象，需要注入多个不同的源，你需要为每个源重复上面的代码。

### @JsonDeserialize

`@JsonDeserialize`注释用于为Java对象的某个字段指定自定义的反序列化类。比如你想要将布尔值`false`和`true`优化为`0`和`1`。

首先你需要在自定义反序列化的字段上加上`@JsonDeserialize`注释。示例如下：

```java
public class PersonDeserialize {

    public long    id      = 0;
    public String  name    = null;

    @JsonDeserialize(using = OptimizedBooleanDeserializer.class)
    public boolean enabled = false;
}
```

自定义的反序列化类`OptimizedBooleanDeserializer`如下所示：

```java
public class OptimizedBooleanDeserializer
    extends JsonDeserializer<Boolean> {

    @Override
    public Boolean deserialize(JsonParser jsonParser,
            DeserializationContext deserializationContext) throws
        IOException, JsonProcessingException {

        String text = jsonParser.getText();
        if("0".equals(text)) return false;
        return true;
    }
}
```

注意`OptimizedBooleanDeserializer`类使用泛型`Boolean`继承`JsonDeserializer`。这样一来`deserialize()`方法返回`Boolean`对象。如果你想要反序列化其他类型（比如`java.util.Date`），必须在泛型括号中指定类型。

调用`jsonParser`参数的`getText()`方法获取反序列化字段的值。你可以根据目标字段，将这个值转换成任意的值和类型。

最后来看看如何使用自定义的反序列化器以及`@JsonDeserializer`注释：

```java
PersonDeserialize person = objectMapper
        .reader(PersonDeserialize.class)
        .readValue(new File("data/person-optimized-boolean.json"));
```

注意，我们首先需要使用`ObjectMapper`的`reader()`方法来为`PersonDeserialize`类创建一个`reader`，然后调用`readValue()`。

## 写 注释

Jackson也包含一系列只影响序列化过程的的注释。我们称之为"写 注释"。

### @JsonInclude

`@JsonInclude`注释告诉Jackson只包含符合某些情况的属性。比如，只包含那些"非null"、"非空"、"非默认值"的属性。如下所示：

```java
import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_EMPTY)
public class PersonInclude {

    public long  personId = 0;
    public String name     = null;

}
```

在上例中，只包含`name`属性，因为`@JsonInclude`中设置的值是"not-empty"，意味着"非null"、"非空字符串"。

`@JsonInclude`注释换一种说法是`@JsonIncludeOnlyWhen`，但是它的名称更长。

### @JsonGetter

`@JsonGetter`注释告诉Jackson通过getter方法来获取某个字段的值，而不是直接通过字段访问。如果Java类使用jQuery风格的getter和setter名称，`@JsonGetter`注释是有用的。比如你用了`personId()`和`personId(long id)`方法而不是`getPersonId()`和`setPersonId()`方法。

示例如下：

```java
public class PersonGetter {

    private long  personId = 0;

    @JsonGetter("id")
    public long personId() { return this.personId; }

    @JsonSetter("id")
    public void personId(long personId) { this.personId = personId; }

}
```

如你所见，`personId()`方法由`@JsonGetter`注释。值设置的方法由`@JsonGetter`注释。`personId`在JSON对象中使用了`id`作为名称。输出的JSON对象如下所示：

```java
{"id":0}
```

注意，`@JsonSetter`注释在反序列化过程中使用，而不是序列化过程中。

### @JsonAnyGetter

`@JsonAnyGetter`注释可以让你使用`Map`作为属性容器，这些属性会被序列化到JSON中。示例如下：

```java
public class PersonAnyGetter {

    private Map<String, Object> properties = new HashMap<>();

    @JsonAnyGetter
    public Map<String, Object> properties() {
        return properties;
    }
}
```

当看到`@JsonAnyGetter`注释时，Jackson获取`@JsonAnyGetter`注释的方法返回的`Map`，然后将`Map`中每个键值对当做属性。换句话说，`Map`中的所有键值对都会被作为`PersonAnyGetter`对象的一部分被序列化。

### @JsonPropertyOrder

`@JsonPropertyOrder`用于指定Java对象中的字段以何种顺序序列化到JSON中。示例如下：

```java
@JsonPropertyOrder({"name", "personId"})
public class PersonPropertyOrder {

    public long  personId  = 0;
    public String name     = null;

}
```

默认情况下，`PersonPropertyOrder`中的字段以它们定义的顺序来序列化。如果需要改变这种顺序，就需要使用`@JsonPropertyOrder`注释来指定不同的顺序。

### @JsonRawValue

`@JsonRawValue`注释告诉Jackson这个属性在序列化时应该直接输出。如果字段是`String`类型，序列化输出时这个值会被默认加上引号，但是如果加上了`@JsonRawValue`注释，Jackson就不会加上引号。

如下所示，当`PersonRawValue`类不加`@JsonRawValue`注释：

```java
public class PersonRawValue {

    public long   personId = 0;

    public String address  = "$#";
}
```

序列化输出如下：

```java
{"personId":0,"address":"$#"}
```

现在我们在`address`属性中加上`@JsonRawValue`注释：

```java
public class PersonRawValue {

    public long   personId = 0;

    @JsonRawValue
    public String address  = "$#";
}
```

序列化`address`属性时Jackson不会加上引号，输出如下：

```java
{"personId":0,"address":$#}
```

当然这是一个无效的JSON，什么时候才真正需要这么做呢？

当`address`属性包含了一个JSON字符串，这个JSON字符串就会作为JSON对象结构的一部分被序列化到最后的JSON对象中，而不是以字符串的形式被序列化。示例如下：

```java
public class PersonRawValue {

    public long   personId = 0;

    @JsonRawValue
    public String address  =
            "{ \"street\" : \"Wall Street\", \"no\":1}";

}
```

序列化输出如下：

```java
{
"personId":0,"address":{ "street" : "Wall Street", "no":1}
}
```

我们主要到，address中的JSON字符串现在是JSON结构中的一部分。

如果没有`@JsonRawValue`注释，序列化输出的JSON就会像这样：

```java
{"personId":0,"address":"{ \"street\" : \"Wall Street\", \"no\":1}"}
```

`address`属性被包裹在引号中，其中的所有引号都被转义了。

### @JsonValue

`@JsonValue`告诉Jackson不要尝试序列化这个对象本身，而是调用这个对象的某个方法，将这个方法返回的对象序列化到JSON。注意如果这个方法返回的字符串中包含引号，Jackson会将这些引号全部转义，所以你不能返回一个JSON对象。如果你有这种需求，应该使用`@JsonRawValue`来替代。

示例如下：

```java
public class PersonValue {

    public long   personId = 0;
    public String name = null;

    @JsonValue
    public String toJson(){
        return this.personId + "," + this.name;
    }

}
```

序列化输出结果如下：

```java
"0,null"
```

结果上会被加上引号，如果里面的字符串包含引号，则会被转义。

### @JsonSerialize

`@JsonSerialize`注释用于给Java对象的字段指定一个自定义的序列化器。示例如下：

```java
public class PersonSerializer {

    public long   personId = 0;
    public String name     = "John";

    @JsonSerialize(using = OptimizedBooleanSerializer.class)
    public boolean enabled = false;
}
```

注意`@JsonSerialize`注释处于`enabled`字段之上。

`OptimizedBooleanSerializer`会将`true`转化为`1`，将`false`转化为`0`，代码如下：

```java
public class OptimizedBooleanSerializer extends JsonSerializer<Boolean> {

    @Override
    public void serialize(Boolean aBoolean, JsonGenerator jsonGenerator, 
        SerializerProvider serializerProvider) 
    throws IOException, JsonProcessingException {

        if(aBoolean){
            jsonGenerator.writeNumber(1);
        } else {
            jsonGenerator.writeNumber(0);
        }
    }
}
```


> https://coding.net/u/wangqi/p/jackson-test/git?public=true
> http://tutorials.jenkov.com/java-json/index.html
> https://my.oschina.net/liuyuantao/blog/865822


