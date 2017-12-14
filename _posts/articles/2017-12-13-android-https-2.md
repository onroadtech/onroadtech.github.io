---
layout: article
title: "Android HTTPS之自签名证书认证（二）"
modified:
categories: Android
excerpt: "使用okhttp框架 双向认证"
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-12-13T12:00:00
---

# 使用okhttp框架 双向认证

在上一篇博客中《[Android HTTPS之自签名证书认证（一）](http://www.onroad.tech/android/android-https-1/)单向认证》，我们主要介绍了https单向认证，单向认证安全级基本上可以满足大部分应用场景，但仍有部分应用场景需要更高的安全级别，如涉及资金安全等场景，接下来在这一篇博客中，我们将在上一篇博客的基础上介绍一下如何进行双向认证，双向认证安全级别更高。

## 1.1 生成客户端证书库及证书

首先对于双向证书验证，也就是说，客户端也要有个“.p12文件”，服务器那边会同时有个“cer文件”与之对应。

我们在上一篇博客中已经生成了服务端证书及证书：server.p12和server.cer文件。

接下来我们依葫芦画瓢，生成客户端证书库及证书，我们命名为:client.p12，client.cer.

- 生成client端证书库

```
keytool -genkey -alias client -keyalg RSA -storetype PKCS12 -keysize 2048 -keystore E:/ssl/client.p12 -dname "CN=Liting Wang,OU=onroad,O=onroad,L=Xiamen,ST=Fujian,c=cn" -validity 3650 -storepass 123456 -keypass 123456
```

- 利用证书库client.p12来签发证书

```
keytool -export -alias client -file E:/ssl/client.cer -keystore E:/ssl/client.p12 -storepass 123456
```
## 1.2配置服务端

服务端的配置比较简单，跟单向认证差不多，不过需要添加些属性。

```xml
<Connector SSLEnabled="true" clientAuth="true"  
    			maxThreads="150" port="8443" 
    			protocol="org.apache.coyote.http11.Http11NioProtocol" 
    			scheme="https" secure="true" sslProtocol="TLS"
    			keystoreFile="conf/server.p12" keystorePass="123456"
    			truststoreFile="conf/client.cer"/>
```

比单向认证多了truststoreFile="conf/client.cer"，clientAuth属性被修改为true. 理论上值truststoreFile为我们的client.cer文件。但尝试启动服务器，会发生Invalid keystore format错误：

> java.io.IOException: Invalid keystore format
>
> ​	at sun.security.provider.JavaKeyStore.engineLoad(Unknown Source)
>
> ​	at sun.security.provider.JavaKeyStore$JKS.engineLoad(Unknown Source)
>
> ​	at sun.security.provider.KeyStoreDelegator.engineLoad(Unknown Source)
>
> ​	at sun.security.provider.JavaKeyStore$DualFormatJKS.engineLoad(Unknown Source)
>
> ​	at java.security.KeyStore.load(Unknown Source)
>
> ​	at org.apache.tomcat.util.net.jsse.JSSESocketFactory.getStore(JSSESocketFactory.java:451)
>
> ​	at org.apache.tomcat.util.net.jsse.JSSESocketFactory.getTrustStore(JSSESocketFactory.java:407)
>
> ​	at org.apache.tomcat.util.net.jsse.JSSESocketFactory.getTrustManagers(JSSESocketFactory.java:658)
>
> ​	at org.apache.tomcat.util.net.jsse.JSSESocketFactory.getTrustManagers(JSSESocketFactory.java:570)
>
> ​	at org.apache.tomcat.util.net.NioEndpoint.bind(NioEndpoint.java:360)
>
> ​	at org.apache.tomcat.util.net.AbstractEndpoint.init(AbstractEndpoint.java:742)
>
> ​	at org.apache.coyote.AbstractProtocol.init(AbstractProtocol.java:458)
>
> ​	at org.apache.coyote.http11.AbstractHttp11JsseProtocol.init(AbstractHttp11JsseProtocol.java:120)
>
> ​	at org.apache.catalina.connector.Connector.initInternal(Connector.java:960)
>
> ​	at org.apache.catalina.util.LifecycleBase.init(LifecycleBase.java:102)
>
> ​	at org.apache.catalina.core.StandardService.initInternal(StandardService.java:568)
>
> ​	at org.apache.catalina.util.LifecycleBase.init(LifecycleBase.java:102)
>
> ​	at org.apache.catalina.core.StandardServer.initInternal(StandardServer.java:851)
>
> ​	at org.apache.catalina.util.LifecycleBase.init(LifecycleBase.java:102)
>
> ​	at org.apache.catalina.startup.Catalina.load(Catalina.java:576)
>
> ​	at org.apache.catalina.startup.Catalina.load(Catalina.java:599)
>
> ​	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
>
> ​	at sun.reflect.NativeMethodAccessorImpl.invoke(Unknown Source)
>
> ​	at sun.reflect.DelegatingMethodAccessorImpl.invoke(Unknown Source)
>
> ​	at java.lang.reflect.Method.invoke(Unknown Source)
>
> ​	at org.apache.catalina.startup.Bootstrap.load(Bootstrap.java:310)
>
> ​	at org.apache.catalina.startup.Bootstrap.main(Bootstrap.java:484)

Invalid keystore format。说keystore的格式不合法。为什么会这样呢，我个人理解是truststoreFile的值应该是证书库，但我们给的是证书文件，所以我们可以尝试把client.cer导入证书库中:

```java
keytool -import -alias client -file E:/ssl/client.cer -keystore E:/ssl/sever_trust.p12 storepass 123456
```
接下里修改server.xml为：

```xml
<Connector SSLEnabled="true" clientAuth="true"  
    			maxThreads="150" port="8443" 
    			protocol="org.apache.coyote.http11.Http11NioProtocol" 
    			scheme="https" secure="true" sslProtocol="TLS"
    			keystoreFile="conf/server.p12" keystorePass="123456"
    			truststoreFile="conf/sever_trust.p12"/>
```

启动，发现警报解除。

此时用浏览器访问我们的服务器，发现已经无法访问了，显示基于证书的身份验证失败。

如图所示：

![image](http://www.onroad.tech/images/20171212/02.png)

## 1.3 Android端配置

与server.p12对应，client.p12应该要用于我们Android端。在上一篇博客中，我们在支持https单向认证的时候调用了这么两行代码：

```java
sslContext.init(null, trustManagerFactory.getTrustManagers(), 
    new SecureRandom());
mOkHttpClient.setSslSocketFactory(sslContext.getSocketFactory());
```

注意sslContext.init的第一个参数我们传入的是null，第一个参数的类型实际上是KeyManager[] km,主要就用于管理我们客户端的key。



所以我们对setCertificates()进行改写，增加了客户端的证书库传递参数，为了区别了上一个方法，我将其重命令为setTwoWayCertificates().

```java
/******************************
*  双向认证
******************************/
public void setTwoWayCertificates(InputStream clientcertificates, InputStream... certificates)
{
    try {
        CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        keyStore.load(null);
        int index = 0;
        for (InputStream certificate : certificates) {
            String certificateAlias = Integer.toString(index++);
            keyStore.setCertificateEntry(certificateAlias, certificateFactory.generateCertificate(certificate));

            try {
                if (certificate != null)
                    certificate.close();
            } catch (IOException e) {
            }
        }
        SSLContext sslContext = SSLContext.getInstance("TLS");
        TrustManagerFactory trustManagerFactory = TrustManagerFactory.
                getInstance(TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(keyStore);

        //初始化keystore
        KeyStore clientKeyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        clientKeyStore.load(clientcertificates, "123456".toCharArray());

        KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        keyManagerFactory.init(clientKeyStore, "123456".toCharArray());

        sslContext.init(keyManagerFactory.getKeyManagers(), trustManagerFactory.getTrustManagers(), new SecureRandom());
        mOkHttpClient.setSslSocketFactory(sslContext.getSocketFactory());
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```



核心代码其实就是：

```java
KeyStore clientKeyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        clientKeyStore.load(clientcertificates, "123456".toCharArray());

        KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        keyManagerFactory.init(clientKeyStore, "123456".toCharArray());

        sslContext.init(keyManagerFactory.getKeyManagers(), trustManagerFactory.getTrustManagers(), new SecureRandom());
```

再次修改Application调用代码：

```java
public class Android4HttpsApplication extends Application{
    @Override
    public void onCreate() {
        super.onCreate();
        try {
            //单向认证
            /*OkHttpClientManager.getInstance()
                    .setOneWayCertificates(getAssets().open("server.cer"));*/
            //双向认证
            OkHttpClientManager.getInstance()
                    .setTwoWayCertificates(getAssets().open("client.bks"),getAssets().open("server.cer"));
        } catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

运行，报错：

> 12-12 08:53:31.226 4117-4117/? E/MainActivity: java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.

为什么呢？

因为我们生成的证书格式为.p12，但是android平台只识别bks格式的证书文件，因此找不到证书路径。

所以需要将我们的client.p12转成.bks格式

网上有很多转换工具，本文采[Portecle](http://sourceforge.net/projects/portecle/files/)下载[Download portecle-1.9.zip (3.4 MB)](http://sourceforge.net/projects/portecle/files/latest/download?source=files)。

双击打开portecle.jar，打开client.p12，Tools --> Change Keystore Type --> Bks将其转换为bks格式，输入密码保存生成client.bks,

![image](http://www.onroad.tech/images/20171212/03.png)

![image](http://www.onroad.tech/images/20171212/04.png)

将client.bks复制到Android project的assets目录下，

再次运行得到如下log

> 12-12 09:10:19.720 19220-19220/? D/MainActivity: Response is Hello world!

说明已可以正常访问，表示我们的双向认证成功。



参考：

- [Android Https相关完全解析 当OkHttp遇到Https](http://blog.csdn.net/lmj623565791/article/details/48129405)

- [https证书生成、服务器配置、Android端配置](http://tonysun3544.iteye.com/blog/2265448)



- <https://stackoverflow.com/questions/30745342/javax-net-ssl-sslpeerunverifiedexception-hostname-not-verified>


----

> 完整代码可到我的github下载:
>
> server: <https://github.com/onroadtech/SpringbootBase/tree/springboot_https_self_signed_certificate_one_two_way_certificate>
>
> android:https://github.com/onroadtech/Android4HTTPS/tree/https_two_way_certification
