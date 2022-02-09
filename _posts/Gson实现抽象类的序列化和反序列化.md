---
title: Gson实现抽象类的序列化和反序列化
tags: 编程基础
categories: 编程基础
keywords: 'Gson自定义适配器,Gson抽象类，Gson序列化，Gson反序列化，抽象类序列化，抽象类反序列化'
'description:Gson自定义适配器,Gson对抽象类序列化反序列化，如何使用Gson序列化反序列化抽象类，Exception in thread "main" java.lang.RuntimeException': Failed to invoke public xxx() with no args
abbrlink: 9854a92d
date: 2022-01-21 17:30:08
---

Gson算是比较轻量和简单的一个Json框架了，使用也很简单，直接：
```java
Gson gson = new Gson();
```
就行了，但是有一种场景下，假设要序列化的对象是个抽象类的一个实现，例如这样：

```java
abstract class A {
    protected int a;
    protected int b;
}

class B extends A {
    private int c;
}

class C {
    private A b = new B();
}
```

当你创建一个B的对象，序列化后，如果反序列化的时候传入的class type是A的话是会出错的，或者序列化/反序列化C对象的时候也会出错，也就是，：

```
Exception in thread "main" java.lang.RuntimeException: Failed to invoke public xxx() with no args
```

这是因为默认new 出来的Gson对象在反序列的时候，并不知道这个抽象对象A，应该如何去创建对象，在Java中抽象类是无法创建对象实例的，因此会出现`no args`的问题。

要解决这个问题，思路就很清晰，在某个实际告诉Gson，把这个对象反序列化成什么实现类，要达到这个效果就需要自己重写一下Json的序列化和反序列化的适配。

```java
public class CustomJsonAdapter<T> implements JsonSerializer<T>, JsonDeserializer<T> {
    
    @Override
    public T deserialize(JsonElement jsonElement, Type type, JsonDeserializationContext context) throws JsonParseException {
        try {
            return new Gson().fromJson(jsonElement, (Type) Class.forName(((Class) obj).getName()));
        } catch (ClassNotFoundException e) {
            throw new JsonParseException(e);
        }
    }

    @Override
    public JsonElement serialize(T t, Type type, JsonSerializationContext jsonSerializationContext) {
        return jsonSerializationContext.serialize(t);
    }
}

```

例如通过上面的方式实现一个自己的适配器，在deserialize里面实现反序列化的解析，告诉Gson，应该把这个jsonElement映射成哪个对象。

使用的话就比较坑，有两种情况，高版本的Gson可以直接`@JsonAdapter`注解，也就是这样：
```java
@JsonAdapter(CustomJsonAdapter.class)
abstract class A {
    protected int a;
    protected int b;
}
```
直接将注解定义在这个抽象类上，也就是告诉Gson如果遇到这个抽象类的定义需要去反序列化，使用哪个适配器去包装，还有一种方式是在创建Gson对象的时候进行注入：

```java
Gson gson = new GsonBuilder()
            .registerTypeAdapter(A.class, new CustomJsonAdapter())
            .create();
```

这两种实现的区别是低版本的Gson只支持后者，高版本的Gson支持注解的形式，我自己发现`gson-2.2.4.jar`是不支持注解的，而`gson-2.8.9.jar`是支持注解的。
