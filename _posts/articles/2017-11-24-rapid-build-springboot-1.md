---
layout: article
title: "快速构建Spring Boot 1.5.8 maven Web 项目"
modified:
categories: SpringBoot
excerpt: "构建Spring Boot web 项目(一)"
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-11-24T16:49:00
---



Springboot提供了快速构建maven项目的方法，具体可以参考spring boot的官方文档<http://docs.spring.io/spring-boot/docs/1.5.8.RELEASE/reference/htmlsingle/>

下面我们就一步步从头开始构建我们的Spring Boot maven项目：

## 1. Spring Initialize

 从springboot 官网<http://start.spring.io/>快速构建我们项目的原始代码. Spring boot 支持Maven Project 和 Gradle Project, 编程语言支持Java, Kotlin 和Groovy, 目前spring boot release 的稳定版本为1.5.8.

 本文选用的是Maven project, 开发语言为Java, Spring Boot 版本为1.5.8；填写Project Metadata信息，及项目所依赖的库，一般如果你创建的是Web应用，Web这个库肯定是要选的。如图： 

![img](http://www.onroad.tech/images/20171124/01.png)

也可以打开Switch to the full version, 配置更详细的project metadata.

![img](http://www.onroad.tech/images/20171124/02.png)

 点击Generate Project按钮就可以下载配置好的spring boot web 项目源码了，这就是我们的Spring Boot初始代码。

## 2. 将源码导入IDE/eclipse

更新下载maven依赖库，并编译spring boot项目有两种方法：

- 在windows 控制台下直接运行项目源码里的mvnw.cmd脚本;
- eclipse 右击项目，选择Maven–>Update Project;

编译完成后，我们会发现项目中多了个target目录，这个目录是用来存放编译后的class 文件及我们最终需要生成的执行文件。如下图：

![这里写图片描述](http://www.onroad.tech/images/20171124/03.png)

 resources是用来放本web项目的资源文件，static文件夹用来存放静态资源文件，如js库，图片，插件等等， templates用来存放html模板等，application.properties用来配置该项目的一些属性及引入的框架配置。

## 3. 添加Controller代码

打开SpringBootBaseApplication.java, 增加一个Controller映射，

```java
@RequestMapping("/")
String home() {
	return "Hello world!";
}
```

整个文件代码变为：

```java
package tech.onroad.springbootbase;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@SpringBootApplication
public class SpringBootBaseApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootBaseApplication.class, args);
	}

	@RequestMapping("/")
	String home() {
		return "Hello world!";
	}

}
```



 @RestController这个注解用于告诉Spring，当它在处理web request时，SpringBootBaseApplication 这个类是web Controller;

 @RequestMapping(“/”)这个注解告诉Spring, 当web 发过来的http根目录(如：localhost:8080/SpringBootBase/)请求时，映射到home()这个方法;

 重新编译发布我们的web项目， 启动server，看到eclipse 控制台输出如下信息，即表明我们的spring boot项目启动成功 。

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.5.8.RELEASE)

2017-11-23 21:04:13.076  INFO 10184 --- [ost-startStop-1] t.o.springbootbase.ServletInitializer    : Starting ServletInitializer v0.0.1-SNAPSHOT on XMNB4003247 with PID 10184 (started by liting.wang in C:\Users\liting.wang\Desktop)
2017-11-23 21:04:13.076  INFO 10184 --- [ost-startStop-1] t.o.springbootbase.ServletInitializer    : No active profile set, falling back to default profiles: default
2017-11-23 21:04:13.263  INFO 10184 --- [ost-startStop-1] ationConfigEmbeddedWebApplicationContext : Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@1705953c: startup date [Thu Nov 23 21:04:13 CST 2017]; root of context hierarchy
2017-11-23 21:04:14,763 [localhost-startStop-1] INFO  org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/SpringBootBase] - Initializing Spring embedded WebApplicationContext
2017-11-23 21:04:14.763  INFO 10184 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1500 ms
2017-11-23 21:04:15.902  INFO 10184 --- [ost-startStop-1] o.s.b.w.servlet.ServletRegistrationBean  : Mapping servlet: 'dispatcherServlet' to [/]
2017-11-23 21:04:15.902  INFO 10184 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'errorPageFilter' to: [/*]
2017-11-23 21:04:15.902  INFO 10184 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'characterEncodingFilter' to: [/*]
2017-11-23 21:04:15.902  INFO 10184 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
2017-11-23 21:04:15.902  INFO 10184 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'httpPutFormContentFilter' to: [/*]
2017-11-23 21:04:15.902  INFO 10184 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'requestContextFilter' to: [/*]
2017-11-23 21:04:16.699  INFO 10184 --- [ost-startStop-1] s.w.s.m.m.a.RequestMappingHandlerAdapter : Looking for @ControllerAdvice: org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@1705953c: startup date [Thu Nov 23 21:04:13 CST 2017]; root of context hierarchy
2017-11-23 21:04:16.855  INFO 10184 --- [ost-startStop-1] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/]}" onto java.lang.String tech.onroad.springbootbase.SpringBootBaseApplication.home()
2017-11-23 21:04:16.871  INFO 10184 --- [ost-startStop-1] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error]}" onto public org.springframework.http.ResponseEntity<java.util.Map<java.lang.String, java.lang.Object>> org.springframework.boot.autoconfigure.web.BasicErrorController.error(javax.servlet.http.HttpServletRequest)
2017-11-23 21:04:16.871  INFO 10184 --- [ost-startStop-1] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error],produces=[text/html]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.boot.autoconfigure.web.BasicErrorController.errorHtml(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse)
2017-11-23 21:04:16.922  INFO 10184 --- [ost-startStop-1] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-11-23 21:04:16.922  INFO 10184 --- [ost-startStop-1] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-11-23 21:04:17.035  INFO 10184 --- [ost-startStop-1] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**/favicon.ico] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-11-23 21:04:17.456  INFO 10184 --- [ost-startStop-1] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
2017-11-23 21:04:17.487  INFO 10184 --- [ost-startStop-1] t.o.springbootbase.ServletInitializer    : Started ServletInitializer in 5.644 seconds (JVM running for 9.493)
2017-11-23 21:04:17,534 [main] INFO  org.apache.coyote.http11.Http11NioProtocol - Starting ProtocolHandler ["http-nio-8080"]
2017-11-23 21:04:17,549 [main] INFO  org.apache.coyote.ajp.AjpNioProtocol - Starting ProtocolHandler ["ajp-nio-8009"]
2017-11-23 21:04:17,565 [main] INFO  org.apache.catalina.startup.Catalina - Server startup in 8104 ms
```
 在浏览器输入<http://localhost:8080/SpringBootBase/>若能看到Hello World!信息，说明我们基于spring boot 1.5.8框架创建的web应用已经能正常运行了。

![img](http://www.onroad.tech/images/20171124/04.png)

----

> 完整代码可到我的github下载:
>
> <https://github.com/onroadtech/SpringbootBase/tree/spring_boot_base_version>
