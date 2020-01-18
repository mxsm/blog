---
title: 详解SpringBoot内嵌Tomcat的自动装配(SpringBoot 2.2.3.RELEASE)
categories:
  - Spring
  - SpringBoot
tags:
  - Spring
  - SpringBoot
  - 2.2.3.RELEASE
abbrlink: 872f9884
date: 2020-01-17 11:00:00
---
### 什么是自动装配
在SpringBoot源码中有一个spring-boot-autoconfigure模块,在这个模块中做了很多默认的配置。也就是我们俗称的“自动装配”。  
在SpringBoot的自动装配会根据添加的依赖，自动加载依赖相关的配置属性并启动依赖，自动装配的底层原理： **spring的条件注解@Conditional来实现** 。
### 为什么要自动装配
利用自动装配模式代替XML配置模式，比如使用SpringMVC时，需要配置组件扫描、调度器、试图解析器等，现在有了自动装配，这些都可以不用配置了，SpringBoot默认已经帮我们配置好了。利用内嵌的Tomcat通过自动装配就不需要配置外置的Tomcat，减少配置的麻烦。**说白了自动装配就是减少了配置。**
### SpringBoot Tomcat自动装配详解

我们会从下面四个方面去分析Tomcat的自动装配。

- **SpringBoot项目配置**
- **SpringBoot项目启动**
- **SpringBoot Tomcat如何装配**
- **SpringBoot Tomcat如何启动**

#### SpringBoot项目配置

在[SpringBoot官网](https://docs.spring.io/spring-boot/docs/2.2.3.RELEASE/reference/html/getting-started.html#getting-started)的pom.xml文件配置：

```xml


<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>myproject</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <!-- Inherit defaults from Spring Boot -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.3.RELEASE</version>
    </parent>

    <!-- Override inherited settings -->
    <description/>
    <developers>
        <developer/>
    </developers>
    <licenses>
        <license/>
    </licenses>
    <scm>
        <url/>
    </scm>
    <url/>

    <!-- Add typical dependencies for a web application -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <!-- Package as an executable jar -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

其实上面的主要有用的就两个配置：

```xml
   <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.3.RELEASE</version>
    </parent>
```

这个配置就是设置SpringBoot的版本，还有就是另外一个配置：

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```

这个就是最重要的。这个依赖实现了自动装配，我们接着往下看这个依赖的源码。

在SpringBoot项目的有一个 **spring-boot-starters** 模块，通过源码可以看到这个模块里面都是pom.xml的pom文件，那么我们来看一下 **spring-boot-starter-web** 模块的pom文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starters</artifactId>
		<version>${revision}</version>
	</parent>
	<artifactId>spring-boot-starter-web</artifactId>
	<name>Spring Boot Web Starter</name>
	<description>Starter for building web, including RESTful, applications using Spring
		MVC. Uses Tomcat as the default embedded container</description>
	<properties>
		<main.basedir>${basedir}/../../..</main.basedir>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-json</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation</artifactId>
			<exclusions>
				<exclusion>
					<groupId>org.apache.tomcat.embed</groupId>
					<artifactId>tomcat-embed-el</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
		</dependency>
	</dependencies>
</project>

```

通过上面可以看出来导入了 **spring-web** 、 **spring-webmvc** 。同时还导入了 **spring-boot-starter-tomcat** 、 **spring-boot-starter** 依赖。 

> 从这里可以看出来SpringBoot默认的启动容器是tomcat

看一下 **spring-boot-starte**r 的pom文件

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starters</artifactId>
		<version>${revision}</version>
	</parent>
	<artifactId>spring-boot-starter</artifactId>
	<name>Spring Boot Starter</name>
	<description>Core starter, including auto-configuration support, logging and YAML</description>
	<properties>
		<main.basedir>${basedir}/../../..</main.basedir>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-autoconfigure</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-logging</artifactId>
		</dependency>
		<dependency>
			<groupId>jakarta.annotation</groupId>
			<artifactId>jakarta.annotation-api</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-core</artifactId>
		</dependency>
		<dependency>
			<groupId>org.yaml</groupId>
			<artifactId>snakeyaml</artifactId>
			<scope>runtime</scope>
		</dependency>
	</dependencies>
</project>
```

通过上面可以看出来导入一个重要的模块 **spring-boot-autoconfigure** 这个模块里面包含了所有的代码。

> 通过 **spring-boot-autoconfigure** 模块可以看出来，所有的SpringBoot starter的代码都在这个模块，对于不是web项目的SpringBoot项目，只需要引入：
>
> ```xml
> <dependency>
>  <groupId>org.springframework.boot</groupId>
>  <artifactId>spring-boot-starter</artifactId>
> </dependency>
> ```

看一下 **spring-boot-starter-tomcat**  的pom文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starters</artifactId>
		<version>${revision}</version>
	</parent>
	<artifactId>spring-boot-starter-tomcat</artifactId>
	<name>Spring Boot Tomcat Starter</name>
	<description>Starter for using Tomcat as the embedded servlet container. Default
		servlet container starter used by spring-boot-starter-web</description>
	<properties>
		<main.basedir>${basedir}/../../..</main.basedir>
	</properties>
	<dependencies>
		<dependency>
			<groupId>jakarta.annotation</groupId>
			<artifactId>jakarta.annotation-api</artifactId>
		</dependency>
		<dependency>
			<groupId>org.apache.tomcat.embed</groupId>
			<artifactId>tomcat-embed-core</artifactId>
			<exclusions>
				<exclusion>
					<groupId>org.apache.tomcat</groupId>
					<artifactId>tomcat-annotations-api</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.apache.tomcat.embed</groupId>
			<artifactId>tomcat-embed-el</artifactId>
		</dependency>
		<dependency>
			<groupId>org.apache.tomcat.embed</groupId>
			<artifactId>tomcat-embed-websocket</artifactId>
		</dependency>
	</dependencies>
</project>
```

而这个里面的导入的是内嵌的Tomcat。这里就分析完成了在SpringBoot的配置pom文件配置导入了一些什么东西。

#### SpringBoot项目启动

看一下我们一般SpringBoot项目启动的代码写法：

```java
@SpringBootApplication
public class RaftApplication{
    public static void main(String[] args) {
        SpringApplication.run(RaftApplication.class, args);
    }
}
```

通过 **main** 方法来启动，之前有一篇 《[SpringBoot启动分析](https://blog.ljbmxsm.com/pages/f2f8b808/)》的文章。讲解了SpringBoot是如何启动的，在另外一篇文章中 《[SpringBoot源码解析之autoconfigure](https://blog.ljbmxsm.com/pages/beb882ec/)》讲解了自动装配的底层原理。而在之前的项目配置引入依赖引入了 **<artifactId>spring-boot-autoconfigure</artifactId>** 这个模块。通过查看该模块的 **spring.factories** 的数据有一个自动配置的有这样一个配置：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
```

这个类揭示了Tomcat如何自动装配。

#### SpringBoot Tomcat如何装配

研究一下 **`ServletWebServerFactoryAutoConfiguration`** 类的源码。探索Tomcat如何自动装配

> 注意：自动装配的底层原理根据的是Spring框架的@Conditional注解

```java
@Configuration(proxyBeanMethods = false)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
	//省略代码

}
```

**@ConditionalOnClass(ServletRequest.class)** 注解判断classpath中是否有 **ServletRequest** 类。

**@ConditionalOnWebApplication(type = Type.SERVLET)** 判断是否为servlet。

**@EnableConfigurationProperties(ServerProperties.class)** 加载 **ServerProperties** 配置。

```java
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
```

这一段导入了 **BeanPostProcessorsRegistrar** 和 三个web运行容器：

- **Tomcat**

  [Tomcat官网](http://tomcat.apache.org/)

- **Jetty**

  [Jetty官网](https://www.eclipse.org/jetty/)

- **Undertow**

  [Undertow官网](http://undertow.io/)

在 **ServletWebServerFactoryAutoConfiguration** 有三个方法一个静态内部类。

```java
@Bean
public ServletWebServerFactoryCustomizer servletWebServerFactoryCustomizer(ServerProperties serverProperties) {
	return new ServletWebServerFactoryCustomizer(serverProperties);
}
```

**servletWebServerFactoryCustomizer** 方法创建一个ServletWebServerFactoryCustomizer(定制器)。

```java
@Bean
@ConditionalOnClass(name = "org.apache.catalina.startup.Tomcat")
public TomcatServletWebServerFactoryCustomizer tomcatServletWebServerFactoryCustomizer(
			ServerProperties serverProperties) {
	return new TomcatServletWebServerFactoryCustomizer(serverProperties);
}
```

**tomcatServletWebServerFactoryCustomizer** 方法根据 **ConditionalOnClass** 存在 Tomcat那么创建 **TomcatServletWebServerFactoryCustomizer** 定制器。

```java
@Bean
	@ConditionalOnMissingFilterBean(ForwardedHeaderFilter.class)
	@ConditionalOnProperty(value = "server.forward-headers-strategy", havingValue = "framework")
	public FilterRegistrationBean<ForwardedHeaderFilter> forwardedHeaderFilter() {
		ForwardedHeaderFilter filter = new ForwardedHeaderFilter();
		FilterRegistrationBean<ForwardedHeaderFilter> registration = new FilterRegistrationBean<>(filter);
		registration.setDispatcherTypes(DispatcherType.REQUEST, DispatcherType.ASYNC, DispatcherType.ERROR);
		registration.setOrder(Ordered.HIGHEST_PRECEDENCE);
		return registration;
	}
```

----还没分析完
