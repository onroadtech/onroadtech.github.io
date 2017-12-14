---
layout: article
title: "Android HTTPS之自签名证书认证（三）"
modified:
categories: Android
excerpt: "Okhttp从2.4.0升级到3.9.1对HTTPS认证的影响"
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-12-14T15:02:00
---

# Okhttp从2.4升级到3.9.1对HTTPS认证的影响
严格意义上讲，本文不应属于这个系列，但由于前面两篇博客的代码是参考《Android Https相关完全解析 当OkHttp遇到Https》改写的，当时的okhttp框架的版本为2.4.0，但现在okhttp版本升级到了3.9.1，并且查了一下相关资料，发现okhttp从2.x到3.x版本的api变化比较大，因此我也尝试着将okhttp版本进行升级，并做简要记录与大家分享。
## 1.1 okhttp Jar包升级
将```compile 'com.squareup.okhttp:okhttp:2.4.0'```更新为```compile 'com.squareup.okhttp3:okhttp:3.9.1'```，Android Studio会自动下载3.9.1的okhttp jar包。
## 1.2 更新Api及修正相关编译错误
Rebuild project，会发现有许多错误
### 1.2.1 包名更新
Okhttp2.x的包名为com.squareup.okhttp
```java
import com.squareup.okhttp.Call;
import com.squareup.okhttp.Callback;
import com.squareup.okhttp.OkHttpClient;
import com.squareup.okhttp.Request;
import com.squareup.okhttp.Response;
```
但okhttp3.x已经变为okhttp3,如上面的包名则相对应变为：
```java
import okhttp3.Call;
import okhttp3.Callback;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
```
### 1.2.2 OkHttpClient创建方式不同
okhttp2.x直接new OkHttpClient，而okhttp3.x 中提供了Builder，很好的使用了创建者设计模式
```java
OkHttpClient.Builder okHttpClient = new OkHttpClient.Builder();
```
### 1.2.3 OkHttpClient参数的配置变化
之前okhttp2.x参数可以直接OkHttpClient.setConnectTimeout()设置，现在OkHttpClient使用创建者模式，需要在OkHttpClient.Builder上设置可配置的参数:
```java
okHttpClient.connectTimeout(5, TimeUnit.SECONDS);
okHttpClient.readTimeout(5, TimeUnit.SECONDS);
```
### 1.2.4 setCookieHandler变为cookieJar
okhttp2.x调用OkHttpClient的setCookieHandler方法,CookieHandler 的子类CookieManager实现了cookie的具体管理方法，
```java
mOkHttpClient.setCookieHandler(new CookieManager(null, CookiePolicy.ACCEPT_ORIGINAL_SERVER));
```
okhttp3中已经没有setCookieHandler方法了，而改成了cookieJar，需要在OkHttpClient的Builder的cookieJar方法中设置。
```java
okHttpClient.cookieJar(new CookieJar() {
    private final HashMap<HttpUrl, List<Cookie>> cookieStore = new HashMap<>();
    @Override
    public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
        cookieStore.put(url, cookies);
    }

    @Override
    public List<Cookie> loadForRequest(HttpUrl url) {
        return null;
    }
});
```
### 1.2.5 改造setSslSocketFactory
okhttp2.x的sslSocketFactory(SSLSocketFactory sslSocketFactory)已不推荐使用，取而代之的

```java
public Builder sslSocketFactory(SSLSocketFactory sslSocketFactory, X509TrustManager trustManager) 
```

因此我们可以将其改造为

```java
mOkHttpClient = new OkHttpClient.Builder()
    					.sslSocketFactory(sslContext.getSocketFactory(), trustManager)
    					.build();
```
trustManager是X509TrustManager 的一个实例
```java
trustManager = trustManagerForCertificates(certificates);
```
```java
private X509TrustManager trustManagerForCertificates(InputStream in)
    throws GeneralSecurityException {
    CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
    Collection<? extends Certificate> certificates = certificateFactory.generateCertificates(in);
    if (certificates.isEmpty()) {
        throw new IllegalArgumentException("expected non-empty set of trusted certificates");
    }

    // Put the certificates a key store.
    char[] password = "123456".toCharArray(); // Any password will work.
    KeyStore keyStore = newEmptyKeyStore(password);
    int index = 0;
    for (Certificate certificate : certificates) {
        String certificateAlias = Integer.toString(index++);
        keyStore.setCertificateEntry(certificateAlias, certificate);
    }

    // Use it to build an X509 trust manager.
    KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(
            KeyManagerFactory.getDefaultAlgorithm());
    keyManagerFactory.init(keyStore, password);
    TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(
            TrustManagerFactory.getDefaultAlgorithm());
    trustManagerFactory.init(keyStore);
    TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
    if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
        throw new IllegalStateException("Unexpected default trust managers:"
                + Arrays.toString(trustManagers));
    }
    return (X509TrustManager) trustManagers[0];
}
```
所以

```java
/******************************
 *  单向认证
 ******************************/
public void setOneWayCertificates(InputStream... certificates)
/******************************
*  双向认证
******************************/
public void setTwoWayCertificates(InputStream clientcertificates, InputStream... certificates)

```

可分别改造为：

```java
/******************************
 *  单向认证
 ******************************/
public void setOneWayCertificates(InputStream certificates){
    X509TrustManager trustManager;
    try{
        trustManager = trustManagerForCertificates(certificates);
        SSLContext sslContext = SSLContext.getInstance("TLS");
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        TrustManagerFactory trustManagerFactory =
                TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());

        trustManagerFactory.init(keyStore);
        sslContext.init(
                null,
                new TrustManager[] { trustManager },
                new SecureRandom());
        mOkHttpClient = new OkHttpClient.Builder()
                .sslSocketFactory(sslContext.getSocketFactory(), trustManager)
                .build();
    } catch (Exception e){
        Log.e("OkHttpClientManager", e.getMessage());
    }
}

/******************************
*  双向认证
******************************/
public void setTwoWayCertificates(InputStream clientCertificates, InputStream certificates) {
    X509TrustManager trustManager;
    try {
        trustManager = trustManagerForCertificates(certificates);

        SSLContext sslContext = SSLContext.getInstance("TLS");

        //初始化keystore
        KeyStore clientKeyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        clientKeyStore.load(clientCertificates, "123456".toCharArray());

        KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        keyManagerFactory.init(clientKeyStore, "123456".toCharArray());

        sslContext.init(
                keyManagerFactory.getKeyManagers(),
                new TrustManager[] { trustManager },
                new SecureRandom());
        mOkHttpClient = new OkHttpClient.Builder()
                .sslSocketFactory(sslContext.getSocketFactory(), trustManager)
                .build();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### 1.2.6 Callback实现的接口和call的变化

okhttp2.x的callback方法是```void onFailure(Request request, IOException e);void onResponse(Response response) throws IOException;```  okhttp3.x 的Callback方法为```void onFailure(Call call, IOException e);void onResponse(Call call, Response response) throws IOException;```okhttp3对Call做了更简洁的封装，okhttp3.x Call是个接口，okhttp的call是个普通class，一定要注意，无论哪个版本，call都不能执行多次，多次执行需要重新创建。于是

```java
private void deliveryResult(final ResultCallback callback, Request request) {
    mOkHttpClient.newCall(request).enqueue(new Callback() {
        @Override
        public void onFailure(Request request, IOException e) {
            sendFailedStringCallback(request, e, callback);
        }

        @Override
        public void onResponse(Response response) throws IOException {
            try {
                final String string = response.body().string();
                if (callback.mType == String.class) {
                    sendSuccessResultCallback(string, callback);
                } else {
                    Object o = mGson.fromJson(string, callback.mType);
                    sendSuccessResultCallback(o, callback);
                }
            } catch (IOException e) {
                sendFailedStringCallback(response.request(), e, callback);
            } catch (JsonParseException e) {  //Json解析的错误
                sendFailedStringCallback(response.request(), e, callback);
            }
        }
    });
}
```
就变为：

```java
private void deliveryResult(final ResultCallback callback, Request request) {
    mOkHttpClient.newCall(request).enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {
            sendFailedStringCallback(call.request(), e, callback);
        }

        @Override
        public void onResponse(Call call, Response response) throws IOException {
            try {
                final String string = response.body().string();
                if (callback.mType == String.class) {
                    sendSuccessResultCallback(string, callback);
                } else {
                    Object o = mGson.fromJson(string, callback.mType);
                    sendSuccessResultCallback(o, callback);
                }
            } catch (IOException e) {
                sendFailedStringCallback(response.request(), e, callback);
            } catch (JsonParseException e) {  //Json解析的错误
                sendFailedStringCallback(response.request(), e, callback);
            }
        }
    });
}
```

至此，改造完毕，我们分验证单向认证及双向认证均可得到如下log：

> 12-14 02:08:20.698 30391-30391/? D/MainActivity: Response is Hello world!

证明已可以正常通讯，说明我们okhttp3.x版本升级成功。

参考：

- <https://github.com/square/okhttp/blob/3f7a3344a4c85aa3bbb879dabac5ee625ab987f3/samples/guide/src/main/java/okhttp3/recipes/CustomTrust.java#L54>
- [okhttp3与旧版本okhttp的区别分析](http://blog.csdn.net/robertcpp/article/details/51330447)
- [okhttp3 使用详解及简单封装](http://blog.csdn.net/qq_33463102/article/details/60765056)

----

> 完整代码可到我的github下载:
>
> server: <https://github.com/onroadtech/SpringbootBase/tree/springboot_https_self_signed_certificate_one_two_way_certificate>
>
> android:https://github.com/onroadtech/Android4HTTPS/tree/88aa11a9b224df1fd19a5120ff387e81dcd23867

