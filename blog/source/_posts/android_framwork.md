---
title: Android封装思想与应用
date: 2017-08-14 13:32:54
tags:
---
<!--more-->
# 前言

不会偷懒的程序员不是好的程序员，学会如何封装是程序员的一条必经之路，那么你真的会封装吗？
接下来我们一步步来封装实现属于你自己的一个框架。

# Base类都能做些什么
Android中有个坑，如果应用被强杀那么Activity的栈信息将会被保存下来，下次恢复直接走到上次被强杀前的页面，这样就出现很多问题了。

**善用Base类解决应用强杀造成的空指针问题。
空指针场景：
如果一个应用在`application`里初始化了一个静态变量，并且在主页赋值，第三个页面会用到这个值如果应用这时候被强杀，再次进来是直接走到第三个页面的，这时候就会造成空指针异常。这里文字描述比较苍白，可以自己写个`Demo`跑一遍，强杀的话就用`AS`中调试的结束红色按钮，然后在手机上的最近打开的应用里重新打开。**



> 思路： 我们可以将主页`Activity`设置为`singleTask`模式，然后在每个页面启动前判断应用是否强杀，如果是的话就重新走到主页，不是的话去初始化页面。

我们在`BaseActivity`的`onCreate`方法中加入以下代码：

``` 
@Override
	protected void onCreate(@Nullable Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		if (MyApplication.AppStatus == -1) {
			protectApp();
		} else {
			setUpData();
		}

	}

```

结合我们的第一个页面`SplashActivity`

```
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        MyApplication.AppStatus = 0;
        super.onCreate(savedInstanceState);
       
    }
```
在`SplashActivity`的初始化中将` MyApplication.AppStatus`原先为`-1`的值改为`0` ,所以假设应用被强杀再次进入` MyApplication.AppStatus`值应该是`-1` ，所以我们判断这个值就可以知道应用是否是强杀掉的。

现在回头来分析`BaseActivity`中的代码，假如被强杀走`protectApp`方法，否则走`setUpData`方法。 
那么我们知道了应用是否被强杀之后就应该告诉页面该如何执行，所以`protectApp`中的代码应该是将APP的页面重新走到主页去。


```
protected void protectApp() {
		Intent intent = new Intent();
		intent.putExtra("protectApp", "protect");
		intent.setClass(this, MainActivity.class);
		startActivity(intent);

	}
```
到目前为止我们的每个页面只要基础自`BaseActivity`，就实现了在应用被强杀的时候执行`protectAPP`方法跳转到`MainActivity`中去

`MainActivity`由于是`SingleTask`模式所以会回调`OnNewIntent`方法，在这个方法中将页面`finish`并且跳转到欢迎页`SplashActivity`中去

那么如果当前页面就是`MainActivity`呢？ 如果再执行`protectAPP`方法就不对了，所以我们在`MainActivity`中去重写`protectAPP`方法，


```
  @Override
    protected void protectApp() {
        //当前页面被强杀调用
        startActivity(new Intent(MainActivity.this , SplashActivity.class));
        finish();
    }
```

到此为止，我们通过一个`BaseActivity`就解决了应用被强杀造成的空指针问题。

> 梳理一遍流程，当app被强杀，当前页面会被直接启动而跳过启动页，调用`BaseActivity`的`onCreate`方法，判断到是强杀于是走`protectApp`方法，跳转到主页， 由于主页是`SingleTask`模式，会回调`onNewIntent`方法，并且接收`Intent`传递的值从而知道应用被强杀了，于是主页跳转回启动页并且`finish` 。 这样应用在被强杀的时候总能再次打开启动页，空指针的问题就解决了。



## Base类还能做些什么

