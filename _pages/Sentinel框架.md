---
title: sentinel框架
author: leopold hu
date: 2024-10-25
category: Jekyll
layout: post
---

1、SentinelWebAutoConfiguration进行自动装配，里面有一个bean是sentinelWebInterceptor，他会new一个SentinelWebInterceptor。SentinelWebInterceptor继承AbstractSentinelInterceptor，而AbstractSentinelInterceptor又是实现了HandlerInterceptor。即对controller层进行拦截  
2、获取resourceName，去到RequestMapping里面的属性值  
3、获取contextName，默认的name是**sentinel_spring_web_context**  
4、获取origin，基于自定义的RequestOriginParser  
5、初始化Context，ContextUtil.enter(contextName,origin)->创建EntranceNode->创建Context,放入ThreadLocal  
6、标记资源，创建Entry，Entrt e = SphU.entry(resourceName)->执行ProcesserSlotChain  

滑动时间窗口算法  
