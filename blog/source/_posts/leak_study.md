---
title: Android性能优化
date: 2018-05-06 13:32:54
tags:
---
<!--more-->


----------


### 项目中的单例
>在分析性能优化之前偶然的看到项目中的有很多单例模式，单例模式几乎是项目中被应用最多的设计模式，不同单例模式对性能开销也是不一样的。

 1. 饿汉式
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

 2. 懒汉式
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



----------


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


    
    
      
       
       
       
       
       


----------


#### 项目中的单例

```
   // 获取单例
    public static Common getInstance() {
        synchronized (Common.class) {
            if (instance == null) {
                instance = new Common();
                activityList = new ArrayList<>();
            }
        }

        return instance;
    }
```


```
 public static XunfeiSpeekUtils getInstance() {
        synchronized (XunfeiSpeekUtils.class) {
            if (instance == null) {
                instance = new XunfeiSpeekUtils();
            }
        }

        return instance;
    }
```
每次调用`getInstance()`都会进行同步操作，这样是非常不友好的，造成很大的开销。 即使加双重判断锁也会出现DLC失效的问题。


## 内存泄露
>首先要搞清楚**内存泄露**跟**内存溢出**是两个概念，比如一车最多能坐5个人，你却非要塞下10个，车就挤爆了，这就是内存溢出。 而车上的五个人本来应该在车到站后都下车的，结果只下车了3个人，还有两个人一直赖在座位上不肯下来，这就是**内存泄露** ，泄露的多了就会导致OOM产生。

### 非静态内部类
在java中，内部类会隐式的持有外部类的引用，项目中用的比较多的是`Handler`作为非静态内部类的使用，如果`Handler`在`fragment`或者`Activity`结束的时候扔有未执行完的任务，比如一个定时任务等，那么就会导致内存泄露，项目中非静态内部类的`Handler`比比皆是：

```
 private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            Bitmap thumbnailBitmap = (Bitmap) msg.obj;
            mJCVideoPlayerStandard.thumbImageView.setImageBitmap(thumbnailBitmap);
        }
    };
```


----------


```
    Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            int what = msg.what;
            switch (what) {
                case READ_ADDRESS_IN_JSON:
                    String json = (String) msg.obj;
                    //得到地址的实体类对象
                    try {
                        JSONObject object = new JSONObject(json);
                        JSONArray array = object.getJSONArray("data");
                        ...
```


----------


```
  private Handler handler = new Handler() {
        public void handleMessage(android.os.Message msg) {
            switch (msg.what) {
                //0101绑定指令
                case 11:
                    setWriteCharacteristicNotification("0101", writeBtCharacteristic);
                    cancelScan();
                    Common.getInstance().setBluetoothState(true, bluetoothGatt.getDevice().getName());
                    runOnMainThread(new Runnable() {
                        @Override
                        public void run() {
                            EventBus.getDefault().post(new ConnectSuccessEvent("ConnectSuccess", bluetoothGatt.getDevice().getName()));
                        }
                    });

                    handler.removeMessages(11);//移除消息
                    break;
```


----------


```
   Handler myHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case 1000:
                    NoHttpUtils.httpGet(AppConstant.URL_HEARTBEAT, new HashMap(), mOnResponseListener, REQUEST_HEARTBEAT_CODE);
                    break;
                default:
                    break;
            }
        }
    };
    ...
```
上面我们分析非静态内部类的`Handler`可能会导致内存泄露，检测工具就给了我们下面这张图，追踪下去发现是`StateLayout`产生的，并且会发生在每个`Fragment`中，清楚源头之后我们点击进入`StateLayout`类中


<img src="https://i.imgur.com/Is7ig6E.jpg" width="300" height="500"/> 


```
public class StateLayout extends FrameLayout {
    private Handler handler = new Handler();
    ...
```


```
...
  final Runnable runnable = new Runnable() {
                @Override
                public void run() {
                    switch (switchPosition) {
                        case 0:
                            progressTextView.setText(mContext.getString(R.string.state_loading1));
                            break;
                        case 1:
                            progressTextView.setText(mContext.getString(R.string.state_loading2));
                            break;
                        case 2:
                            progressTextView.setText(mContext.getString(R.string.state_loading3));
                            break;
                        case 3:
                            progressTextView.setText(mContext.getString(R.string.state_loading4));
                            switchPosition = -1;
                            break;
                        default:
                            break;
                    }
                    switchPosition++;
                    handler.postDelayed(this, 500);
                }
            };
            handler.post(runnable);
            ...

```

>果然可以在里面看到一个非静态内部类`Handler`， 我们可以看到当前类是继承自`FrameLayout`的，在`Android`控件中绝大部分都是持有布局对应的`fragment`或者`Activity`的引用的，而当前`Handler`是非静态内部类隐性的持有了外部类的引用，间接的就导致在布局中使用了这个`StateLayout`的`fragment`被`Handler`所持有，即使执行了`onDestory`，其引用一直无法被回收掉。

>解决方案： 这里我将`Handler`删除，用一个属性动画代替，就不会再出现`Handler`的内存泄露问题了，不过这里却报了属性动画的内存泄露，查阅相关资料发现，属性动画如果不及时释放也是会导致内存泄露的，最终在显示隐藏回调方法里判断当前`StateLayout`不再显示的时候将动画释放。

```
@Override
    protected void onVisibilityChanged(@NonNull View changedView, int visibility) {
        super.onVisibilityChanged(changedView, visibility);
        if (valueAnimator == null) {
            return;
        }
        if (visibility == GONE || visibility == INVISIBLE) {
            valueAnimator.cancel();
            valueAnimator = null;
        }
    }

```


----------


### Context 导致内存泄漏
```
public class XunfeiSpeekUtils {
    private static XunfeiSpeekUtils instance;

    public static XunfeiSpeekUtils getInstance() {
        synchronized (XunfeiSpeekUtils.class) {
            if (instance == null) {
                instance = new XunfeiSpeekUtils();
            }
        }

        return instance;
    }

    SpeechRecognizer mIat;
    RecognizerDialog mIatDialog;
    HashMap<String, String> mIatResults = new LinkedHashMap<String, String>();
    Context mContext = null;
   ...
```


----------
```
 XunfeiSpeekUtils.getInstance().init(mActivity).speak(voice);
```

**上面这行代码是必然会导致内存泄露的，而且露的还不少，我们可以看下面两张图：**

<img src="https://i.imgur.com/HpPAfeE.png" width="300" height="500"/> 

**红框部分我们点击进去可以看到正是这个`XunfeiSpeekUtils`导致了2.7MB的内存泄露：**




<img src="https://i.imgur.com/52DzjNX.jpg" width="300" height="500"/> 


>案例分析：`XunfeiSpeekUtils`是个单例类，生命周期是很长的，持有的`context`是`mActivity`，这就导致当前activity在销毁的时候其引用扔被这个单例所持有，这就造成的内存泄露，解决方法是传入`getApplicationContext`，因为Application的生命周期是贯穿整个程序的，所以XunfeiSpeekUtils类持有它的引用，也不会造成内存泄露问题。


----------


### 集成LeakCanary内存泄露检测工具到项目
>在只监控`Activity`的时候，打开咱们项目app的时候LeadCanary立刻就发送了通知过来，点击通知栏我们可以看到具体的泄露地方

<img src="https://i.imgur.com/fEfHcoA.jpg" width="300" height="500"/> 


>如何集成这里就不说了，百度都有，而且非常简单。主要来看下怎么理解这个工具的错误信息


<img src="https://i.imgur.com/ZuIXswG.jpg" width="300" height="500"/> <img src="https://i.imgur.com/0OiPGUJ.jpg" width="300" height="500"/> 

 1.以`LoginActivity`为例，由上至下，`ProgressDialogUtils`中的**静态**`imageView`引用的`mContext`导致了`LoginActivity`内存泄露。接着我们打开这个导致内存泄露的`ProgressDialogUtils`一探究竟。

```
public class ProgressDialogUtils extends AlertDialog {


    private static ImageView imageView;
    private static TextView  textView;

   ...

```
可以看到这里所有的**View控件**都是静态的，而我们的`ImageView`是持有`Activity`引用的，`static`变量在内存中是单独存在于内存块中的，这种情况下，`Activity`是没法被彻底销毁的，因为在内存中一直有一个引用，导致`Activity`也无法被回收，自然就会内存泄漏了。 建议，在Android中不要使用`static`修饰控件。


2.接着来看另一个导致内存泄露的`MainActivity` 
```
public class Player implements OnCompletionListener {
	private MediaPlayer mediaPlayer;
	private PlayerHandler mPlayerhandler;
	Context mContext = null;

    public Player(Context context) {
        mContext = context;
        mPlayerhandler = new PlayerHandler();

```
这个就比较简单了，`Player`控件中真正执行操作的其实还是`MediaPlayer`, `MediaPlayer`的源码中是持有`Activity`的引用的，因此在使用完之后需要及时的释放，咱们代码中并未释放过，这里其实还是有个坑的，一般来说释放代码应该像这样的
```
 /**
     * 释放播放器资源
     */
    private void ReleasePlayer() {
        if (player != null) {
            player.stop();
            player.release();
            player = null;
        }
```
但是这样写之后，发现还是存在内存泄露，大致原因就是在源码中release方法并未到某个引用做释放动作，而reset方法可以。改成这样：

```
 /**
     * 释放播放器资源
     */
    private void ReleasePlayer() {
        if (player != null) {
            player.reset();
            player.release();
            player = null;
        }
```

### 使用Lint分析
>Android Studio内置了Lint，只要点一下就可以使用，Lint 的使用路径：工具栏 -> Analyze -> Inspect Code…

<img src="https://i.imgur.com/hMUg3U6.png" />

<img src="https://i.imgur.com/LcKS0ou.png" />

<img src="https://i.imgur.com/rGSNjEg.png" />

<img src="https://i.imgur.com/YOvrSt3.png" />


>自定义View的时候 onDraw onMeasure onLayout等方法会被多次调用 ，如果在这几个方法里去实例化对象，对性能消耗是很大的，比如下面几个：

<img src="https://i.imgur.com/1rJgEz4.png" />


>项目中经常使用Weight属性来做适配，但是如果父布局跟子布局同时都使用Weight是对性能很不好的，解决方法也很简单，去除子布局中非必须的weight。


<img src="https://i.imgur.com/lHcGhSa.png" />


----------
###### 静态变量造成的内存泄露
<img src="https://i.imgur.com/nSEcGPv.png" />


>结语：Android的性能优化是多方面的，比如启动速度优化、UI流畅度优化、apk瘦身、电量优化、内存优化等，这里只是对内存方面做一个简短的介绍。



