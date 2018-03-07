---
title: chameleon项目安利
date: 2016-12-07 17:18:31
tags:
- Java
- Convert
categories:
- Java
---

chameleon(变色龙) 是一个Java的Bean Converter轮子，实现注解化类型转换。

[Github地址](https://github.com/yannxia/chameleon)

<!-- more -->

## 背景
经常在业务里写 A -> B 这样的转换方法，每次都需要手动调用极其的麻烦，而且每次这种代码复用起来又是比较麻烦的，最初尝试过使用 com.google.common.base.Converter的这个转换接口，
包括 Spring的org.springframework.core.convert.converter.Converter，虽然从接口层面定义了 A -> B 问题，但是没有发现ConverterFacotory这样可用的轮子，只能自己尝试制造一个。

## 分析
需要解决的是 A -> B 的问题。我们也有 Converter Interface， 那我们只缺少一个 ConverterFacotory。那问题的解决方案就也有啦，我们就是要去写一个ConverterFacotory。  
现在的Java注解使用的更多些，尤其在SpringBoot中，那Converter Interface其实也不是必须的，使用Annonation也可以，最终还是选择使用注解。

## 编码分析
从SpringCotext从得到的思路，那核心应该是一个HashMap. Key应该是 From（转换方法入参数）+To（转换方法的出参）， Value是Method（具体的转换方法） 和 Object（具体的类）。
那我们可以大致上确定数据结构。


```java
//核心容器的Key
class ConvertKey {
    private List<Class> froms;
    private Class to;

    public ConvertKey(Class to, Class... froms) {
        this.froms = Arrays.asList(froms);
        this.to = to;
    }

    @Override
    public int hashCode() {
        int hashCode = to.getName().hashCode();
        for (Class from : froms) {
            hashCode &= from.getName().hashCode();
        }
        return hashCode;
    }

    @Override
    public boolean equals(Object obj) {
        if (obj instanceof ConvertKey) {
            ConvertKey other = (ConvertKey) obj;
            return other.froms.equals(this.froms) && other.to.equals(this.to);
        }
        return super.equals(obj);
    }
}

```

```java
//核心容器的Value
class CovertInstant {
    Object convertObj;
    Method covertMethod;

    public CovertInstant(Object convertObj, Method covertMethod) {
        this.convertObj = convertObj;
        this.covertMethod = covertMethod;
    }
}
```
```java
//核心容器
public abstract class AbstractConvertFactory implements ConvertFactory {

    protected ConcurrentHashMap<ConvertKey, CovertInstant> keyCovertInstantConcurrentHashMap = new ConcurrentHashMap<>(); //存储核心

    public <T> T convert(Class<T> expect, Object... params) {
    }
}
```

从三个实体定义差不多就可以确定出，我们的代码了，整体设计就是把可以转换的方法放入这个核心容器内。

## 编码
我们额外定义一个注解，用来标记是否为一个转换方法 [Convertor注解](https://github.com/yannxia/chameleon/blob/master/src/main/java/info/yannxia/java/chameleon/annonation/Convertor.java)

我们需要实现一个具体的ConvertFactory。

```java
public class ConvertFactoryImpl extends AbstractConvertFactory {

    private ConvertFactoryImpl() {
    }

    public static ConvertFactoryImpl build(Object... objects) {
        ConvertFactoryImpl convertFactory = new ConvertFactoryImpl();
        Arrays.stream(objects)
                .forEach(convertFactory::setupObject); //JDK8 就是好，安利
        return convertFactory;
    }
}
```

```java
void setupObject(Object object) {
        Method[] methods = object.getClass().getMethods(); //遍历每一个方法，这里仅仅支持Public方法，防止Private的滥用。
        Arrays.stream(methods)
                .filter(AbstractConvertFactory::isConvertorMethod)
                .forEach(method -> {
                    Class[] froms = method.getParameterTypes();
                    Class to = method.getReturnType();
                    ConvertKey convertKey = new ConvertKey(to, froms);
                    CovertInstant covertInstant = new CovertInstant(object, method);

                    if (this.keyCovertInstantConcurrentHashMap.containsKey(convertKey)) {
                        throw new IllegalArgumentException(String.format("already has [%s] -> [%s] method", froms, to));
                    }

                    this.keyCovertInstantConcurrentHashMap.put(convertKey, covertInstant);
                });
    }

    private static boolean isConvertorMethod(Method method) {
        return method.getAnnotation(Convertor.class) != null;
    }
```

## 补充
关于Spring的集成见 [SpringConvertFactoryImplLoader.java](https://github.com/yannxia/chameleon/blob/master/src/main/java/info/yannxia/java/chameleon/SpringConvertFactoryImplLoader.java)

Spring集成的核心在于，如何从构造好的ApplicationContext中取出那些注册好的Bean，然后构建我们自己的ConvertFactory，不过整体实现难度很低就不去贴代码了。

## 测试
[CovertFacotry测试](https://github.com/yannxia/chameleon/blob/master/src/test/java/info/yannxia/java/chameleon/ConvertFactoryTest.java)
[SpringConvertFactoryTest测试](https://github.com/yannxia/chameleon/blob/master/src/test/java/info/yannxia/java/chameleon/SpringConvertFactoryTest.java)


## PS
写博客的时候搜索了下，发现了Spring有个这么个玩意 [Spring](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/validation.html#core-convert)  
那就算是自己造了一个轮子吧。

**预定下一篇博客去研究下Spring的convert机制。**
