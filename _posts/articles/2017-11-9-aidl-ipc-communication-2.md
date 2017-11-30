---
layout: article
title: "跨进程间如何进行AIDL IPC 通信（二）"
modified:
categories: Android
excerpt: "如何使用AIDL传递一个复杂自定义类型数据."
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2017-11-9T18:00:00
---



在上一篇博客[《跨进程间如何进行AIDL IPC 通信（一）》](http://www.onroad.tech/articles/aidl-ipc-communication-1/)中，我们只介绍了基本类型参数的AIDL IPC通信，如果想传递一个复杂的类型，那该如何处理呢，幸好Android支持AIDL传递自定义类型，今天我们将着重介绍自定义类型如何在不同的进程间进行传递。
## 1. Service端
### a) 创建一个自定义类
创建一个自定义对象，咱们还是以Person为例，该对象必须完成以下动作：
- 实现Parcelable接口，并且实现Parcelable接口的public void writeToParcel(Parcel dest, int flags)方法，因为任何复杂的自定义对象都是由基本类型封装而成，因此传输自定义类型时，只要将其拆分成基本类型，而Parcelable接口就是用来约定数据被序列化传输的方式。public int describeContents()：内容接口的描述，通常直接返回0即可
  public void writeToParcel(Parcel parcel, int i)：用于将当前类的成员写入到Parcel容器中，重写该方法时调用Parcel类的write系列方法即可。如writeInt(),writeString()等。
- 自定义类型中必须含有一个名称为CREATOR的静态成员，该成员对象要求实现Parcelable.Creator接口及其方法。
- 通常还需要我们自定义readFromParcel(Parcel parcel)方法，用于从Parcel容器中读出数据，读出的顺序必须与写入时的顺序一致。

其实在Android Studio实现上述两点时非常简单，只要鼠标放点击Parcelable接口上，按下alt + enter快捷键，选择Implement methods即可自动完成实现第一点的编码；再次将鼠标放于Person 类上，按下alt + enter快捷键，选择Add Parcelable Implementation, 即可自动完成第二点的编码。

```
public class Person implements Parcelable {
    private String name;
    private int age;

    public Person(){

    }

    protected Person(Parcel in) {
        name = in.readString();
        age = in.readInt();
    }

    public static final Creator<Person> CREATOR = new Creator<Person>() {
        @Override
        public Person createFromParcel(Parcel in) {
            return new Person(in);
        }

        @Override
        public Person[] newArray(int size) {
            return new Person[size];
        }
    };

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeString(name);
        parcel.writeInt(age);
    }

    public void readFromParcel(Parcel reply) {
        this.name = reply.readString();
        this.age = reply.readInt();
    }
}
```
### b) 创建一个aidl文件声明你的自定义类型
在自定义类型所在包下创建一个aidl文件对自定义类型进行声明，文件的名称与自定义类型同名。 声明方式
```
parcelable Person;
```
如图所示：

![image](http://www.onroad.tech/images/2017110905.PNG)

### c) 在接口aidl文件中使用自定义类型
需要使用import显式导入Person，in,out,inout为数据流向
```
import tech.onroad.aidlserverdemo.bean;

interface IOnroad {
    
    String sayHello(String name, int age);

    void introducePerson(inout Person person);
}
```
### d) 实现aidl文件生成的接口
本例是OnroadService，但并非直接实现接口，而是通过继承接口的Stub来实现（Stub抽象类内部实现了aidl接口），并且实现接口方法的代码。内容如下：
```
private IOnroad.Stub mBinder = new IOnroad.Stub() {
    @Override
    public String sayHello(String name, int age) throws RemoteException {
        String str = "Hello " + name + ", your age is " + age;
        Log.d(TAG, str);
        return str;
    }
    
    @Override
    public void introducePerson(Person person) throws RemoteException {
        Log.d(TAG, "My name is " + person.getName() + ", I am " + person.getAge());
    }
};
```
至此，Service端的工作已完成。

## 2. Client端
### a) 同理将service端的aidl文件及aidl对应的java文件复制到client端
目录结构如下：

![image](http://www.onroad.tech/images/2017110906.PNG)

### b) 同理，为图省事，我们同样在onServiceConnected()方法里调用我们的自定义类型参数AIDL接口
```
Person person = new Person();
person.setName("Liting");
person.setAge(20);
try {
    mIOnroad.introducePerson(person);
} catch (RemoteException e) {
    e.printStackTrace();
}
```
## 3. 测试
将AidlServerDemo及AidlClientDemo的apk装载到Android设备或者模拟器上，运行：得到Service端的log如下
>23169-23210/tech.onroad.aidlserverdemo D/OnroadService: My name is Liting, I am 20

说明可以正常传递我们的自定义类型数据了。

上面的例子说明我们可以将client端的自定义类型参数传递到service端，那service端可否将其作为返回类型回传给client端呢，我们可修改接口尝试一下：
### a) 修改IOnroad.aidl
```
Person introducePerson(inout Person person);
```
### b) re-build service 工程
### c) 在OnroadService里复写public Person introducePerson(Person person)introducePerson()方法
```
@Override
public Person introducePerson(Person person) throws RemoteException {
    Log.d(TAG, "My name is " + person.getName() + ", I am " + person.getAge());
    Person svcPerson = new Person();
    svcPerson.setName("ServicePerson");
    svcPerson.setAge(0);
    return svcPerson;
}
```
### d) 将修改后的IOnroad.aidl及IOnroad.java更新到client工程
### e) 在onServiceConnected()里重新调用public Person introducePerson(Person person)接口
```
@Override
public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
    mIOnroad = IOnroad.Stub.asInterface(iBinder);
    try {
        String svcRet = mIOnroad.sayHello("Liting", 20);
        Log.d("Client", "Service return " + svcRet);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
    Person person = new Person();
    person.setName("Liting");
    person.setAge(20);
    try {
        Person svcPerson = mIOnroad.introducePerson(person);
        Log.d("Client", "Service return: Hello " + svcPerson.getName());
    } catch (RemoteException e) {
        e.printStackTrace();
    }
}
```
### d) 测试得到client端log如下：
>11-09 12:36:00.566 8538-8538/tech.onroad.aidlclientdemo D/Client: Service return Hello Liting, your age is 20
>11-09 12:36:00.569 8538-8538/tech.onroad.aidlclientdemo D/Client: Service return: Hello ServicePerson

说明Service端同样可以传递复杂的自定义类型。





> 完整代码可到我的github下载:
>
> <https://github.com/onroadtech/AidlDemo>