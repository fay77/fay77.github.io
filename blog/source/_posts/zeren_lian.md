---
title: Okhttp源码应用到的责任链模式
date: 2018-02-27 15:32:54
tags:
---
<!--more-->

# 前言

最近在看`OkHttp`源码，在整个`OkHttp`中最重要也是最核心的就是拦截器了，简单看拦截器似乎就是一个个实现`Interceptor`接口的类，再分别在类中实现相应的逻辑，直到我看到下面的代码

    	private Response getResponseWithInterceptorChain() throws IOException {
     	 // Build a full stack of interceptors.
     	 List<Interceptor> interceptors = new ArrayList<>();
    	  interceptors.addAll(client.interceptors());
     	 interceptors.add(retryAndFollowUpInterceptor);
     	 interceptors.add(new BridgeInterceptor(client.cookieJar()));
     	 interceptors.add(new CacheInterceptor(client.internalCache()));
    	  interceptors.add(new ConnectInterceptor(client));
    	  if (!retryAndFollowUpInterceptor.isForWebSocket()) {
    		interceptors.addAll(client.networkInterceptors());
      	}
      	interceptors.add(new CallServerInterceptor(
      	retryAndFollowUpInterceptor.isForWebSocket()));
    
      	Interceptor.Chain chain = new RealInterceptorChain	(
      		interceptors, null, null, null, 0, originalRequest);
      	return chain.proceed(originalRequest);
   	 }

`上述代码中加入了很多个拦截器，这么多拦截器是怎么一个个实现的？`

# 责任链模式 #
**责任链模式将各个拦截器连接环环相扣，最终完成一次网络请求。**


----------
**一个例子说明责任链应用场景**


    public class BrowRequest {
    	private int requestMoney;
    
    	public BrowRequest(int money) {
    		System.out.println("有新请求，需要借款" + money + "元");
    		requestMoney = money;
    	}
    
    	public int getRequestMoney() {
    		return requestMoney;
    	}
    }
    



----------
最核心的类， 处理类，注意`    				this.superior.approveRequest(browRequest);`简单看似乎是自己调自己的方法，其实不然，这句代码将处理交给了下一个对象`setSuperior()`这个方法就是设置下一个传递对象，类似击鼓传花，将花递给了下一个

    abstract class AbstractClerk {
    	private AbstractClerk superior = null;
    	protected String type;
    	public abstract int getLimit();
    
    	public void setSuperior(AbstractClerk superior) {
    		this.superior = superior;
    	}
    
    	public void approveRequest(BrowRequest browRequest) {
    		if (getLimit() >= browRequest.getRequestMoney()) {
    			System.out.println(getType() + "同意借款请求");
    		} else {
    			if (this.superior != null) {
    				//传递给下一个使用
    				this.superior.approveRequest(browRequest);
    			} else {
    				System.out.println("没有人能够同意借款请求");
    			}
    		}
    	}
    
    	public String getType() {
    
    		return type;
    
    	}
    }



----------
接着就比较简单了，定义我们的具体类，继承处理类

    public class Clerk extends AbstractClerk {
    
    	public Clerk() {
    		super.type = "职员";
    	}
    
    	@Override
    	public int getLimit() {
    		return 5000;
    	}
    }

    
    public class Leader extends AbstractClerk {
    	public Leader() {
    		super.type = "组长";
    	}
    
    	@Override
    	public int getLimit() {
    		return 20000;
    	}
    }


    public class Manager extends AbstractClerk {
    	public Manager() {
    		super.type = "经理";
    	}
    
    	@Override
    	public int getLimit() {
    		return 100000;
    	}
    }



----------
最后在客户端调用下就行了

        public class Client {
    	public static void main(String[] args) {
    		AbstractClerk clerk = new Clerk();
    		AbstractClerk leader = new Leader();
    		AbstractClerk manager = new Manager();
    
    		clerk.setSuperior(leader);
    		leader.setSuperior(manager);
    
    
    		//有人借款10000元
    		clerk.approveRequest(new BrowRequest(10000));
    
    		//有人借款111000元
    		clerk.approveRequest(new BrowRequest(100000));
    	}
    }

可以看到责任链跟现实生活中的击鼓传花游戏非常相似，很多对象由每一个对象对其下家的引用而连接起来形成一条链。

最核心的方法是处理类， 一般处理类会包含一个指向下一处理类的成员变量，和一个处理请求的方法 ， 如果符合条件就自己处理，否则就传递给下一个处理。
