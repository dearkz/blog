---
title: 手写mybatis插件
author: leopold hu
date: 2024-10-16
category: Jekyll
layout: post
---

一个接口，四大对象，分别为：  
org.apache.ibatis.plugin.Interceptor  
接口定义了三个方法  
intercept:拦截器，传入的Invocation有proceed()方法，即用来执行下一步的方法，前拦截器从外进内，后拦截器从内出外  
plugin:包装代理对象，加入拦截器链  
setProperties:用xml可以设置属性  

org.apache.ibatis.executor.Executor  执行器  
org.apache.ibatis.executor.parameter.ParameterHandler  参数处理  
org.apache.ibatis.executor.resultset.ResultSetHandler  返回集处理  
org.apache.ibatis.executor.statement.StatementHandler  sql语法构建

在Executor中，执行查询的时候会通过全局配置configuration的newStatementHandler来初始化StatementHandler，这里面就会包括各种handler，如果有配置到拦截器，并且有接口的，就使用JDK代理生成Proxy覆盖，然后在具体查询的过程中就会调用到这些handler，实际使用代理类来进行调用interceptor接口里面的intercept方法