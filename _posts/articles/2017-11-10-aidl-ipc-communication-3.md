---
layout: article
title: "跨进程间如何进行AIDL IPC 通信（三）"
modified:
categories: Android
excerpt: "如何使用AIDL传递一个枚举类型数据."
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-11-10T10:45:14-13:00
---

在前两篇博客[《跨进程间如何进行AIDL IPC 通信（一）》](http://www.onroad.tech/articles/aidl-ipc-communication-1/)[《跨进程间如何进行AIDL IPC 通信（二）》](http://www.onroad.tech/articles/aidl-ipc-communication-2/)中，我们介绍了基本类型参数及自定义复杂类型的AIDL IPC通信，理论上已经基本上涵盖了大部分需求。但如果自定义类型中含有枚举ENUM类型参数时，又改如何处理呢？

枚举本质上是一个类，相当于在自定义类中引入了一个新的类，接下来我们将谈谈带有ENUM类型参数的自定义类型如何在不同的进程间进行传递。
同样以上次两篇博文的代码工程为基础：

## 1. Service端
### a) 创建一个枚举ENUM
这次我们来创建一个业余爱好的枚举，包括游泳，音乐和足球；
```
package tech.onroad.aidlserverdemo.bean;

/**
 * Created by Liting Wang on 10/11/2017.
 */

public enum Hobby {
    SWIM,
    MUSIC,
    FOOTBALL;
}
```

### b) 修改自定义类Person
引入Hobby作为Person的成员变量，生成构造方法及geter and seter方法
```
......
private Hobby hobby;

......
protected Person(Parcel in) {
    name = in.readString();
    age = in.readInt();
    hobby = Hobby.values()[in.readInt()];
}

......
public Hobby getHobby() {
        return hobby;
}

public void setHobby(Hobby hobby) {
    this.hobby = hobby;
}
......
```
特别要注意的是获取枚举序列化的值不能写成hobby = in.readInt()，而应转化成enum的具体值：hobby = Hobby.values()[in.readInt()];

同理复写writeToParcel/readFromParcel接口
```
......
@Override
public void writeToParcel(Parcel parcel, int i) {
    parcel.writeString(name);
    parcel.writeInt(age);
    parcel.writeInt(hobby.ordinal());
}

public void readFromParcel(Parcel reply) {
    this.name = reply.readString();
    this.age = reply.readInt();
    this.hobby = Hobby.values()[reply.readInt()];
}
......
```
其中parcel.writeInt(hobby.ordinal())是将枚举的序号写入Parcel;
Hobby.values()[reply.readInt()]是读取对应序号的枚举值。
只要修改以上几个方法，其它的不用修改。


### b) 实现OnroadService的introducePerson接口
我们将客户端传过来的hobby属性打印出来，并回传ServicePerson的hobby属性给客户端。
```
@Override
public Person introducePerson(Person person) throws RemoteException {
    Log.d(TAG, "My name is " + person.getName() +
            ", I am " + person.getAge() +
            ", My hobby is " + person.getHobby());
    Person svcPerson = new Person();
    svcPerson.setName("ServicePerson");
    svcPerson.setAge(0);
    svcPerson.setHobby(Hobby.FOOTBALL);
    return svcPerson;
}
```
至此Service端代码已修改完成。

## 2. Client端
### a) 将Service端的Person类文件及枚举文件复制到Client端，包名需相同。
由于我们只修改了Person对象，并没有修改aidl文件，所以不需要重新编译，只需要将Service端的Person类文件及枚举文件复制到Client端，包名相同即可。

### b) 我们同样在onServiceConnected()方法里调用我们的自定义类型参数AIDL接口
```
Person person = new Person();
person.setName("Liting");
person.setAge(20);
person.setHobby(Hobby.SWIM);
try {
    Person svcPerson = mIOnroad.introducePerson(person);
    Log.d("Client", "Service return: Hello " + svcPerson.getName() +
            ", you hobby is " + svcPerson.getHobby());
} catch (RemoteException e) {
    e.printStackTrace();
}
```
## 3. 测试
将AidlServerDemo及AidlClientDemo的apk装载到Android设备或者模拟器上，运行：得到Service及client端的log如下
>11-10 05:58:29.259 2637-2651/tech.onroad.aidlserverdemo D/OnroadService: My name is Liting, I am 20, My hobby is SWIM
>11-10 05:58:29.261 4028-4028/tech.onroad.aidlclientdemo D/Client: Service return: Hello ServicePerson, you hobby is FOOTBALL

说明枚举类型也可以正常传递了。





> 完整代码可到我的github下载:
>
> <https://github.com/onroadtech/AidlDemo>