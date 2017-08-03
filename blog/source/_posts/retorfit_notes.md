---
title: Retrofit学习分析
date: 2017-07-31 13:32:54
tags:
---

### Retrofit是如何通过注解来设置okhttp的配置的呢
Retrofit通过注解的方式来配置http请求，即使不用okhttp改用httpClient上层也不需要去修改。

带着多种疑问去看源码
<!--more-->


``` 
  public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(Platform.get().defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
```

这是`Retrofit`中`build`方法，其中有一行我们看到


```
okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }
```
这行代码目测是忘记删了，没有对`callFactory!=null`的情况做处理，并且从这里我们看出`Retrofit`目前只支持`Okhttp`。



```
    @Override public void enqueue(final Callback<T> callback) {
      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(final Call<T> call, final Response<T> response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancelation
                callback.onFailure(call, new IOException("Canceled"));
              } else {
                callback.onResponse(call, response);
              }
            }
          });
        }

        @Override public void onFailure(final Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(call, t);
            }
          });
        }
      });
    }
```

`callbackExecutor.execute`将异步从子线程切换到主线程中并回调`callBack`。

接着我们来看看`Retrofit`中个人认为最神奇的一行代码

`` RequestServes requestSerives = retrofit.create(RequestServes.class);
``
这个类明明是个接口，并且我们没有实现类，`Retrofit`是如何生成我们的实现类的呢？
这个。。。总之就是动态代理了  反正我也看不懂。

