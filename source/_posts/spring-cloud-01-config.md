---
title: spring cloud 01 - config 
date: 2017-03-01 15:27:07
tags: spring-cloud
---

为什么要写这一个系列的博客，主要是记录在自己学习Spring cloud中思考，倘若能够给大家一点点帮助，就更好了，写这个系列的话，抱着少即是多的心态，我们每一章讲的不会特别多，尽可能的每一章都比知识萃取的深一点。

---
#### PRE
本文写于 2017-03-01， 代码针对于 **Spring Cloud Camden.SR5** 版本。


## Spring Cloud Config
>Spring Cloud Config provides server and client-side support for externalized configuration in a distributed system.

如官网所言，Spring cloud config 为分布式系统提供 CS架构的配置服务。按照这么个说法，其实我们就明白了，Spring Cloud Config 必然分为 Client 和 Server。下面我们就分别初始化一个Demo项目进行进一步的学习。

## Config Server
- [Spring官网Demo](https://github.com/spring-cloud/spring-cloud-config)
- [作者编写的Demo](https://github.com/yannxia-self/spring-cloud-sample/tree/master/spring-cloud-config-server-sample)
- [Quick Intro to Spring Cloud Configuration](http://www.baeldung.com/spring-cloud-configuration)

具体的步骤这里就不说了，可以参考 Quick Intro to Spring Cloud Configuration 这个文章写的很详细，关于几点，接下来会细聊一下。


### Config Server如何选择配置文件
> Where do you want to store the configuration data for the Config Server? The strategy that governs this behaviour is the EnvironmentRepository
> 
- {application} maps to "spring.application.name" on the client side;
- {profile} maps to "spring.profiles.active" on the client (comma separated list); and
- {label} which is a server side feature labelling a "versioned" set of config files.


Config Server 是从哪里去读文件的，这里说明了，核心的是EnvironmentRepository这个接口
```java
public interface EnvironmentRepository {

	Environment findOne(String application, String profile, String label);

}
```
我们可以看出来，根据application+ profile+ label 就可以确定一个唯一的环境变量对象。正如上文所言application是 *客户端* 的spring.application.name的属性，profile 是 *客户端* 的spring.profiles.active属性，而 label 是 *服务端* 的版本概念。

在服务端的代码中，我们发现我们是配置了我们所有的配置所在的仓库地址的，比如：
```
spring.cloud.config.server.git.uri: file:///${user.home}/config-repo
```
在此处的含义我们存放配置的路径是 ~/config-repo ，那具体对应的文件名是什么？
很神奇的是在文档中我并没有找到说明 (针对编写时的文档)，而从网上得知，所对应的文件是：  
application+profile+.yml 等
倘若 application = foo， profile = dev， 那配置文件应该是 foo-dev.yml。
这段逻辑在Spring Cloud Config的文档中并无说明，这段逻辑在固有的Spring 文档有说明可以查看
 [24.4 Profile-specific properties](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config-profile-specific-properties) 在这段中说明，Spring的逻辑是这样的。

```java
@Override
	public Environment findOne(String config, String profile, String label) {
		SpringApplicationBuilder builder = new SpringApplicationBuilder(
				PropertyPlaceholderAutoConfiguration.class);
		ConfigurableEnvironment environment = getEnvironment(profile);
		builder.environment(environment);
		builder.web(false).bannerMode(Mode.OFF);
		if (!logger.isDebugEnabled()) {
			// Make the mini-application startup less verbose
			builder.logStartupInfo(false);
		}
		String[] args = getArgs(config, profile, label);
		// Explicitly set the listeners (to exclude logging listener which would change
		// log levels in the caller)
		builder.application()
				.setListeners(Arrays.asList(new ConfigFileApplicationListener()));
		ConfigurableApplicationContext context = builder.run(args);  ①
		environment.getPropertySources().remove("profiles");
		try {
			return clean(new PassthruEnvironmentRepository(environment).findOne(config,
					profile, label));
		}
		finally {
			context.close();
		}
	}
```
在 NativeEnvironmentRepository 中，我们发现这样的代码，我们在①处看见，其实Spring是在Config Server这段将我们传入的参数组装成一个普通启动的参数，去尝试自己去运行一个ApplicationContext，这样就是解释清楚了为什么这里的逻辑是SpringContext中的，所以我们发现一个简单的道理，其实Spring不仅仅是直接吧文件返回这么简单，自己是尝试使用 Spring固有的逻辑在服务器端就将配置解析成 Environment 这个类的。

那再深层次为何是这个文件，经过不懈的努力，在 ConfigFileApplicationListener 中发现。
org.springframework.boot.context.config.ConfigFileApplicationListener.Loader#load(java.lang.String, java.lang.String, org.springframework.boot.context.config.ConfigFileApplicationListener.Profile) 方法

```
private void load(String location, String name, Profile profile)
				throws IOException {
			String group = "profile=" + (profile == null ? "" : profile);
			if (!StringUtils.hasText(name)) {
				// Try to load directly from the location
				loadIntoGroup(group, location, profile);
			}
			else {
				// Search for a file with the given name
				for (String ext : this.propertiesLoader.getAllFileExtensions()) { ①
					if (profile != null) {
						// Try the profile-specific file
						loadIntoGroup(group, location + name + "-" + profile + "." + ext,
								null);
						for (Profile processedProfile : this.processedProfiles) {
							if (processedProfile != null) {
								loadIntoGroup(group, location + name + "-"
										+ processedProfile + "." + ext, profile); ②
							}
						}
						// Sometimes people put "spring.profiles: dev" in
						// application-dev.yml (gh-340). Arguably we should try and error
						// out on that, but we can be kind and load it anyway.
						loadIntoGroup(group, location + name + "-" + profile + "." + ext,
								profile);
					}
					// Also try the profile-specific section (if any) of the normal file
					loadIntoGroup(group, location + name + "." + ext, profile);
				}
			}
		}
```
我们可以从 ① 发现Spring所支持的后缀名："properties","xml","yml","yaml" ，而在②处我们发现就是按照 - 逻辑给拼接起来的。




