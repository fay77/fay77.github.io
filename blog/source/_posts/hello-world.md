---
title: Android中的IPC机制
---

<!--more-->

## 1.IPC简介  

IPC含义为进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程 , Android中IPC通信的方式有很多，例如:
- Binder
- Scoket

这里面最有特色的就是**Binder** 

## 2.Android中的多进程模式


#### 2.1开启多进程模式
　　Android中开启多线程模式非常的简单，只要在四大组件(Activity、Service、Receive、ContentProvider)的`AndroidMeniFest`中指定`android:process`属性。
	
    android:process="com.ryg.chapter_2.remote"
	android:process=":remote"

　　上面两种写法是有区别的：
1. `：` 的含义是指在当前线程名前面附加当前的包名，这是一种简写的方法。并且以`:`开头的进程属于私有进程，其他应用的组件不可以和它跑在同一个进程中。

2. `andoid:process="com.ryg.chapter_2.remote"`这是完整的写法，该写法的进程属于全局进程，其他应用可以通过共享`ShareUid`方式和它跑在同一个进程中。



----------

#### 2.2 多进程模式的运行机制
一句话形容多进程：
>当开启了多进程以后，各种奇怪的现象都出现了

　　分析：我们知道Android会为每个进程分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址控件，这就导致在不同虚拟机中访问同一个类的对象产生多份副本。



多进程会造成以下的问题：
* 静态成员和单例模式完全失效
* 线程同步机制完全失效
* SharedPreferences的可靠性下降
* Application会多次创建

----------

以上是多进程所带来的问题，但是我们不能因为多进程有很多问题就不去正视他，为了解决这个问题，系统提供了很多跨进程通信方法：

- 通过**Intent**来传递数据
- 共享文件和**sharedPerences**
- 基于**Binder**的**Message**和**AIDL**
- **Socket**等等

为了更好的理解IPC机制，我们先来熟悉一些基础概念，比如：
`Serializable`
`Paracelabel`

#### 2.3.1 Serializable接口
`Serializable`实现起来非常简单，几乎所有的工作都是系统完成的。如下代码：
```

    public class User implements Serializable {
    private static final long serialVersionUID = 321112538042654820L;
    public int userId;
    public String userName;
    public boolean isMale;

}
```
使用AS的同学会发现`serialVersionUID`无法自动提示，这是因为Android Studio设置中忽略了这个，需要重新设置下

1. File–>Settings–>Editor–>Inspections–>Java–>Serialization issues–>Serializable class without ‘serialVersionUID’ 勾选中该选项即可。

![](http://i.imgur.com/9Ek0nK3.png)
2. 进入实现了Serializable中的类，选中类名，Alt+Enter弹出提示，然后直接导入完成

**如何进行对象的序列号和反序列化呢**

```//序列号过程
        User user = new User(0, "jack", true);
        try {
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("cache.txt"));
            objectOutputStream.writeObject(user);
            objectOutputStream.close();
        } catch (IOException e) {
            e.printStackTrace();
        }




     //反序列化过程
        try {
            ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("cache.txt"));
            try {
                User user1 = (User) inputStream.readObject();
                inputStream.close();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
```

#### 2.3.2 Parcelable接口
>Parcelable也是一个接口，同样是实现这个接口就可以实现序列化和反序列化。下面是用法：

```
    public class User implements Parcelable {
    public int userId;
    public String userName;
    public boolean isMale;
    public Book mBook;

    public User(int userId, String userName, boolean isMale, Book book) {
        this.userId = userId;
        this.userName = userName;
        this.isMale = isMale;
        mBook = book;
    }


    protected User(Parcel in) {
        userId = in.readInt();
        userName = in.readString();
        isMale = in.readByte() != 0;
        mBook = in.readParcelable(Book.class.getClassLoader());
    }

    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(userId);
        dest.writeString(userName);
        dest.writeByte((byte) (isMale ? 1 : 0));
        dest.writeParcelable(mBook, flags);
    }
}```
所有的工作AS都会有提示只需无脑CTRL ENTER。

----------

#### 那么两种序列化之间改如何选取呢？
- Serializable是Java中的序列号接口，使用起来简单但是**开销大**
- Parcelable是Android中的序列化方式，因此更适合在Android平台，缺点是使用起来稍微麻烦，但是效率很高。**因此首选`Parcelable`** 

但是在以下两种情况下应选`Serializable`

- 将对象序列化到存储设备中
- 将对象序列化后通过网络传输

## 2.3.3 Binder

