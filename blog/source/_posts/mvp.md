---
title: MVP模式探讨
date: 2018-02-24 9：38
tags:
---
# 前言
<!--more-->
  决定重写捋一遍android的各个技术点，常用的模式框架等等，就从这第一篇开始吧！ 

# MVP模式到底是什么？ #

  个人理解MVP模式就是将数据层与视图层彻底分离，降低耦合性，简洁代码，缺点就是随之会增加很多类、接口文件。 不过。。。跟现在项目的0框架相比真的舒服很多了。

## 简单描述 ##
- MVP顾名思义就是分为 M层、 V层、 P层 ，  M层是数据层，负责开启网络请求或者从数据库等拉取数据， V层是视图层，用于渲染界面展示视图的，而P层就在这两个层之间充当桥梁，连接这两个层，至此，M层与V层相互独立，通过一个P层，视图层专心渲染视图，数据层专心拉取数据，分工明确。


----------
大致的流程是这样的：

1. 首先，在V层中， V层需要告诉P层，V：  P层，我需要XXX数据，你给我赶紧弄过来。 
2. P层听到V层于是就告诉M层，  P：  M层，V告诉我他要XXX数据了，你赶紧的吧，好了告诉我啊。
3. 于是，M层就开始开启网络请求拉取数据了，成功或者失败都会告诉P层。
4. 在P层中这时候已经拿到M层返回的结果了，这时候P会通过接口回调返回给V层。




> 我们简单的看下一个代码， 业务大致就是一个页面请求列表数据，用MVP模式改造如下：
> 
1. View层告诉P层，我想看最近热映的十部电影都有哪些

      //View层
      public class MovieActivity implements IMovieView{

      ....
      MoviePresenter mMoviePresenter;

       @Override  
      public void onCreate(Bundle savedInstanceState) {  
      mMoviePresenter = new MoviePresenter(this);//构造方法的参数为IView接口
      mMoviePresenter.getPlayingMovie(10);
       }

       ....
       }




P层告诉M层， V他要看最近的十部电影，你给我弄过来


       Presenter层
       public class MoviePresenter {
  
       private IMovieView mIView;

       public MoviePresenter(IMovieView iMovieView) {
       mIView = iMovieView
      }

      //获取正在上映的电影
      public void getPlayingMovie(int count) {

      //通知M层开启请求
      MovieModel.getInstance.getPlayingMovie(count, ...);
      }

      ....
      }
   

M层开始网络请求数据，返回数据通知P层

      //Model层
      public class MovieModel {
    
      ...
    
      //开启网络请求获取正在上映的电影
      public void getPlayingMovie(int count,  HttpObserver<List<MovieRes>>  observer,  
      PublishSubject<LifeCycleEvent> lifecycleSubject) {
    
      //开启网络请求，结果返回后通过传入的observer回调给P层
      Observable observable = RetrofitUtil.getApiService().getPlayingMovie(count);
        RetrofitUtil.composeToSubscribe(observable, observer, lifecycleSubject);
      }
        }
    




P层拿到M层返回的数据通过接口回调返回给V层 ，整个过程结束。

      //Presenter层
      public class MoviePresenter {
    
      ....
    
      //获取正在上映的电影
      public void getPlayingMovie(int count) {
    
      //通知M层开启请求
      MovieModel.getInstance.getPlayingMovie(count, new HttpObserver<List<MovieRes>>() {
       @Override
      public void onNext(String title, List<MovieRes> list) {
    
      //通过接口IView将成功的结果回调给V层
      if (mIView != null) {
      mIView.getMovieSuccess(list);
      }
      }
    
      @Override
      public void onError(int errType, String errMessage) {
    
      //通过接口将失败的结果回调给V层
      if (mIView != null) {
      mIView.getMovieFail(errType, errMessage);
      }
       }
    
       });
         }
    


----------




  
    
    
      //IView接口代码，用于Presenter层与View层的交互
      public interface IMovieView extends IBaseView {
    
      //成功获取电影数据
      void getMovieSuccess(List<MovieRes> list);
      //获取电影数据失败
      void getMovieFail(int status, String desc);
      
      }
    







----------

    
     	 //View层（实现回调接口IMovieView中的方法）
    
     	 public class MovieActivity implements IMovieView{
    
    	     ...
      
   	   //网络请求成功的回调
   	   @Override
  	    public void getMovieSuccess(List<MovieRes> list) {
  	    if (CollectionUtil.isEmpty(list)) {
   	   //数据为空则设置页面为“无数据”状态
   		   getLoadLayout().setLayoutState(State.NO_DATA);
 		     }else{
  	    //设置页面为“成功”状态，显示正文布局
  	    getLoadLayout().setLayoutState(State.SUCCESS);
    
  	    //列表展示数据
  	    mMovieAdapter = new MovieAdapter(mActivity, list, 	this);
  	    mRvMovie.setLayoutManager(new LinearLayoutManager(getContext()));
  	    mRvMovie.setHasFixedSize(false);
   	 	  mRvMovie.setAdapter(mMovieAdapter);
   	   }
    
       }
    
      //网络请求失败的回调
      @Override
      public void getMovieFail(int status, String desc) {
      //设置页面为“失败”状态
      getLoadLayout().setLayoutState(State.FAILED);
      }
    
         ...
       }`
    


## 总结 ##
  MVP模式相对于MVC模式最大的不同就是，MVC中， M层与V层是相通的，而MVP中是不相通的。 