---
title: MyBatis探究(五)——枚举类处理
date: 2018/05/21 11:29:00
---

在数据库的使用过程中，经常会遇到用某个数值来表示某种状态、类型或者阶段的情况。

以前我都是使用Integer来表示的，缺点是数值的含义不明确，最好的解决方案是使用枚举来替换整型类型。比如有这样一个枚举：

```java
public enum Sex {
    MALE(1), FEMALE(0);

    private int code;
    Sex(int code) {
        this.code = code;
    }
}
```

我们希望将表示性别的值存入数据库，即`Sex.MALE`存入数据库的值为1，`Sex.FEMALE`存入数据库的值为0
<!-- more -->
本文参考了[
如何在MyBatis中优雅的使用枚举](https://segmentfault.com/a/1190000010755321)。

## Mybatis自带的枚举转换器

Mybatis内置了两个枚举转换器：

- org.apache.ibatis.type.EnumTypeHandler：这是默认的枚举转换器，该转换器将枚举实例转换为实例名称的字符串，即将`Sex.MALE`转换为`MALE`
- org.apache.ibatis.type.EnumOrdinalTypeHandler：这个转换器将枚举实例的ordinal属性作为取值，即`Sex.MALE`转换为`0`，`Sex.FEMALE`转换为`1`

    使用`EnumOrdinalTypeHandler`的方式是在Mybatis配置文件中定义：
    
    ```
    <typeHandlers>
        <typeHandler handler="org.apache.ibatis.type.EnumOrdinalTypeHandler" javaType="love.wangqi.domain.enums.Sex"/>
    </typeHandlers>
    ```

以上两种转换器都不能满足我们的需求，所以要自己编写一个转换器

## 方案

MyBatis提供了`org.apache.ibatis.type.BaseTypeHandler`类用于我们自己扩展类型转换器，上面的`EnumTypeHandler`和`EnumOrdinalTypeHandkler`也都实现了这个接口。

### 定义接口

我们需要一个接口来确定某部分枚举类的行为。如下：

```java
public interface BaseCodeEnum {
    int getCode();
}
```

该接口只有一个返回编码的方法，返回值将被存入数据库。

### 改造枚举

```java
public enum Sex implements BaseCodeEnum {
    MALE(1), FEMALE(0);

    private int code;

    Sex(int code) {
        this.code = code;
    }

    @Override
    public int getCode() {
        return code;
    }
}
```

### 编写转换工具类

现在我们能顺利的将枚举转换为某个数值了，还需要一个工具将数值转换为枚举实例。

```java
public class CodeEnumUtil {
    public static <E extends Enum<?> & BaseCodeEnum> E codeOf(Class<E> enumClass, int code) {
        E[] enumConstants = enumClass.getEnumConstants();
        for (E e : enumConstants) {
            if (e.getCode() == code)
                return e;
        }
        return null;
    }
}
```

### 自定义类型转换器

```java
public class CodeEnumTypeHandler<E extends Enum<?> & BaseCodeEnum> extends BaseTypeHandler<BaseCodeEnum> {
    private Class<E> type;

    public CodeEnumTypeHandler(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
    }

    /**
     * 用于定义设置参数时，该如何把Java类型的参数转换为对应的数据库类型
     * @param ps
     * @param i
     * @param parameter
     * @param jdbcType
     * @throws SQLException
     */
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, BaseCodeEnum parameter, JdbcType jdbcType) throws SQLException {
        ps.setInt(i, parameter.getCode());
    }

    /**
     * 用于定义通过字段名称获取字段数据时，如何把数据库类型转换为对应的Java类型
     * @param rs
     * @param columnName
     * @return
     * @throws SQLException
     */
    @Override
    public BaseCodeEnum getNullableResult(ResultSet rs, String columnName) throws SQLException {
        int code = rs.getInt(columnName);
        return rs.wasNull() ? null : codeOf(code);
    }

    /**
     * 用于定义通过字段索引获取字段数据时，如何把数据库类型转换为对应的Java类型
     * @param rs
     * @param columnIndex
     * @return
     * @throws SQLException
     */
    @Override
    public BaseCodeEnum getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        int code = rs.getInt(columnIndex);
        return rs.wasNull() ? null : codeOf(code);
    }

    /**
     * 用于定义调用存储过程后，如何把数据库类型转换为对应的Java类型
     * @param cs
     * @param columnIndex
     * @return
     * @throws SQLException
     */
    @Override
    public BaseCodeEnum getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        int code = cs.getInt(columnIndex);
        return cs.wasNull() ? null : codeOf(code);
    }

    private E codeOf(int code) {
        try {
            return CodeEnumUtil.codeOf(type, code);
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot convert " + code + " to " + type.getSimpleName() + " by code value.", e);
        }
    }
}
```

### 使用

接下来需要指定哪个类使用我们自己编写转换器进行转换，在MyBatis配置文件中配置如下：

```
<typeHandlers>
    <typeHandler handler="love.wangqi.domain.typehandler.CodeEnumTypeHandler" javaType="love.wangqi.domain.enums.Sex"/>
</typeHandlers>
```

经测试`Sex.MALE`被转换为`1`，`Sex.FEMALE`被转换为`0`，达到了预期的效果

### 修改默认枚举转换器

我们在MyBatis中添加`typeHandler`用于指定哪些类使用我们自定义的转换器，一旦系统中的枚举类多了起来，MyBatis的配置文件维护起来会变得非常麻烦，也容易出错。我们可以在`SqlSessionFactory`创建的过程中指定默认枚举转换器。

首先，我们需要一个能确定转换行为的转换器：

```java
public class AutoEnumTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {
    private BaseTypeHandler typeHandler = null;

    public AutoEnumTypeHandler(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        if (BaseCodeEnum.class.isAssignableFrom(type)) {
            // 如果实现了BaseCodeEnum则使用我们自定义的转换器
            typeHandler = new CodeEnumTypeHandler(type);
        } else {
            // 默认转换器也可换成EnumOrdinalTypeHandler
            typeHandler = new EnumTypeHandler<>(type);
        }
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
        typeHandler.setNonNullParameter(ps, i, parameter, jdbcType);
    }

    @Override
    public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return (E) typeHandler.getNullableResult(rs, columnName);
    }

    @Override
    public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return (E) typeHandler.getNullableResult(rs, columnIndex);
    }

    @Override
    public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return (E) typeHandler.getNullableResult(cs, columnIndex);
    }
}
```

接下来，我们在`SqlSessionFactory`的创建过程中，指定默认枚举转换器

```java
// 取得类型转换注册器
TypeHandlerRegistry typeHandlerRegistry = sqlSessionFactory.getConfiguration().getTypeHandlerRegistry();
// 注册默认枚举转换器
typeHandlerRegistry.setDefaultEnumTypeHandler(AutoEnumTypeHandler.class);
```

#### spring boot

如果在spring boot中引入`mybatis-spring-boot-starter`包，则不需要手动创建SqlSessionFactory，只需要在配置文件中指定`mybatis.default-enum-type-handler=love.wangqi.springbootmybatisdemo.model.enums.typehandler.AutoEnumTypeHandler`

#### mybatis plus

如果使用的是mybatis plus，则需要在配置文件中指定typeHandler所在的包：`mybatis-plus.type-handlers-package`。并且不再需要`AutoEnumTypeHandler`，mybatis plus会根据枚举类型选择合适的类型转换器。

## 与前端交互

经过以上处理，我们实现了枚举类型到数据库数据的相互转化。我们还需要使用枚举类型与前端进行交互，这部分详见[Jackson枚举类处理][1]

[1]: /articles/Java/Jackson枚举类处理.html

> https://segmentfault.com/a/1190000010755321

