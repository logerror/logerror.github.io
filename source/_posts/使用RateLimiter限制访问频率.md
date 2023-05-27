---
title: 使用RateLimiter限制访问频率
categories: Java
tags:
  - qps ratelimiter
description: 使用RateLimiter限制访问频率
abbrlink: 1900677851
date: 2019-06-12 17:23:44
---

****背景****     
通过查看接口访问日志发现某个注册接口请求量突然暴涨，并引发连锁反应，导致整个系统变慢。系统已经对接口做了做了权限认证，包含token认证，时效认证，禁用启用等功能，但没有对访问频率做限制，除了需要优化处理逻辑外，也要加上对接口的访问频率限制，以防止非预期的请求对系统压力过大而引起的系统瘫痪，当流量过大时，可以采取拒绝或者引流等机制。


****原因及方案分析****    

常用的限流算法有两种：漏桶算法和令牌桶算法。

漏桶(Leaky Bucket)算法思路很简单,水(请求)先进入到漏桶里,漏桶以一定的速度出水(接口有响应速率),当水流入速度过大会直接溢出(访问频率超过接口响应速率),然后就拒绝请求,可以看出漏桶算法能强行限制数据的传输速率。但是漏桶的漏出速率是固定的参数,所以,即使网络中不存在资源冲突(没有发生拥塞),漏桶算法也不能使流突发(burst)到端口速率.因此,漏桶算法对于存在突发特性的流量来说缺乏效率.
![](https://i.imgur.com/doxSzsv.png)    
令牌桶算法(Token Bucket)和 Leaky Bucket 效果一样但方向相反的算法,更加容易理解.随着时间流逝,系统会按恒定1/QPS时间间隔(如果QPS=100,则间隔是10ms)往桶里加入Token(想象和漏洞漏水相反,有个水龙头在不断的加水),如果桶已经满了就不再加了.新请求来临时,会各自拿走一个Token,如果没有Token可拿了就阻塞或者拒绝服务.令牌桶的另外一个好处是可以方便的改变速度. 一旦需要提高速率,则按需提高放入桶中的令牌的速率. 一般会定时(比如100毫秒)往桶中增加一定数量的令牌, 有些变种算法则实时的计算应该增加的令牌的数量.

![](https://i.imgur.com/WCQ32wH.png)
****解决思路****

使用令牌桶算法限制接口访问频率sss。

****详细实现****    
	
   Google开源工具包Guava提供了限流工具类RateLimiter，该类基于令牌桶算法来完成限流，非常易于使用。在项目中用了以下实现 

   1.频率是可以设置的，所以要存储初始值，频率发生改变时调用setRate方法即可，需要注意的是setRate的意义是每秒允许多少次访问，参数是一个double类型的值

		RateLimiter rateLimit = RestComponentFactory.interfaceRateMap.getOrDefault(interfaceVo.getToken(), null);
		Double newRateLimit = interfaceVo.getQps();
		if(interfaceVo.getQps()!= null && interfaceVo.getQps() > 0){
			if(rateLimit == null){
				rateLimit = RateLimiter.create(interfaceVo.getQps());
				RestComponentFactory.interfaceRateMap.put(interfaceVo.getToken(), rateLimit);
			}else{
				if(rateLimit.getRate() != newRateLimit){
					rateLimit.setRate(newRateLimit);
				}
			}

			boolean rateFlag = rateLimit.tryAcquire();
			if(!rateFlag){
				throw new Exception470();
			}
		}
 



****补充说明**** 
RateLimiter类位于  

com.google.common.util.concurrent.RateLimiter

	<artifactId>guava</artifactId>
	<groupId>com.google.guava</groupId>





​ 
​