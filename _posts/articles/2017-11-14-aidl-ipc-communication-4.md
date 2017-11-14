---
layout: article
title: "跨进程间如何进行AIDL IPC 通信（四）"
modified:
categories: articles
excerpt: "如何使用AIDL打成jar包供第三方app调用."
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-11-14T12:11:00
---

在前三篇博客中，我们介绍了如何进行AIDL IPC通信，代码设计及AIDL数据传输的类型。在第一篇博客代码设计中，我们曾说过，需要将AIDL接口文件及对应生成的Java文件要原封不动拷贝到客户端，这样客户端才能正常编译执行，但是这样操作非常麻烦且不合理，所以今天我们将要介绍如何将AIDL接口文件及对应生成的Java文件生成Jar包库，直接丢给第三方客户端使用。严格意义上说，这篇博客不属于AIDL IPC通信的范畴，但为了使整个AIDL IPC通信设计的连续性，我还是决定将其放到这里。同样以前面的工程为例：
## 1. 创建Module
创建Module其实就是为了创建android lib库。由于我们的AIDL接口是在service端设计的，且由service端对外提供，因此我们选择在service端工程创建这个module.
点击File菜单，New --> New Module --> Android Library，输入要创建的Module Name等信息。创建成功后会在我们的工程中多了个module目录结构，如下如。

![image](http://www.onroad.tech/images/20171114/01.PNG)

## 2. 新建AIDL接口
右击Module Java包文件夹，新建IAidlJarTest.aidl接口，随便写个接口函数
```
package tech.onroad.aidljar;
interface IAidlJarTest {
    void add(int a, int b);
}
```
重新编译，同样会在如下目录生成对应的Java文件

![image](http://www.onroad.tech/images/20171114/02.PNG)

## 3. 添加引用module编译
在app module的build.gradle文件dependencies中添加一行compile project(':aidljar')，使得app module依赖我们的AidlJar module。
```
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:26.1.0'
    implementation 'com.android.support.constraint:constraint-layout:1.0.2'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.1'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.1'
    compile project(':aidljar')
}
```
这样就可以在app module中调用刚才的IAidlJarTest.aidl的接口了。
## 4. 新建一个Service来实现aidl接口
具体如何创建Service及实现接口方法请参考《[跨进程间如何进行AIDL IPC 通信（一）](http://www.onroad.tech/articles/aidl-ipc-communication-1/)》第一节。
创建AidlJarService并实现add()接口，代码如下：
```
public class AidlJarService extends Service {
    public final String TAG = "AidlJarService";
    public AidlJarService() {
    }

    private IAidlJarTest.Stub mBinder = new IAidlJarTest.Stub() {

        @Override
        public void add(int a, int b) throws RemoteException {
            Log.d(TAG, "Add result: " + (a + b));
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        // TODO: Return the communication channel to the service.
        return mBinder;
    }
}
```
还有别忘了在AndroidManifest.xml注册该service.
```
<service
    android:name=".service.AidlJarService"
    android:enabled="true"
    android:exported="true">
    <intent-filter>
        <action android:name="tech.onroad.aidlservicedemo.aidljarservice" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</service>
```
## 5. 编写生成Jar包脚本
划重点了^^
在Module aidljar的build.gradle加入如下脚本：
```
task makeAidlJar(type: Copy) {
    //delete exist jar library
    delete 'build/libs/aidltest.jar'
    //copy from file path
    from('build/intermediates/bundles/release/')
    //to dist file path
    into('build/libs/')
    include('classes.jar')
    rename ('classes.jar', 'aidltest.jar')
}

makeAidlJar.dependsOn(build)
```
然后在AS的Terminal执行命令：gradlew makeAidlJar
BUILD SUCCESSFUL 后就会生成aidltest.jar
如下图所示

![image](http://www.onroad.tech/images/20171114/03.PNG)

至此Service端和Jar library都已生成成功。接下来我们将尝试一下在client端引用这个jar包，看能不能正常调用。

## 6. 在client端引入该jar包库
将service端生成的aidltest.jar复制到AidlClientDemo工程中的app/libs/目录下，如图

![image](http://www.onroad.tech/images/20171114/04.PNG)

右击aidltest.jar,选择Add As Library...
接下来我们就可以正常调用该库的接口了。
## 7. 调用jar包库的接口
为了方便，我直接在client端工程MainActivity.java修改. 具体步骤可参考《[跨进程间如何进行AIDL IPC 通信（一）](http://www.onroad.tech/articles/aidl-ipc-communication-1/)》的第二节，编码完成后如下
```
private IAidlJarTest mIAidlJarTest;
ServiceConnection aidlServiceconn = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        mIAidlJarTest = IAidlJarTest.Stub.asInterface(iBinder);
        try {
            mIAidlJarTest.add(8, 8);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        mIAidlJarTest = null;
    }
};
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    Intent mServiceIntent = new Intent();
    //mServiceIntent.setAction("tech.onroad.aidlservicedemo.onroadservice");
    //mServiceIntent.setPackage("tech.onroad.aidlserverdemo");
    //bindService(mServiceIntent, conn, Context.BIND_AUTO_CREATE);

    //Test aidl jar service
    mServiceIntent.setAction("tech.onroad.aidlservicedemo.aidljarservice");
    mServiceIntent.setPackage("tech.onroad.aidlserverdemo");
    bindService(mServiceIntent, aidlServiceconn, Context.BIND_AUTO_CREATE);
}
```
## 8. 测试运行
将两个apk push到模拟器运行，Log 控制台输入如下log

![image](http://www.onroad.tech/images/20171114/05.PNG)

运行结果与我们预期一致，说明Client成功调用aidltest.jar的接口。



> 完整代码可到我的github下载:
>
> <https://github.com/onroadtech/AidlDemo>

-----完-----