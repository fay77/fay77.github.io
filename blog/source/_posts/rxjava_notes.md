---
title: RxJava学习笔记
date: 2017-07-31 13:32:54
tags:
---
# 前言
之前学了一段时间的RxJava，由于项目中没用到过，回头看居然又忘了，索性今天再去捋一遍吧。
注：只记录学习笔记，详细的不赘述。

先贴上个人觉得RxJava讲的最好的一个系列： [http://www.jianshu.com/p/464fa025229e](http://www.jianshu.com/p/464fa025229e)
<!--more-->
### ObservableEmitter
 ObservableEmitter 发射器，可以发出`onNext(T value)`、`onComplete()`和`onError(Throwable error)`必须遵循的规则：
> 
上游即`Observable`在发送`onNext`、 `onComplete` 、`onError`之后均可以持续发送接下来的事件，也可以不发送`onComplete`、但是`onComplete`和`onError`不可以同时存在并且唯一。
下游接收到`onComplete`、`onError`之后不再继续接收。


### Disposable
可以简单理解成两根水管之间的一个开关。当调用他的`dispose()`方法时，它就会将两根水管切断，导致下游接收不到事件 ，但是上游会**继续**发送事件。


> 上游在切断水管之后依旧发送了3、4、onComplete事件，而下游只接收到了切断之前的事件


### subscribe()
有多个重载方法

1. subscribe() 不带任何参数，表示下游不关心上游发送了什么，发你的去吧
2. 带有一个`Consumer`表示下游只关心`onNext`事件。

## 线程操控
上下游默认是工作在同一线程中的。

```
		observable.subscribeOn(Schedulers.newThread()).observeOn(AndroidSchedulers.mainThread()).subscribe(consumer);

```
> `subscribeOn()` 指定的是上游发送事件的线程, `observeOn()` 指定的是下游接收事件的线程.

- 多次调用`subscribeOn`只有第一次生效
- 多次调用`observerOn`，每次调用下游都会切换一次线程

尽量使用RxJava内置的线程来使用，内部是线程池来维护效率比较高：
- `Schedulers.io` 代表io操作的线程，通常用于网络、读写文件等io密集型操作。
- `Schedulers.computation()`代表CPU计算密集型操作，例如需要大量计算的操作
- `Schedulers.newThread()`代表一个常规的新线程
- `AndroidSchedulers.mainThread`代表Android的主线程


## 变换操作符Map
将上游的事件通过指定的函数去变化，下游接收到的就是变化后的一个个事件了。

``` 		
Observable.create(new ObservableOnSubscribe<Integer>() {
			@Override
			public void subscribe(ObservableEmitter<Integer> e) throws Exception {
				e.onNext(1);
				e.onNext(2);
				e.onNext(3);
			}
		}).map(new Function<Integer, String>() {
			@Override
			public String apply(Integer integer) throws Exception {
				return "this is result" + integer;
			}
		}).subscribe(new Consumer<String>() {
			@Override
			public void accept(String s) throws Exception {
				Log.d(TAG, s);

			}
		});
```

## FlatMap
上游每次发送一个事件，`FlatMap`就会创建一个新水管，发送转换之后的事件，不保证顺序

```
flatMap(new Function<Integer, ObservableSource<String>>() {
			@Override
			public ObservableSource<String> apply(Integer integer) throws Exception {
				final List<String> list = new ArrayList<String>();
				for (int j = 0; j < 3; j++) {
					list.add("i am value" + j);
				}
				return Observable.fromIterable(list).delay(10, TimeUnit.MILLISECONDS);
			}
		})
```

> 延迟只是为了证明`flatMap`是无序的 ，如果需要有序的可以使用`concatMap`，用法与`flatMap`一模一样。



## zip操作符
通过一个函数将多个`Observable`结合到一起然后再发送


```

		Observable.zip(observable, observable1, new BiFunction<Integer, String, String>() {
			@Override
			public String apply(Integer integer, String s) throws Exception {
				return integer + s;
			}
		}).subscribe(new Observer<String>() {
			@Override
			public void onSubscribe(Disposable d) {
				Log.d(TAG, "onSubscribe");

			}

			@Override
			public void onNext(String value) {
				Log.d(TAG, "onNext: " + value);

			}

			@Override
			public void onError(Throwable e) {
				Log.d(TAG, "onError");

			}

			@Override
			public void onComplete() {
				Log.d(TAG, "onComplete");

			}
		});

	}
```

由`zip`引出的`Backpressure`概念：

`zip`是用于组合两根`水管`，假如一根水管发送快，一根发送慢，那么发送快的势必需要等待，因此需要将数据缓存到一个类似`水缸`的地方。但是如果一直往水缸里发送数据，内存就会爆掉。
`Backpressure`就是控制流量的

###### 控制流量方案1：

```
Observable.create(new ObservableOnSubscribe<Integer>() {                         
    @Override                                                                    
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception { 
        for (int i = 0; ; i++) {   //无限循环发事件                                              
            emitter.onNext(i);                                                   
        }                                                                        
    }                                                                            
}).subscribe(new Consumer<Integer>() {                                           
    @Override                                                                    
    public void accept(Integer integer) throws Exception {                       
        Thread.sleep(2000);                                                      
        Log.d(TAG, "" + integer);                                                
    }                                                                            
});

```