---
layout: article
title: "Android HTTPS之自签名证书认证（一）"
modified:
categories: Android
excerpt: "使用okhttp框架 单向认证"
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-12-12T14:19:00
---

# 使用okhttp框架 单向认证

鉴于技术或者框架的不断更新，网上介绍的Android客户端访问自签名证书的网站的博客很多已无法使用，故本博文将详细介绍一下Android SSL/TLS HTTPS请求。

开发环境：

- JDK 1.8

- Tomcat 8.0

- IDE: Android Studio 2.3.3


## 1.1 生成自签名证书

- 生成server端证书库

```java
keytool -genkey -alias server -keyalg RSA -storetype PKCS12 -keysize 2048 -keystore E:/ssl/server.p12 -validity 3650 -storepass 123456 -keypass 123456
```

>C:\Users\liting.wang>keytool -genkey -alias server -keyalg RSA -storetype PKCS12-keysize 2048 -keystore E:/ssl/server.p12 -validity 3650 -storepass 123456 -keypass 123456
>
>您的名字与姓氏是什么?
>
>[Unknown]: Liting Wang
>
>您的组织单位名称是什么?
>
>\[Unknown]: Onroad
>
>您的组织名称是什么?
>
>\[Unknown]: Onroad
>
>您所在的城市或区域名称是什么?
>
>\[Unknown]: Xiamen
>
>您所在的省/市/自治区名称是什么?
>
>\[Unknown]: Fujian
>
>该单位的双字母国家/地区代码是什么?
>
>\[Unknown]: CN
>
>CN=Liting Wang, OU=Onroad, O=Onroad, L=Xiamen, ST=Fujian, C=CN是否正确?
>
>\[否]: y

填入各类信息，也可以放空，最后输入y, 即可在E:/ssl/目录下生成server.p12证书库

当然了，如果你觉得一个个的输入比较麻烦，也可以用命令行的方式表达，如下

```java
keytool -genkey -alias server -keyalg RSA -storetype PKCS12 -keysize 2048 -keystore E:/ssl/server.p12 -dname "CN=Liting Wang,OU=onroad,O=onroad,L=Xiamen,ST=Fujian,c=cn" -validity 3650 -storepass 123456 -keypass 123456
```

>解释：
>
>-genkey：生成key
>
>-alias：别名，独一无二，通常不区分大小写
>
>-keyalg： 指定密钥的算法（如 RSA,  DSA（如果不指定默认采用DSA））
>
>-storetype：指定密钥仓库类型，无指定默认为JKS，但oracle建议使用PKCS12
>
>-keysize ：指定证书大小
>
>-keystore：指定密钥库的名称，可指定路径，例如：E:/ssl/ 需要注意的是运行该命令之前需要先创建好该目录。
>
>-validity：证书的有效期，单位：天。



- 利用Server.p12来签发证书

```java
keytool -export -alias server -file E:/ssl/server.cer -keystore E:/ssl/server.p12 -storepass 123456 
```

生成的server.cer证书是给客户端使用的。

## 1.2  Tomcat 配置

找到Tomcat安装目录的tomcat 8.0/conf/下找到server.xml配置文件，并以文本形式打开

在Service标签中，加入：

```xml
<Connector SSLEnabled="true" acceptCount="100" clientAuth="false" 
    disableUploadTimeout="true" enableLookups="true" keystoreFile="conf/server.p12" keystorePass="123456" maxSpareThreads="75" 
    maxThreads="200" minSpareThreads="5" port="8443" 
    protocol="org.apache.coyote.http11.Http11NioProtocol" scheme="https" 
    secure="true" sslProtocol="TLS"
      /> 
```

单向认证：clientAuth须为false。keystoreFile的值为我们刚才生成的server.p12, 我们将其到conf/目录下，keystorePass值为密钥库密码:123456。

启动Tomcat 8.0 service，(Server端代码请参考<https://github.com/onroadtech/SpringbootBase/tree/acc0324d5b38ec698c7c87e0d20f6fcb07f82454/>)在浏览器地址栏里输入https://localhost:8443/即可看到证书不可信任的警告，选择继续访问, 如下图所示：

![image](http://www.onroad.tech/images/20171212/01.png)

如果在这个Tomcat部署了项目，如上一篇的《[Spring Boot之HTTPS配置](http://www.onroad.tech/springboot/config-https-springboot-2/)》介绍的SpringbootBase项目，按如下的URL访问http://ip:8443/SpringBootBase/进行访问，得到不安全链接，选择继续访问，得到的结果为：Hello world!

我们Android端就是要利用server端的证书server.cer访问https://ip:8443/SpringBootBase/，如果也能得到上述结果，就可以说明我们的https单向认证成功。

## 1.3 Android 端开发

对于自签名的网站的访问，网上部分的做法是直接设置信任所有的证书，但这种做法肯定会存在安全隐患的，下面我们介绍如何使用okhttp框架让OkHttpClient去信任我们自己签名的证书

首先把第一节生成的server.cer放到assets文件夹下，其实你可以随便放哪，反正能读取到就行，然后在OkHttpClientManager里面添加如下的方法：

```java
public void setCertificates(InputStream... certificates)
{
  try
  {
    CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
    KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
    keyStore.load(null);
    int index = 0;
    for (InputStream certificate : certificates)
    {
      String certificateAlias = Integer.toString(index++);
      keyStore.setCertificateEntry(certificateAlias, certificateFactory.generateCertificate(certificate));

      try
      {
        if (certificate != null)
          certificate.close();
      } catch (IOException e)
      {
      }
    }

    SSLContext sslContext = SSLContext.getInstance("TLS");

    TrustManagerFactory trustManagerFactory =
      TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());

    trustManagerFactory.init(keyStore);
    sslContext.init
      (
      null,
      trustManagerFactory.getTrustManagers(),
      new SecureRandom()
    );
    mOkHttpClient.setSslSocketFactory(sslContext.getSocketFactory());
  } catch (Exception e)
  {
    e.printStackTrace();
  }
}
```

代码内部，我们：

- 构造CertificateFactory对象，通过它的generateCertificate(is)方法得到Certificate。
- 然后将得到的Certificate放入到keyStore中。
- 接下来利用keyStore去初始化我们的TrustManagerFactory
- 由trustManagerFactory.getTrustManagers获得TrustManager[]初始化我们的SSLContext
- 最后，设置我们mOkHttpClient.setSslSocketFactory即可。

这样就完成了我们代码的编写，当客户端进行SSL连接时，就可以根据我们设置的证书去决定是否该服务端网站是不是证书所俯的网址。

记得在Application中进行初始化：

```java
public class Android4HttpsApplication extends Application{
    @Override
    public void onCreate() {
        super.onCreate();
        try
        {
            OkHttpClientManager.getInstance()
                    .setCertificates(getAssets().open("server.cer"));
        } catch (IOException e)
        {
            e.printStackTrace();
        }
    }
}
```

然后尝试以下代码访问我们自己搭建的服务器：

```java
private void postTest() {
    OkHttpClientManager.getAsyn("https://192.168.0.101:8443/SpringBootBase/", new OkHttpClientManager.ResultCallback<String>()
    {

        @Override
        public void onError(com.squareup.okhttp.Request request, Exception e) {
            Log.e(TAG, e.getMessage());
        }

        @Override
        public void onResponse(String u)
        {
            Log.d(TAG,"Response is " + u);
        }
    });
}
```

运行，发现报如下错误

> 12-11 13:32:18.092 22181-22181/? E/MainActivity: Hostname 192.168.0.101 not verified:
>
> ​                                                     certificate: sha1/iY0cn+EnYvx3fnRugIsLx4sFJEE=
>
> ​                                                     DN: CN=Liting Wang,OU=onroad,O=onroad,L=Xiamen,ST=Fujian,C=cn
>
> ​                                                     subjectAltNames: []

出现这个错误的原因是：如果请求的主机是一个IP, 而“CN”又没有去匹配这个IP地址，就会报这个错误。所以一般"CN"填写的是主机绑定的域名，而不是keytool提示的名和姓。但是如果我们没有申请域名或者我们的主机外网无法访问，那应该如何处理。其实keytool还有另外一个参数- ext SAN=IP,可以用于指定访问的主机IP地址，当然了也可以指定域名-ext SAN=dns:xxxxxxxx,ip:xxx.xxx.xxx.xxx

重新用如下命令生成server端证书库及导出证书

```java
keytool -genkey -alias server -keyalg RSA -storetype PKCS12 -keysize 2048 -keystore E:/ssl/server.p12 -dname "CN=Liting Wang,OU=onroad,O=onroad,L=Xiamen,ST=Fujian,c=cn" -validity 3650 -storepass 123456 -keypass 123456 -ext SAN=IP:192.168.0.101
```

```java
keytool -export -alias server -file E:/ssl/server.cer -keystore E:/ssl/server.p12 -storepass 123456
```

将生成的server.p12及server.cer重新分别复制到Tomcat conf目录下及Android project的Assets目录下，重启Tomcat service及运行android app， 即可得到如下log

> 12-12 02:18:06.704 3407-3407/? D/MainActivity: Response is Hello world!

说明我们的https单向验证成功。



参考：

- [Android Https相关完全解析 当OkHttp遇到Https](http://blog.csdn.net/lmj623565791/article/details/48129405)

- [https证书生成、服务器配置、Android端配置](http://tonysun3544.iteye.com/blog/2265448)



- <https://stackoverflow.com/questions/30745342/javax-net-ssl-sslpeerunverifiedexception-hostname-not-verified>


----

> 完整代码可到我的github下载:
>
> server: <https://github.com/onroadtech/SpringbootBase/tree/acc0324d5b38ec698c7c87e0d20f6fcb07f82454/>
>
> android:https://github.com/onroadtech/Android4HTTPS/tree/https_one_way_certification
