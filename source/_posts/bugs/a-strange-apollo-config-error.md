---
title: 记一次奇怪的Apollo配置中心错误
date: 2018-05-20 15:04:08
tags:
categories: ['java', 'apollo']
---

{% raw %}

<script src="https://cdn.bootcss.com/webfont/1.6.28/webfontloader.js"></script>
<script src="https://cdn.bootcss.com/snap.svg/0.5.1/snap.svg-min.js"></script>
<script src="https://cdn.bootcss.com/underscore.js/1.9.0/underscore-min.js"></script>
<script src="https://cdn.bootcss.com/raphael/2.2.7/raphael.min.js"></script>
<script src="https://cdn.bootcss.com/js-sequence-diagrams/1.0.6/sequence-diagram-min.js"></script>

{% endraw %}

<!-- more -->

## 背景

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:apollo="http://www.ctrip.com/schema/apollo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/mvc
       http://www.springframework.org/schema/mvc/spring-mvc.xsd
       http://www.ctrip.com/schema/apollo
       http://www.ctrip.com/schema/apollo.xsd">

	<apollo:config namespaces="mallcenter" order="-99999"/>

    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="configLocation" value="/WEB-INF/sqlMapConfig.xml"/>
        <property name="mapperLocations" value="classpath*:mybatis/*mapper.xml"/>
        <property name="typeAliasesPackage" value="cn.htd.zeus.mall.biz.dmo"/>
    </bean>

    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="cn.daocloud.**.dao"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    </bean>

    <bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
        <constructor-arg index="0" ref="sqlSessionFactory"/>
    </bean>

    <bean id="dataSource" class="cn.htd.zeus.mall.common.util.XBasicDataSource"
          destroy-method="close">
        <property name="driverClassName" value="${dataSource.driverClassName}"/>
        <property name="url" value="${dataSource.url}"/>
        <property name="username" value="${dataSource.username}"/>
        <property name="password" value="${dataSource.password}"/>
        <property name="initialSize" value="${dataSource.initialSize}"/>
    </bean>

 </beans>
```

启动 项目的时候会报错  `${dataSource.initialSize}` 不能转为 `int`，这个很容易知道是因为我们没有获取到配置 `dataSource.initialSize` 下面就漫长的分析之路。

## 第一回合：怀疑 apollo 没有成功注册PostProcessBeanFactory

因为这个项目是War项目，找个Tomcat镜像，war放进去RemoteDebug一下，我们知道 Apollo 是
`com.ctrip.framework.apollo.spring.config.ConfigPropertySourcesProcessor` 在这个类装载的时候会注册

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesPlaceholderConfigurer.class.getName(), PropertySourcesPlaceholderConfigurer.class);
        BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloAnnotationProcessor.class.getName(), ApolloAnnotationProcessor.class);
        BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueProcessor.class.getName(), SpringValueProcessor.class);
        BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloJsonValueProcessor.class.getName(), ApolloJsonValueProcessor.class);
        this.processSpringValueDefinition(registry);
    }
```

 果不其然，还没有运行到这里的时候系统已经挂掉了，那问题可以大概确认，apollo 并没有起作用。

 ## 第二回合：确认Spring在哪个阶段挂掉了

 ```java
 // Allows post-processing of the bean factory in context subclasses.
   postProcessBeanFactory(beanFactory);

   // Invoke factory processors registered as beans in the context.
   invokeBeanFactoryPostProcessors(beanFactory);

   // Register bean processors that intercept bean creation.
   registerBeanPostProcessors(beanFactory);

   // Initialize message source for this context.
   initMessageSource();

   // Initialize event multicaster for this context.
   initApplicationEventMulticaster();

   // Initialize other special beans in specific context subclasses.
   onRefresh();

   // Check for listener beans and register them.
   registerListeners();

   // Instantiate all remaining (non-lazy-init) singletons.
   finishBeanFactoryInitialization(beanFactory);

   // Last step: publish corresponding event.
   finishRefresh();
 ```
 上面的代码大家应该很熟悉，我们都知道Spring的顺序，通过Debug，我们发现Apollo在 `invokeBeanFactoryPostProcessors(beanFactory)` 阶段已经挂掉了，那问题到这里就有点清晰了，我们的Apollo作用的范围在 `invokeBeanFactoryPostProcessors`之后，那问题就来了，为什么我们的 `datasource` 对象在 `invokeBeanFactoryPostProcessors `阶段初始化？


## 第三回合：抓住凶手
上个Section我们知道，`invokeBeanFactoryPostProcessors` 系统挂掉，那为什么我们的 `datasource` 会在 `invokeBeanFactoryPostProcessors` 运行，从注释中，我们可以看出来这个阶段其实在注册 `factory processors`,通过Debug，我们发现了在注册完 Apollo之后，又注册了 `MapperScannerConfigurer`，`SqlSessionFactoryBean`, `XBasicDataSource`, DataSource 和 SqlSessionFactoryBean的关系很容易发现，写着呢，那 MapperScannerConfigurer 查看Google

>注 意 , 没 有 必 要 去 指 定 SqlSessionFactory 或 SqlSessionTemplate , 因 为 MapperScannerConfigurer 将会创建 MapperFactoryBean,之后自动装配。 ------<第六章 注入映射器>
>

那我们明白，因为我们的注入了一个非必要的Bean在注册PostBeanFactory导致的失败。


{% raw %}

<div id="diagram"></div>
<script>
	var data =
	['Title: 此次错误启动时序图',
	 'Refresh()->invokeBeanFactoryPostProcessors(): 1.注册factory processors',
	 'invokeBeanFactoryPostProcessors()->MapperScannerConfigurer(): 2.构建MapperScannerConfigurer',
	 'MapperScannerConfigurer()->SqlSessionFactoryBean(): 3.构建SqlSessionFactoryBean',
	 'SqlSessionFactoryBean()->XBasicDataSource(): 4.构建XBasicDataSource',
	 'Note left of XBasicDataSource(): 此阶段的ApolloConfig没有成功注入',
	 ].join('\n');
  	var diagram = Diagram.parse(data);
  	diagram.drawSVG("diagram", {theme: 'simple'});
</script>
{% endraw %}
