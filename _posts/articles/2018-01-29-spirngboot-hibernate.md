---
layout: article
title: "Spring Boot（六）：Spring Boot 集成 hibernate & JPA"
modified:
categories: SpringBoot
excerpt: "构建Spring Boot Web Apps 项目（六）"
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2018-01-29T17:31:00
---

转眼间，2018年的十二分之一都快过完了，忙于各类事情，博客也都快一个月没更新了。今天我们继续来学习Springboot对象持久化。

首先JPA是Java持久化API，定义了一系列对象持久化的标准，而hibernate是当前非常流行的对象持久化开源框架，Spring boot就默认集成了这种框架，加速web应用开发。

## 1. 创建数据库

Hibernate 可以自动帮我们创建表，但不能帮我们创建数据库，所以说创建数据库及为数据库指定用户和权限，这是我们必须要做的工作。本文用的是MySQL数据库管理系统软件。

a) 首先用root进入MySQL;

```mysql
mysql -hlocalhost -uroot -proot;
```

其中:

> -h 后面跟着MySQL主机名称，我们这里将MySQL安装在本地，所以用localhost;
>
> -u 后面跟着要访问MySQL的用户名，安装成功后，默认会有root用户，该用户拥有访问整个数据库的所有权限；
>
> -p 后面跟着该用户的密码，root用户的密码由安装MySQL里指定，如若未指定，MySQL会随机生成一个密码，可在C:\Users\\{username}\AppData\Roaming\si.data 里查看。

b) 新创建一个数据库

登录MySQL成功后，我们可以用我们的超级用户root为我们的项目创建一个新的database.

```mysql
create database springbootbase;
```

执行上面的sql脚本，即可创建一个名为springbootbase新的空数据库。

c) 为新数据库springbootbase新增一张表

理论上，hibernate可以根据JPA POJO为我们自动生成我们想要的表，但是我个人觉得还是自己手动创建会更好一点，可以自己理解设计我们的关系型数据，理清各张表的关联情况。

接下来，我们来创建一张新表user

```mysql
create table user (
  id int(11) auto_increment not null primary key, 
  username varchar(255) not null, 
  password varchar(255) not null, 
  role enum('ADMIN', 'USER') not null default 'USER');
```

该表有4个字段：id, username, password, role. 其中id类型为int(11), 自增，非空，主键；username, password类型为varchar(255), 非空；role类型为枚举，非空，默认值为USER。

d) 创建MySQL新用户并授权

由于我们不能直接让我们的web应用直接使用MySQL root超级用户，因为如果让web应用直接使用root账号，这样会导致诸多风险，比如我们原本设计web应用只能访问springbootbase数据库，但root却也可以访问其它database，这样可能会导致其它信息暴露；再比如root可以删除某个数据库的权限，如果不使用不当，将某个数据库删除，那将悔之晚矣。因为我们有必要为web应用单独创建用户，并授予相应的权限来保证我们的数据库安全。

```mysql
create user 'onroad'@'localhost' identified by 'onroadtech';
```

该sql表示创建一个用户名为onroad，密码为onroadtech的新用户。

```mysql
grant select, insert, update on springbootbase.user to 'onroad'@'localhost';
```

该sql表示授权onroad用户对数据库springbootbase具有查询，插入及更新操作。一般普通用户授权这三个权限即可满足大部分web应用开发需求。

到此，我们的数据库已搭建成功。

## 2. Spring boot 集成hibernate & JPA

我们还是以[SpringBootBase](https://github.com/onroadtech/SpringbootBase/)这个项目为基础来集成hibernate & JPA：

### 2.1. 添加Maven依赖

由于Spring boot默认已经集成了Hibernate, 所在我们只需在pom.xml引用jpa及mysql连接库.

```xml
<dependency>  
  <groupId>org.springframework.boot</groupId>  
  <artifactId>spring-boot-starter-data-jpa</artifactId>  
</dependency>  

<dependency>  
  <groupId>mysql</groupId>  
  <artifactId>mysql-connector-java</artifactId>  
</dependency>
```

### 2.2 在application.properties里配置数据库地址及相应的JPA

```xml
#----------------------database-------------------------
spring.datasource.url = jdbc:mysql://localhost:3306/springbootbase?useUnicode=true&characterEncoding=utf8
spring.datasource.username = onroad
spring.datasource.password = onroadtech
spring.datasource.driverClassName = com.mysql.jdbc.Driver

#----------------------JPA------------------------------
# Specify the DBMS
spring.jpa.database = MYSQL
# Show or not log for each sql query
spring.jpa.show-sql = true
# Hibernate ddl auto (create, create-drop, update)
spring.jpa.hibernate.ddl-auto = update
# Naming strategy
spring.jpa.hibernate.naming-strategy = org.hibernate.cfg.DefaultNamingStrategy

# stripped before adding them to the entity manager
spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.MySQL5Dialect
```

### 2.3 创建一个实体类UserVO与之对应

```java
@Entity
@Table(name = "user") //引入@Table注解，name赋值为表名
public class UserVO {
	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	@Column(name = "id", nullable = false)
	private Long id;
	private String username;
	private String password;
	private String role;

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

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

	public String getRole() {
		return role;
	}

	public void setRole(String role) {
		this.role = role;
	}

	@Override
	public String toString() {
		return "userame: " + username + ", password: " + password + "role: " + role;
	}

}
```

### 2.4 新建UserVORepository接口继承CrudRepository，实现对数据库的增删改查

```java
public interface UserVORepository extends CrudRepository<UserVO, Long>{
  
    @Query("select u from UserVO u where u.username=:username")
    public UserVO findUserByName(@Param("username") String username);
  
  	public UserVO findUserByUsername(String username);
    
}
```

其实上面这两个接口是等价的，查询的结果也是一样的，只是表达方式不一样。

### 2.5 前端增加一个注册页面register.html

```html
<!DOCTYPE html>
<html lang="zh-cn" xmlns:th="http://www.thymeleaf.org">
<head>
	<meta charset="utf-8"/>
	
	<title>Register</title>
	
	<link rel="stylesheet" th:href="@{css/bootstrap.min.css}"/>
	<link rel="stylesheet" th:href="@{css/customer/login.css}"/>
</head>

<body>
	<h3 align="center">注册</h3> 
	<div class="container">
		<form class="form-signin" th:action="@{/reg}" th:object="${user}" method="post">
			<input type="text" class="form-control" placeholder="账号" th:field="*{username}"/>
			<input type="password" class="form-control" placeholder="密码" th:field="*{password}"/>
			<p th:if="${param.success}" class="error-code">注册成功</p>
			<p th:if="${param.error}" class="error-code">用户名已被注册</p>
			<button class="btn btn-lg btn-primary btn-block" type="submit">注册</button>
		</form>
	</div>
</body>
</html>
```

### 2.6 首页增加一个超链接到register页面

```html
<h2 class="form-signin-heading">请 登 录     <a th:href="@{/register}">注册</a></h2>
```

当然这个链接请求也不能拦截， 所以需要在WebSecurityConfig类中的configure()方法加上：

```java
http
	.authorizeRequests()
	.antMatchers("/", "/register", "/reg").permitAll()
```

### 2.7 再增加个RegisterController响应register请求

```java
@Controller
public class RegisterController {
	@Autowired UserVORepository userRepo;

	@RequestMapping(value ="/register", method = RequestMethod.GET)
	public String register(Model model, UserVO user) {
		model.addAttribute("user", user);
		return "register";
	}
	
	@RequestMapping(value ="/reg")
	public String addUser(Model model, UserVO user) {
		model.addAttribute("user", user);
		//UserVO isNewUser = userRepo.findUserByName(user.getUsername());
		UserVO isNewUser = userRepo.findUserByUsername(user.getUsername());
        //判断该用户名是否被注册过
		if (null == isNewUser) {
			userRepo.save(user);
			return "redirect:register?success";
		} else {
			return "redirect:register?error";
		}
	}
}
```

点击首页注册链接，跳转到注册页面，输入注册用户名及密码，点击注册，调用UserVORepository接口进行查询，如果注册的用户不存在，则新增并且返回成功，如果已存在，则返回错误。

## 3. 验证

访问https://localhost:8443/SpringBootBase/得到如下界面：

![image](http://www.onroad.tech/images/20180129/2018012901.png)

点击注册链接，跳转到注册页面：

![image](http://www.onroad.tech/images/20180129/2018012902.png)

输入账户名及密码，点击注册，若成功，则显示如下图

![image](http://www.onroad.tech/images/20180129/2018012903.png)

若该用户名已被注册，则提示对应的错误信息。

![image](http://www.onroad.tech/images/20180129/2018012904.png)

进入springbootbase数据库查看该注册用户信息是否存到user表里

```mysql
select * from user;
```

![image](http://www.onroad.tech/images/20180129/2018012905.png)

说明我们的数据已经正确写到数据库的user表里了。

----

> 完整代码可到我的github下载:
> [https://github.com/onroadtech/SpringbootBase/](https://github.com/onroadtech/SpringbootBase/)
> branch: master
> commit-id: [2af95a8a65ea3ed069502c58c026c4d8719eff99](https://github.com/onroadtech/SpringbootBase/tree/2af95a8a65ea3ed069502c58c026c4d8719eff99)