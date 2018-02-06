---
layout: single
title: Spring boot（三）：@RequestMapping之Form表单参数传递及POJO绑定实例讲解
permalink: /spring-boot-topic-request-mapping/
sidebar:
  nav: SpringBoot
toc: true
description:
excerpt: "构建Spring Boot Web Apps 项目（三）"
date: 2017-12-21
---

今天我们将谈谈前端form表单参数如何透过@RequestMapping与后台的Java Bean (POJO)绑定。

本文以前面两篇博客《[Spring boot （一）：快速构建Spring Boot 1.5.8 maven Web 项目](http://www.onroad.tech/spring-boot-topic-rapid-build-spring-boot/)》《[Spring boot（二）：HTTPS之自签名证书配置](http://www.onroad.tech/spring-boot-topic-config-https-spring-boot/)》为基础，代码基于[SpringBootBase](https://github.com/onroadtech/SpringbootBase)工程。

首先来写个简单的登录界面表单代码index.html，界面非常简单，两个输入框，一个登录按钮：

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

再来写个POJO视图对象UserVO
```java
public class UserVO {
	
	public String username;
	public String password;

	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	@Override
	public String toString() {
		return "username: " + username + ", password: " + password;
	}

}
```
再写个controller
```java
@Controller
public class LoginController {
	
	@RequestMapping(value = "login", method = RequestMethod.POST)
	public String login(UserVO user){
		System.out.println("POJO: " + user.getClass().getName() + 
				", hash code: " + user.hashCode() + ", " + user.toString());
		return "redirect:/";
	}

}
```
启动Tomcat service，在地址栏输入https://localhost:8443/SpringBootBase/ 显示如下界面，

![图1](http://www.onroad.tech/images/20171222/01.png)

输入账号admin及密码123456，在控制台可见输出如下log

> POJO: tech.onroad.springbootbase.bean.UserVO, hash code: 1036686981, username: admin, password: 123456

说明前端表单传过来的两个参数username, password会自动绑定到UserVO这个POJO上。其实这都是Spring @RequestMapping这个注解的功劳，它会自动扫描形参的POJO，并创建对象，如果前端传进来的参数与POJO成员变量名相同，会通过POJO的setter方法传给该对象。

---

## Q & A

### 1. 匹配参数里是否会忽略对象成员变量的大小写？

我们将UserVO的成员变量username改写为userName,即

```java
public class UserVO {
	
	private String userName;
	private String password;

	public String getUserName() {
		return userName;
	}

	public void setUserName(String userName) {
		this.userName = userName;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	@Override
	public String toString() {
		return "userName: " + userName + ", password: " + password;
	}

}
```
Controller代码不变

再次运行一下，同样输入账号admin及密码123456，得到的log为：

> POJO: tech.onroad.springbootbase.bean.UserVO, hash code: 1762238496, userName: null, password: 123456

userName的值为空，很显然在匹配过程中是**区分大小写**的。

### 2. 如果形参有两个POJO对象，它们刚好都有一个username的成员变量，那又会如何匹配呢？ 

同样我们通过代码来验证一下：

首先，我们增加一个StudentVO POJO, 该对象有一个成员变量也叫username,

```java
public class StudentVO {
	private String username;
	private int age;
	
	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	public int getAge() {
		return age;
	}
	
	public void setAge(int age) {
		this.age = age;
	}
	
	@Override
	public String toString() {
		return "username: " + username + ", age: " + age;
	}

}
```
controller login方法更改为
```java
@Controller
public class LoginController {
	
	@RequestMapping(value = "login", method = RequestMethod.POST)
	public String login(UserVO user, StudentVO student){
		System.out.println("POJO: " + user.getClass().getName() + 
				", hash code: " + user.hashCode() + ", " + user.toString());
		System.out.println("POJO: " + student.getClass().getName() + 
				", hash code: " + student.hashCode() + ", " + student.toString());
		return "redirect:/";
	}
}
```
运行， 同样输入账号admin及密码12345，得到如下log
> POJO: tech.onroad.springbootbase.bean.UserVO, hash code: 1260683413, userame: admin, password: 123456
> POJO: tech.onroad.springbootbase.bean.StudentVO, hash code: 816397424, username: admin, age: 0

说明只要形参的成员变量名与前端传递进来的参数名一致，都会通过setter方法将参数传给对应Java bean的成员变量。



----

> 完整代码可到我的github下载:
>
> https://github.com/onroadtech/SpringbootBase/
>
> branch: master
>
> commit-id: 94dd507dfc9c71e62e1b0f3b19f54843c583d84e

