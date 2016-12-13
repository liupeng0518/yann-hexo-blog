---
title: spring-convert
date: 2016-12-13 16:43:18
tags: Spring Java
---

## PRE
**S -> Source**  
**T -> Target**

## KeySPI

### Converter
```java
package org.springframework.core.convert.converter;

public interface Converter<S, T> {
    T convert(S source);
}
```
从名称就能很轻易的看出，这是最基础的从 S -> T.  这点应该和Spring学习。对比我喜欢用A -> B 
的确不如Spring的 Soruce 和 Target 专业。细节处应该好好学(doge:)。大家都是写Java的，至于用法，就不用细说。

### ConverterFactory
```java
package org.springframework.core.convert.converter;

public interface ConverterFactory<S, R> {
    <T extends R> Converter<S, T> getConverter(Class<T> targetType);
}
```
这个从ConverterFactory貌似看不出来啥，核心是 \<T extends R> 这个返回值，说明最终得到的是 R的SubClass。用法可以参考 [StringToEnumConverterFactory](https://github.com/spring-projects/spring-framework/blob/bc14c5ba83e1f211628456bbccce7b2531aac58c/spring-core/src/main/java/org/springframework/core/convert/support/StringToEnumConverterFactory.java) 这个类.核心是因为传入T的实际类型，可以通过反射做些什么，大部分都用在枚举中，其他场景使用很少。

### GenericConverter
```java
package org.springframework.core.convert.converter;

public interface GenericConverter {
    public Set<ConvertiblePair> getConvertibleTypes();
    Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
}
```
GenericConverter如官网所言 *When you require a* **sophisticated**  *Converter implementation, consider the GenericConverter interface.*
当我们需要复杂些的实现的时候可以考虑，这个实现是多种转换的聚合。

### ConditionalGenericConverter
```java
public interface ConditionalGenericConverter
        extends GenericConverter, ConditionalConverter {
    boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType);
}
```
如名所见，满足Source 和 TargetType 的条件。至于 [TypeDescriptor](https://github.com/spring-projects/spring-framework/blob/e49813f2c4c6bb645c0990b3bd0fc290fc7c9f8e/spring-core/src/main/java/org/springframework/core/convert/TypeDescriptor.java)可以看作成S和T的包装类。


### ConversionService
```java
package org.springframework.core.convert;

public interface ConversionService {

    boolean canConvert(Class<?> sourceType, Class<?> targetType);
    <T> T convert(Object source, Class<T> targetType);
    boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);
    Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
}
```
终于我们得到了最终的SPI，从SPI中我们就看出来，前两个接口是普通的 S -> T，下面两个增加了ConditionalGenericConverter中的条件转换。

到这里Spring的核心SPI就结束了，官方文档也就说了这么多，那下面进行我们自己的探秘，文档不写的东西，让我们都去代码里面找答案。

## SPI 实现
[DefaultConversionService.java](https://github.com/spring-projects/spring-framework/blob/b22a59a0c4ea118147dc45c563d68234b8692d97/spring-core/src/main/java/org/springframework/core/convert/support/DefaultConversionService.java)是Spring的默认标准实现，我们从这个类开始看起。

先立FLAG，根据我对源码的了解，肯定有一个类似于[keyCovertInstantConcurrentHashMap](https://github.com/yannxia/chameleon/blob/master/src/main/java/info/yannxia/java/chameleon/AbstractConvertFactory.java) 的东西作为一个核心容器。

[Flag](http://ww2.sinaimg.cn/large/759074fcjw1f0d87evnxdj21bc0qo0ws.jpg)

看代码，老司机告诉你们一个真理，从第一行往下看是不明智的，从调用端入口看，那才是明智之选。
所以我们先从convert这个最核心的方法看起来。

```java
public class GenericConversionService implements ConfigurableConversionService{
	public <T> T convert(Object source, Class<T> targetType) {
			Assert.notNull(targetType, "targetType to convert to cannot be null");
			return (T) convert(source, TypeDescriptor.forObject(source), TypeDescriptor.valueOf(targetType));
	}
}
```
这就是DefaultConversionService的Convert的入口，从这里，我们一步一步的深入核心，然后我们可以发现，代码的流向是  
convert() -> getConverter() -> this.converters.find(sourceType, targetType)

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
Bingo,我们最终发现一个非常重要的东西getRegisteredConverter()方法。  

```java
private GenericConverter getRegisteredConverter(TypeDescriptor sourceType,
				TypeDescriptor targetType, ConvertiblePair convertiblePair) {

			// Check specifically registered converters
			ConvertersForPair convertersForPair = this.converters.get(convertiblePair);
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
从代码上，我们就发现了，this.converters 和 this.globalConverters 这两个就是整个转换的核心所在。

