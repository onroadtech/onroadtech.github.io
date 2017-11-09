---
layout: article
title: "跨进程间如何进行AIDL IPC 通信（一）"
modified:
categories: articles
excerpt: "AIDL是Android跨进程间的一种非常重要的IPC通信机制."
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-11-9T10:45:14-13:00
---



AIDL是Android跨进程间的一种非常重要的IPC通信机制，今天我们将来详细讲述如何不同app之间如何进行AIDL IPC通信。

## 1. 创建service端工程

由于Android Studio刚刚发布3.0版本，所以我们就用它来尝个鲜。
可用原来android studio 版本点击菜单Help-->Check for Updates... 之后Android studio就会自动下载更新包进行升级，可惜我升级半天，升级期间报几个冲突，结果升级失败，自动回滚，导致原来的AS2.3.3无法启动，无奈只能直接下载AS3.0手动安装。

### a) 创建AIDL Service端工程AidlServerDemo

创建完后工程项目结构如下

![image](http://www.onroad.tech/images/2017110901.PNG)

### b) 创建AIDL文件

右击Java目录 New --> AIDL --> AIDL File
输入接口名称IOnroad
此时会在与java同一目录下生成aidl文件夹及对应的包名和aidl文件

![image](http://www.onroad.tech/images/2017110902.PNG)

AS自动帮我们生成了一个基本类型的aidl接口

```
void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
```

但我们并不打算用它，我们自己来定义一个简单的基本类型

```
String sayHello(String name, int age);
```

点击菜单Build-->Make Project,即可在对应目录下生成Java接口文件
![image](http://www.onroad.tech/images/2017110903.PNG)

### c) 创建一个service来实现这个接口类

```
public class OnroadService extends Service {
    public OnroadService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        // TODO: Return the communication channel to the service.
        throw new UnsupportedOperationException("Not yet implemented");
    }
}
```

我们创建了一个OnroadService, AS自动帮我们生成了一个构造函数及实现了onBind接口，由于该接口未绑定相应的binder,所以暂时返回UnsupportedOperationException("Not yet implemented")异常。

接下来我们将在这个service里实现我们的aidl接口, 并在onBind方法里返回mBinder给调用者，调用者就可以透过mBinder调用我们的aidl接口了。

```
private IOnroad.Stub mBinder = new IOnroad.Stub() {
        @Override
        public String sayHello(String name, int age) throws RemoteException {
            String str = "Hello " + name + ", your age is " + age;
            Log.d(TAG, str);
            return str;
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        // TODO: Return the communication channel to the service.
        return mBinder;
    }
```

### d) 在AndroidManifest.xml注册该service

```
        <service
            android:name=".service.OnroadService"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="tech.onroad.aidlservicedemo.onroadservice"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
        </service>
```

至此，整个service端已创建完成。

## 2. 创建Client端工程

### a) 创建AIDL Client端工程AidlClientDemo

### b) 复制service端的aidl文件及aidl对应的java文件到客户端 

为了能在Client端掉用service端的接口，必须将service端的aidl文件及AS自动生成的aidl对应的java文件复制到该工程，且保持包名不变。如下图

![image](http://www.onroad.tech/images/2017110904.PNG)

### c) 创建ServiceConnection类

用于绑定Service, 若绑定成功，会回调onServiceConnected()方法，我们可以复写该方法，获取该service的ibinder接口; 若Service断开连接，则必须销毁binder实例，否则会造成内存泄漏。

```
ServiceConnection conn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            mIOnroad = IOnroad.Stub.asInterface(iBinder);
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            mIOnroad = null;
        }
    };
```

### d) 绑定服务

为了方便，我们让Activity启动的时候就去绑定服务

```
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent mServiceIntent = new Intent();
        mServiceIntent.setAction("tech.onroad.aidlservicedemo.onroadservice");
        mServiceIntent.setPackage("tech.onroad.aidlserverdemo");
        bindService(mServiceIntent, conn, Context.BIND_AUTO_CREATE);
    }
```

setAction里的参数tech.onroad.aidlserverdemo就是我们Service在AndroidManifest.xml注册的action, setPackage需指定service所在的包名，因为在Android 5.0以后，android已不再支持隐式Intent Service,所以必须指定service所在的包名。这样Client端绑定Service就已经完成。

### e） 调用aidl接口

接下来我们尝试调用一下service端的aidl接口,看能否调用成功。

```
public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            mIOnroad = IOnroad.Stub.asInterface(iBinder);
            try {
                String svcRet = mIOnroad.sayHello("dflw", 20);
                Log.d("Client", "Service return " + svcRet);
            } catch (RemoteException e) {
                e.printStackTrace();
            }

        }
```

为了图方便，我直接绑定成功后的回调接口onServiceConnected()调用service的AIDL接口。

## 3. 测试

将AidlServerDemo及AidlClientDemo的apk装载到Android设备或者模拟器上，

> 11-09 05:52:24.622 5439-5452/tech.onroad.aidlserverdemo D/OnroadService: Hello Liting, your age is 20
>
> 11-09 05:52:24.622 6662-6662/tech.onroad.aidlclientdemo D/Client: Service return Hello Liting, your age is 20

由上面的log可知，Client端已能正确调用Service端的接口，并成功将参数传递给Service端，且Service端也能将结果正确返回给Client.

完整代码可到我的github下载。
https://github.com/onroadtech/AidlDemo