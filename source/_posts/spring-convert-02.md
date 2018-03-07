---
title: Spring-convert-(具体调用)
date: 2016-12-27 20:36:10
tags:
- Java
- Spring
categories:
- Java
- Spring
---

续上章，我们还是从具体实现指出看看，Spring Converter到底是怎么工作的。

<!--more-->


## 目录
- [注册Converter](#注册Converter)
- [调用Converter](#调用Converter)
- [查询Converter](#查询Converter)
- [总结](#总结)

---


### 注册Converter

```java
private static void addScalarConverters(ConverterRegistry converterRegistry) {
    converterRegistry.addConverterFactory(new NumberToNumberConverterFactory());
    converterRegistry.addConverter(Number.class, String.class, new ObjectToStringConverter());
}
```
这里我们发现其实有2个不同的注册方法，分别是 addConverter 和 addConverterFactory，我们先看看addConverter。

```java
public <S, T> void addConverter(Class<S> sourceType, Class<T> targetType, Converter<? super S, ? extends T> converter) {
    addConverter(new ConverterAdapter(
            converter, ResolvableType.forClass(sourceType), ResolvableType.forClass(targetType)));
}
```
其实这里是把最终的ObjectToStringConverter的包装成一个ConverterAdapter，而去看ConverterAdapter的声明我们会发现，他最终的接口表现形式是ConditionalGenericConverter。
那问题也清晰起来，还记得上一章，我们聊到ConditionalGenericConverter是一个带条件的转换，具体能否调用，我们需要看

```java
public boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType) {
    // Check raw type first...
    if (this.typeInfo.getTargetType() != targetType.getObjectType()) {  //直接比较 Target 类型，不匹配直接就false
        return false;
    }
    // Full check for complex generic type match required?
    ResolvableType rt = targetType.getResolvableType();
    if (!(rt.getType() instanceof Class) && !rt.isAssignableFrom(this.targetType) &&
            !this.targetType.hasUnresolvableGenerics()) { //比较是不是 targetType的子类，或者不能够泛型化 直接 false
        return false;
    }
    return !(this.converter instanceof ConditionalConverter) ||
            ((ConditionalConverter) this.converter).matches(sourceType, targetType);  //如果是ConditionalConverter 还需要继续自己的matches比较。
}
```
这里我们发现最终注册的ConverterAdapter，是需要比较S和T再确定是否能够调用的。

**这个matches** 其实很有意思。最后一步比较其实就允许 ConditionalConverter的2层嵌套。

---

继续看另外一个addConverterFactory

```java
public void addConverterFactory(ConverterFactory<?, ?> factory) {
    ResolvableType[] typeInfo = getRequiredTypeInfo(factory.getClass(), ConverterFactory.class);
    if (typeInfo == null && factory instanceof DecoratingProxy) {
        typeInfo = getRequiredTypeInfo(((DecoratingProxy) factory).getDecoratedClass(), ConverterFactory.class);
    }
    if (typeInfo == null) {
        throw new IllegalArgumentException("Unable to determine source type <S> and target type <T> for your " +
                "ConverterFactory [" + factory.getClass().getName() + "]; does the class parameterize those types?");
    }
    addConverter(new ConverterFactoryAdapter(factory,
            new ConvertiblePair(typeInfo[0].resolve(), typeInfo[1].resolve())));
}
```
这里我们也发现最终也是得到了ConverterFactoryAdapter，这个就不去细看，和上面那个ConverterAdapter极为相似，我们从中学会了设计模式-[适配器模式](http://design-patterns.readthedocs.io/zh_CN/latest/structural_patterns/adapter.html).
果然Java处处是设计模式。

```java
private static class Converters {
    private final Set<GenericConverter> globalConverters = new LinkedHashSet<GenericConverter>();
    public void add(GenericConverter converter) {
        Set<ConvertiblePair> convertibleTypes = converter.getConvertibleTypes();
        if (convertibleTypes == null) {
            Assert.state(converter instanceof ConditionalConverter,
                    "Only conditional converters may return null convertible types");
            this.globalConverters.add(converter);
        }
        else {
            for (ConvertiblePair convertiblePair : convertibleTypes) {
                ConvertersForPair convertersForPair = getMatchableConverters(convertiblePair);
                convertersForPair.add(converter);
            }
        }
    }
}
```

最终所有的Covert都通过Adapter适配到，我们的核心容器 globalConverters 中。

---

### 调用Converter
```java
public Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType) {
    Assert.notNull(targetType, "targetType to convert to cannot be null");
    if (sourceType == null) {
        Assert.isTrue(source == null, "source must be [null] if sourceType == [null]");
        return handleResult(null, targetType, convertNullSource(null, targetType));
    }
    if (source != null && !sourceType.getObjectType().isInstance(source)) {
        throw new IllegalArgumentException("source to convert from must be an instance of " +
                sourceType + "; instead it was a " + source.getClass().getName());
    }
    GenericConverter converter = getConverter(sourceType, targetType);
    if (converter != null) {
        Object result = ConversionUtils.invokeConverter(converter, source, sourceType, targetType);
        return handleResult(sourceType, targetType, result);
    }
    return handleConverterNotFound(source, sourceType, targetType);
}
```

我们可以发现 getConverter(sourceType, targetType) 这一行，就是查询Converter的核心。而调用的过程就很简单，直接调用Convert.convert()方法就OK了，那我们把注意力放在查询的过程。

---

### 查询Converter

```java
protected GenericConverter getConverter(TypeDescriptor sourceType, TypeDescriptor targetType) {
    ConverterCacheKey key = new ConverterCacheKey(sourceType, targetType); //构建缓存Key
    GenericConverter converter = this.converterCache.get(key); //从缓存中取
    if (converter != null) {
        return (converter != NO_MATCH ? converter : null);
    }

    converter = this.converters.find(sourceType, targetType);
    if (converter == null) {
        converter = getDefaultConverter(sourceType, targetType);
    }

    if (converter != null) {
        this.converterCache.put(key, converter);
        return converter;
    }

    this.converterCache.put(key, NO_MATCH);
    return null;
}
```

这行代码，我们就发现一个在Spring里面用的特别多的设计理念，包括我们平时也可以用的就是缓存。我们下次在自己的项目里可以考虑多使用缓存。关于这个，我也会在后续写一点关于Cache使用的小技巧出来(好像给自己又挖了一个坑)。
那我们继续看find这个方法。

```java
public GenericConverter find(TypeDescriptor sourceType, TypeDescriptor targetType) {
    // Search the full type hierarchy
    List<Class<?>> sourceCandidates = getClassHierarchy(sourceType.getType());
    List<Class<?>> targetCandidates = getClassHierarchy(targetType.getType());
    for (Class<?> sourceCandidate : sourceCandidates) {
        for (Class<?> targetCandidate : targetCandidates) {
            ConvertiblePair convertiblePair = new ConvertiblePair(sourceCandidate, targetCandidate);
            GenericConverter converter = getRegisteredConverter(sourceType, targetType, convertiblePair);
            if (converter != null) {
                return converter;
            }
        }
    }
    return null;
}
```
Wow，我第一次看到这里极为的惊叹，这就是Spring的能力而我所达不到的，所有的声明起的也很清晰，这里是拿出所有的继承的类型。
这样话，可能我们注册的Convert是一个Source类型的父类，那我们也是可以从中获得匹配的方法，在我们没有和入参一样类型的情况下。继续进去看getRegisteredConverter这个方法。

```java
private GenericConverter getRegisteredConverter(TypeDescriptor sourceType,
				TypeDescriptor targetType, ConvertiblePair convertiblePair) {

    // Check specifically registered converters
    ConvertersForPair convertersForPair = this.converters.get(convertiblePair); //尝试从用户自己注册的converters中取
    if (convertersForPair != null) {
        GenericConverter converter = convertersForPair.getConverter(sourceType, targetType);
        if (converter != null) {
            return converter;
        }
    }
    // Check ConditionalConverters for a dynamic match
    for (GenericConverter globalConverter : this.globalConverters) {
        if (((ConditionalConverter) globalConverter).matches(sourceType, targetType)) {
            return globalConverter;
        }
    }
    return null;
}
```
从这段代码里面，我们就能看出，Spring给用户拓展留下的入口是converters这个变量，自己维护的是globalConverters。而ConvertersForPair有兴趣看的读者可以也是维护了一个GenericConverter的数组。
可见最终所有的converter想要可用都是需要调用matches进行类型匹配的，所以我们自己现实一个 Convert 也是不可完全不顾 Source 的类型从而实现一个。


**到这里，Convert服务的注册，发现，调用。**我们都已经了解，但是还缺点什么。比如我们经常会有这样的需求，比如把 List A 转换成 List B，如果我们还需要自己实现岂不是特别笨。
当然Spring这么聪明也为我们想到了。

```java
public static void addCollectionConverters(ConverterRegistry converterRegistry) {
    ConversionService conversionService = (ConversionService) converterRegistry;

    converterRegistry.addConverter(new ArrayToCollectionConverter(conversionService));
    converterRegistry.addConverter(new CollectionToArrayConverter(conversionService));

    converterRegistry.addConverter(new ArrayToArrayConverter(conversionService));
    converterRegistry.addConverter(new CollectionToCollectionConverter(conversionService));
    converterRegistry.addConverter(new MapToMapConverter(conversionService));

    converterRegistry.addConverter(new ArrayToStringConverter(conversionService));
    converterRegistry.addConverter(new StringToArrayConverter(conversionService));

    converterRegistry.addConverter(new ArrayToObjectConverter(conversionService));
    converterRegistry.addConverter(new ObjectToArrayConverter(conversionService));

    converterRegistry.addConverter(new CollectionToStringConverter(conversionService));
    converterRegistry.addConverter(new StringToCollectionConverter(conversionService));

    converterRegistry.addConverter(new CollectionToObjectConverter(conversionService));
    converterRegistry.addConverter(new ObjectToCollectionConverter(conversionService));

    if (streamAvailable) {
        converterRegistry.addConverter(new StreamConverter(conversionService));
    }
}
```
在注册服务的，我们从名字也可以看出，提供了很多类似于 Array to Collection 方法，而这些Convert是没有具体的类型转换的，是负责将容器互相转换。
我们就举个ArrayToCollectionConverter的例子。

我们看看
```
org.springframework.core.convert.support.ArrayToCollectionConverter

public boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType) {
		return ConversionUtils.canConvertElements(
				sourceType.getElementTypeDescriptor(), targetType.getElementTypeDescriptor(), this.conversionService);
	}

```
ArrayToCollectionConverter的匹配方式是是单独的工具类。源码中的注释写的很清楚就不解释了。我们还看看如何转换的。

```java
public Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType) {
    if (source == null) {
        return null;
    }

    int length = Array.getLength(source);
    TypeDescriptor elementDesc = targetType.getElementTypeDescriptor();
    Collection<Object> target = CollectionFactory.createCollection(targetType.getType(),
            (elementDesc != null ? elementDesc.getType() : null), length);

    if (elementDesc == null) {
        for (int i = 0; i < length; i++) {
            Object sourceElement = Array.get(source, i);
            target.add(sourceElement);
        }
    }
    else {
        for (int i = 0; i < length; i++) {
            Object sourceElement = Array.get(source, i);
            Object targetElement = this.conversionService.convert(sourceElement,
                    sourceType.elementTypeDescriptor(sourceElement), elementDesc); //遍历调用
            target.add(targetElement);
        }
    }
    return target;
}
```
其实从某种角度上看，也应是是这样的，分为构造返回的Collections类型，然后遍历调用。这里就发现cache的重要性了，因为增加的缓存，我们在大规模的List转换的时候，查找只需要进行一次就可以了。

---

## 总结

Spring-Convert 虽然只是Spring 的一个小功能，但是其实这个使用的地方特别多，比如HttpRequest里面获得原始类型都是String，也就是通过ConvertService 转换成不同的Int，Long等等。
也算是阅读Spring源码的一点点收获。
