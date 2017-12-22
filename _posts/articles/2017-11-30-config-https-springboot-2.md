---
layout: article
title: "Spring boot（二）：HTTPS之自签名证书配置"
modified:
categories: SpringBoot
excerpt: "构建Spring Boot Web Apps 项目（二）"
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-11-30T13:52:00
---



其实为Spring Boot配置HTTPS非常简单，只需要两步：

- 生成或者获取ssl证书
- 在Spring Boot中配置HTTPS
  - 内嵌Tomcat配置
  - Local Tomcat 配置



## 1. 生成或者获取ssl证书

获取SSL证书主要有两种，一种是自己通过工具自动生成，另外一种是通过SSL证书服务商获取，当然了后一种是需要收费的。

本文主要介绍用JDK自带的keytool工具来生成SSL证书。如何查看我们的JDK是否带有keytool工具，只需要windows 控制台输入如下命令

```java
keytool --help
```

若输出如下信息，则证明此JDK版本带有keytool工具。

![image](http://www.onroad.tech/images/20171130/01.png)



接下来开始生成证书，在命令行终端输入

```java
keytool -genkey -alias tomcat  -storetype PKCS12 -keyalg RSA -keysize 2048  -keystore keystore.p12 -validity 3650
```

简单说明一下各个参数的含义

>1.-storetype 指定密钥仓库类型 
>
>2.-keyalg 生证书的算法名称，RSA是一种非对称加密算法 
>
>3.-keysize 证书大小 
>
>4.-keystore 生成的证书文件的存储路径 
>
>5.-validity 证书的有效期

当然了，我们也可以不指定密钥仓库类型，JDK会默认选用JKS, 如

```java
keytool -genkeypair -alias tomcat -keyalg RSA -keysize 2048 -keystore keystore.jks -validity 3650
```

两种都可以，但Oracle推荐使用第一种.

```
C:\Users\liting.wang>keytool -genkey -alias tomcat  -storetype PKCS12 -keyalg RS
A -keysize 2048  -keystore keystore.p12 -validity 3650
输入密钥库口令:
再次输入新口令:
您的名字与姓氏是什么?
  [Unknown]:
您的组织单位名称是什么?
  [Unknown]:
您的组织名称是什么?
  [Unknown]:
您所在的城市或区域名称是什么?
  [Unknown]:
您所在的省/市/自治区名称是什么?
  [Unknown]:
该单位的双字母国家/地区代码是什么?
  [Unknown]:
CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown是否正确?
  [否]:  是

```

执行完上面一行命令后，在你的系统的当前用户目录下会生成一个keystore.p12文件

我们也可以使用下面的命令查看我们证书内容

```
keytool -list -v -storetype pkcs12 -keystore keystore.p12
```

```
C:\Users\liting.wang>keytool -list -v -storetype pkcs12 -keystore keystore.p12
输入密钥库口令:

密钥库类型: PKCS12
密钥库提供方: SunJSSE

您的密钥库包含 1 个条目

别名: tomcat
创建日期: 2017-11-29
条目类型: PrivateKeyEntry
证书链长度: 1
证书[1]:
所有者: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
发布者: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
序列号: 6b628be1
有效期开始日期: Wed Nov 29 18:52:33 CST 2017, 截止日期: Sat Nov 27 18:52:33 CST
2027
证书指纹:
         MD5: D6:71:F9:ED:11:A7:8A:AB:69:DC:9A:99:E2:4B:94:CE
         SHA1: 88:E5:A7:1C:77:5E:D9:83:F4:14:76:D2:E3:1F:31:FB:37:29:31:13
         SHA256: D6:52:D8:56:A7:07:E6:99:3D:EB:BB:E8:D5:E7:E1:D0:66:B6:AD:D5:BC:
65:01:9D:60:6D:95:BA:CD:63:75:C0
         签名算法名称: SHA256withRSA
         版本: 3

扩展:

#1: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: 31 84 6B E8 DB 63 6B 5E   63 CE A9 DA C8 72 82 88  1.k..ck^c....r..
0010: EB F9 71 5B                                        ..q[
]
]



*******************************************
*******************************************
```

如果你已经有SSL证书，你也可将其导入到keystore里，供Spring Boot使用

```
keytool -import -alias tomcat -file myCertificate.crt -keystore keystore.p12 -storepass password
```



## 2. 在Spring Boot里配置开记HTTPS

### 2.1 内嵌Tomcat配置

将这个文件拷贝到我们项目的resources目录下，与application.properties同一目录，然后修改application.properties文件，添加HTTPS支持。在application.properties中添加如下代码：

```properties
# Define a custom port instead of the default 8080
server.port = 8443
# Tell Spring Security (if used) to require requests over HTTPS
security.require-ssl=true
# The format used for the keystore 
server.ssl.key-store-type:PKCS12
# The path to the keystore containing the certificate
server.ssl.key-store=classpath:keystore.p12
# The password used to generate the certificate
server.ssl.key-store-password=password
# The alias mapped to the certificate
server.ssl.key-alias=tomcat
```

第一行指定https请求访问的端口;

第二行是告诉Spring Security 请求也需要透过HTTPS, 签名文件;

第三行指定密钥仓库类型;

第四行指定密钥仓库路径;

第五行指定签名密码;

第六行是别名。

OK，这样配置完成之后我们就可以通过HTTPS来访问我们的Web了.

### 2.2 Local Tomcat配置

以上是针对Spring Boot内嵌Tomcat，如果Local的Tomcat，那应该如何配置呢？

首先到你的Tomcat安装目录的conf目录下找到server.xml配置文件，打开https connector，即将

```xml
<!--
	<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
		maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
		clientAuth="false" sslProtocol="TLS" />
-->
```

去掉注释并修改为

```xml
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" 
               keystoreFile="${user.home}/keystore.p12"
               keystorePass="123456" />
```

其中 keystoreFile="${user.home}/keystore.p12"为SSL证书的路径，keystorePass="123456"为你证书的密码。

以上为Tomcat8.0的配置。
Tomcat9.0跟Tomcat8.0有点不太一样，但原理是一样的，将

```
<!--
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="150" SSLEnabled="true">
    <SSLHostConfig>
        <Certificate certificateKeystoreFile="conf/localhost-rsa.jks"
                     type="RSA" />
    </SSLHostConfig>
</Connector>
-->
```
去掉注释并修改为：
```
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true">
    <SSLHostConfig>
        <Certificate certificateKeystoreFile="conf/keystore.p12"
        			 certificateKeystorePassword="123456"
        			 certificateKeyAlias="tomcat"
                     type="RSA" />
    </SSLHostConfig>
</Connector>
```
其实也就是换了个表达形式而已，换汤不换药。具体可参考TOMCAT9.0的官方配置文档<http://tomcat.apache.org/tomcat-9.0-doc/ssl-howto.html>。

重启Tomcat Service，如果有看到输出Starting ProtocolHandler ["http-nio-8443"]的log信息，则说明配置成功

![image](http://www.onroad.tech/images/20171130/02.png)

这样就可以通过HTTPS来访问我们的Web了



## 3. HTTP自动重定向到HTTPS

光有HTTPS肯定还不够，很多用户可能并不知道，用户有可能继续使用HTTP来访问你的网站，这个时候我们需要添加HTTP自动重定向到HTTPS的功能，当用户使用HTTP来进行访问的时候自动转为HTTPS的方式。

### 3.1 内嵌Tomcat配置

配置很简单，在入口类中添加相应的重定向Bean就行了，如下：

```java
import org.apache.catalina.Context;
import org.apache.catalina.connector.Connector;
import org.apache.tomcat.util.descriptor.web.SecurityCollection;
import org.apache.tomcat.util.descriptor.web.SecurityConstraint;
import org.springframework.boot.context.embedded.EmbeddedServletContainerFactory;
import org.springframework.boot.context.embedded.tomcat.TomcatEmbeddedServletContainerFactory;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ConnectorConfig {

	@Bean
	public EmbeddedServletContainerFactory servletContainer() {
		TomcatEmbeddedServletContainerFactory tomcat = new TomcatEmbeddedServletContainerFactory() {
			@Override
			protected void postProcessContext(Context context) {
				SecurityConstraint securityConstraint = new SecurityConstraint();
				securityConstraint.setUserConstraint("CONFIDENTIAL");
				SecurityCollection collection = new SecurityCollection();
				collection.addPattern("/*");
				securityConstraint.addCollection(collection);
				context.addConstraint(securityConstraint);
			}
		};
		tomcat.addAdditionalTomcatConnectors(getHttpConnector());
		return tomcat;
	}

	private Connector getHttpConnector() {
		Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
		connector.setScheme("http");
		connector.setPort(8080);
		connector.setSecure(false);
		connector.setRedirectPort(8443);
		return connector;
	}

}
```

这个时候当我们访问http://localhost:8080/SpringBootBase的时候系统会自动重定向到https://localhost:8443/SpringBootBase这个地址上。这里的Connector实际就是server.xml中配置的Tomcat的Connector节点。

### 3.2 Local Tomcat 配置

同样在conf目录下，打开web.xml，在最后一个的节点后加上

```xml
<security-constraint>
  <web-resource-collection>
    <web-resource-name>OPENSSL</web-resource-name>
    <url-pattern>/*</url-pattern>
  </web-resource-collection>
  <user-data-constraint>
    <transport-guarantee>CONFIDENTIAL</transport-guarantee>
  </user-data-constraint>
</security-constraint>
```

重启Tomcat Service即可，这个时候当我们访问http://localhost:8080/SpringBootBase的时候Tomcat也会将其自动重定向到https://localhost:8443/SpringBootBase这个地址上.



----

> 完整代码可到我的github下载:
>
> <https://github.com/onroadtech/SpringbootBase/tree/spring_boot_https>
