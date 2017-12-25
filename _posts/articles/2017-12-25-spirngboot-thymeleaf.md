---
layout: article
title: "Spring Boot（四）：Spring Boot 集成 Thymeleaf"
modified:
categories: SpringBoot
excerpt: "构建Spring Boot Web Apps 项目（四）"
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-12-25T18:08:00
---

Thymeleaf是一款用于渲染XML/XHTML/HTML5内容的模板引擎。类似JSP，Velocity，FreeMaker等，它也可以轻易的与Spring MVC等Web框架进行集成作为Web应用的模板引擎。与其它模板引擎相比，Thymeleaf最大的特点是能够直接在浏览器中打开并正确显示模板页面，而不需要启动整个Web应用，这是由于它支持 html 原型，然后在html 标签里增加额外的属性来达到模板+数据的展示方式。浏览器解释html 时会忽略未定义的标签属性，所以thymeleaf 的模板可以静态地运行；当有数据返回到页面时，Thymeleaf 标签会动态地替换掉静态内容，使页面动态显示。---摘自百度君。

要写出简洁优雅的前台代码，Spring Boot 君推荐使用Thymeleaf替代jsp。接下来咱们就看看如何利用Thymeleaf模板引擎写出优雅的html代码。

## 1.1 添加Maven依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```
可以查看依赖关系,发现spring-boot-starter-thymeleaf下面已经包括了spring-boot-starter-web,所以可以把spring-boot-starter-web的依赖去掉.

## 1.2 改造html代码
接下来我们以login登录界面来谈谈Thymeleaf模板引擎的使用方法, 
还是以上一篇博客《[Spring boot（三）：@RequestMapping之Form表单参数传递及POJO绑定实例讲解](http://www.onroad.tech/springboot/spirngboot-requestmapping-1/)》的工程项目代码为基础，改造一下我们的登录界面

```html
<!DOCTYPE html>
<html lang="zh-cn">
<head>
	<meta charset="utf-8"/>
	
	<title>Login</title>
	
	<link rel="stylesheet" href="/SpringBootBase/css/bootstrap.min.css"/>
	<link rel="stylesheet" href="/SpringBootBase/css/customer/login.css"/>
</head>

<body>
	<div class="container">
		<form class="form-signin" action="./login" method="post">
			<h2 class="form-signin-heading">请 登 录</h2>
			<input type="text" class="form-control" placeholder="账号" name="username"/>
			<input type="password" class="form-control" placeholder="密码" name="password"/> 
			<button class="btn btn-lg btn-primary btn-block" type="submit">登录</button>
		</form>
	</div>
</body>
</html>
```
### 1.2.1 xmlns:th命名空间
使用Thymeleaf引擎需要在html标签添加Thymeleaf模板引擎的命名空间：xmlns:th="http://www.thymeleaf.org"，这样的话才可以在其他标签里面使用th:*这样的语法.这是下面语法的前提。
### 1.2.2 th:href="@{...}"的使用
引用的绝对路径也可以用模板改造，“@{}”为引用静态资源文件，如：
```html
<link rel="stylesheet" href="/SpringBootBase/css/bootstrap.min.css"/>
<link rel="stylesheet" href="/SpringBootBase/css/customer/login.css"/>
```
可以用thymeleaf模板改写为@{}的形式表示
```html
<link rel="stylesheet" th:href="@{css/bootstrap.min.css}"/>
<link rel="stylesheet" th:href="@{css/customer/login.css}"/>
```
会引入/static目录下的/css/下的文件；
### 1.2.3 th:action="@{...}"的使用
表单POST的action url也可以用@{}表示：如
```html
<form class="form-signin" action="./login" method="post">
    ...
</form>
```
```html
<form class="form-signin" th:action="@{/login}" method="post">
    ...
</form>
```

于是改造完成后的代码就变成了这样：
```html
<!DOCTYPE html>
<html lang="zh-cn" xmlns:th="http://www.thymeleaf.org">
<head>
	<meta charset="utf-8"/>
	
	<title>Login</title>
	
	<link rel="stylesheet" th:href="@{css/bootstrap.min.css}"/>
	<link rel="stylesheet" th:href="@{css/customer/login.css}"/>
</head>

<body>
	<div class="container">
		<form class="form-signin" th:action="@{/login}" method="post">
			<h2 class="form-signin-heading">请 登 录</h2>
			<input type="text" class="form-control" placeholder="账号" name="username"/>
			<input type="password" class="form-control" placeholder="密码" name="password"/> 
			<button class="btn btn-lg btn-primary btn-block" type="submit">登录</button>
		</form>
	</div>
</body>
</html>
```
我们再来运行一下，得到如下结果：
![image](http://www.onroad.tech/images/20171225/01.png)
发现与我们上一篇的博客运行的结果不一致了，css样式找不到了，为什么会这样呢？
因为Spring Boot默认扫描的Thymeleaf的模板路径为resources/templates/,而此时我们的index.html在webapp目录下。
接下来我们将index.html移到resources/templates/目录下，并再写个controller，将根目录的访问都重定向到resources/templates/index.html
LoginController代码：
```java
@RequestMapping(value ="/", method = RequestMethod.GET)
String home() {
	return "index";
}
```
运行一下，结果正常
输入用户名及密码，后台也如期打印出
> POJO: tech.onroad.springbootbase.bean.UserVO, hash code: 1537182416, userame: admin, password: 123456


## 1.3 th:object="${...}"与th:field="*{...}"的使用
${...}用于获取变量值，对于javaBean的话使用变量名.属性名，如${user.name}
举个例子，如若需要将后台的值回显到前台，那应该怎么做，此时th:object="${...}"与th:field="*{...}"就派上用场了。
```html
<form class="form-signin" th:action="@{/login}" th:object="${user}" method="post">
	<h2 class="form-signin-heading">请 登 录</h2>
	<input type="text" class="form-control" placeholder="账号" th:field="*{username}"></input>
	<input type="password" class="form-control" placeholder="密码" th:field="*{password}"></input> 
	<button class="btn btn-lg btn-primary btn-block" type="submit">登录</button>
</form>
```
在表单代码中增加th:object属性，将name属性换成th:field属性，其中th:object定义表单数据提交对象user,用th:field定义表单数据属性,用*{}锁定上级定义的对象，{}内填写对象属性，提交表单时自动将属性值注入到对象中。
那前台如何知道th:object="${user}"与后台的哪个Java Bean对应呢？在controller代码里就可以看出端倪。
```java
@RequestMapping(value ="/", method = RequestMethod.GET)
String home(Model model, UserVO user) {
	model.addAttribute("user", user);
	return "index";
}

@RequestMapping(value = "login", method = RequestMethod.POST)
public String login(Model model, UserVO user){
	System.out.println("POJO: " + user.getClass().getName() + 
			", hash code: " + user.hashCode() + ", " + user.toString());
	model.addAttribute("user", user);
	return "index";
}
```
相比于前面的代码，我们多了model.addAttribute("user", user); 也就是说这个user对象是从后台通过Model传递过来的，当然也就能识别啦。
我们来运行一下，输入https://localhost:8443/SpringBootBase/得到如下界面：
![image](http://www.onroad.tech/images/20171225/02.png)

输入用户名admin及密码123456，点击登录，得到如下结果
![image](http://www.onroad.tech/images/20171225/03.png)

发现浏览器地址栏的URL已经变为https://localhost:8443/SpringBootBase/login，且账户名admin还被显示着，说明我们的前台已能正确接收后台传递过来的变量值了。至于密码栏为什么被清空，那是因为密码框这个input的type="password"所致。

Thymeleaf模板引擎还有非常多的语法，如判断条件th:if, th:unless;th:inline,...... 以后若需要详细介绍及demo，再慢慢更新。

最后建议在application.properties配置关闭thymeleaf缓存，因为Spring Boot使用thymeleaf时默认是有缓存的，即你把一个页面代码改了不会刷新页面的效果，你必须重新运行spring-boot的main()方法才能看到页面更改的效果。
```properties
spring.thymeleaf.cache: false
```

----

> 完整代码可到我的github下载:
>
> https://github.com/onroadtech/SpringbootBase/
>
> tag: [spring_boot_thymeleaf](https://github.com/onroadtech/SpringbootBase/tree/spring_boot_thymeleaf)
>
> commit-id: [35860470354212e14da0af993025b6555ff917ac](https://github.com/onroadtech/SpringbootBase/tree/35860470354212e14da0af993025b6555ff917ac)
