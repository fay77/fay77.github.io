# Android 性能优化

<!--more-->

---

## 理解Java中的堆栈概念
> 我理解的堆栈比较简单，就是几乎所有的对象实例都在堆中分配内存，所以java中的堆内存是被线程所共享的，而栈内存恰恰相反，它是线程私有的，生命周期与线程周期捆绑在一起，每一个方法从调用到完成，就对应虚拟机中的一个入栈到出栈的过程。 


----------


###### 项目中的单例
> 单例模式几乎是项目中被应用最多的设计模式，不同单例模式对性能开销也是不一样的。

 - 饿汉式
```
public class HungSingle {
    private static final HungSingle hungSingle = new HungSingle();

    //构造函数私有
    private HungSingle() {

    }

    //公有的静态函数，对外暴露获取单例对象的接口
    public static HungSingle getHungSingle() {
        return hungSingle;
    }

}
```
`HungSingle`类不能通过new的形式构造对象，只能通过`HungSingle.getHungSingle()`方法来获取，而这个`HungSingle`对象是静态对象，并且在声明的时候就已经初始化，这就保证了`HungSingle`对象的唯一性。

 1. 懒汉式
```
public class Singleton {
    private static Singleton instance;

    private Singleton() {
        
    }

    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```
上述方法中添加`synchronized`字段保证在多线程情况下单例对象的唯一性，但是会有一个问题，每次调用`getInstance()`方法都会进行同步，这样会消耗不必要的资源，这也是懒汉式存在的最大问题，**这种模式一般不建议使用**。

 2. Double Check Lock (DCL)单例
 ```
public class Singleton {
        private static Singleton sInstance = null;

        private Singleton() {
        
     }

        public static Singleton getInstance() {
         if (sInstance == null) {
             synchronized (Singleton.class) {
                  if (sInstance == null) {
                      sInstance = new Singleton();
                    }
             }
          }
         return sInstance;
        }
    }
 
 ```
 可以看到`getInstance()`方法中对`sInstance`进行了两次判空，第一层判断主要为了避免不必要的同步，第二层的判断则是为了在`null`的情况下创建实例。
 
 >假设有个线程A执行到了` sInstance = new Singleton()`语句，它大致做了如下3件事：
 
 
    1. 给`Singleton`的实例分配内存；
    2. 调用`Singleton`的私有构造函数，初始化成员字段；
    3. 将`sInstance`对象指向分配的内存空间， 此时`sInstance`对象就不为空了。           
    >在**JDK 1.5**之前上面的第二和第三顺序是无法保证的，如果3执行完、2未执行，这时候被切换到B线程，由于`sInstance`在线程A内执行过第3步了，`sInstance`已经是非空了，所以线程B直接取走了`sInstance`,再使用就会出错，这就是**DLC失效问题**。 在JDK1.5之后官方加入了`volaatile`关键字，因此在1.5之后的版本只需要将`sInstance`的定义改为`private volatile static Singleton sInstance = null `即可。
    
    
    

 3. 静态内部类单例模式 （最推荐的模式）
 ```
 public class Singleton{
        private Singleton() {

        }

     public static Singleton getInstance() {
         return SingletonHolder.sInstance;
     }

     /**
      * 静态内部类
      */
     private static class SingletonHolder {
          private static final Singleton sInstance = new Singleton();
      }
    }

 
 ```
 只有在第一次调用`Singleton`的`getInstance`方法时才会导致`sInstance`被初始化。因此，第一次调用`getInstance`方法会导致虚拟机加载`SingletonHolder`类，这种方式不仅能够保证线程安全，也能够保证单例对象的唯一性，同时也延迟了单例的实例化。
 
 
 





