---
title: tomcat之初始化Catalina容器
author: leopold hu
date: 2024-10-30
category: Jekyll
layout: post
---

初始化的主要脉络  
首先看Bootstrap的main，程序的入口，new了一个新的Bootstrap对象后会执行init方法  
执行完init方法以后接着后面执行daemon.load方法，在load方法中可以看到通过反射调用了Catalina（这个在上面的init方法里面已经设置好了类是org.apache.catalina.startup.Catalina）的load方法  
Catalina.load里面会初始化一个Server  
Server的主要职责：
1. 生命周期管理
2. 管理多个Service
3. 配置和初始化
4. 监听和处理全局事件
5. 管理JNDI资源

接着进行Server的init，实际调用的是StandardServer的initInternal，处理完JNDI的初始化后开始for循环执行service的init  
实际上调用的是StandardService的init方法，一个Server里面可以有很多个Service，Service是用来将**多个**连接器和**一个**容器组合在一起的，提供完整的请求处理功能。  
初始化完Service后，再为这个Service初始化容器，容器是负责处理请求的核心组件，能够根据URI定位到对应的应用或资源，通常是StandardEngine，获取realm对象  
初始化完容器以后，下一步就开始初始化连接器，可以看到用的是for循环把所有的Connector都进行init，Connector首先会设置一个Adapter  
Adapter是用来适配和协调连接器和容器之间的接口，它将连接器接收到的网络请求转换为容器能够理解和处理的格式，并将容器返回的响应数据发送回客户端。他将来自连接器的原始请求（HTTP请求或者其他协议的数据流）转换为Tomcat内部的Request和Response对象，这样容器才能处理它（容器接受到请求后会进一步分发请求到正确的Host、Context），容器处理完后又会从Response对象提取响应数据，变回网络传输的格式给回Connector发送给客户端。使用的是CoyoteAdapter，接下来就是调用protocolHandler的init方法  
protocolHandler用于处理与特定协议相关的底层网络通信，将收到的字节流转换为符合特定协议的请求对象，然后交给Adapter


ProtocolHandler 和 Adapter 的区别
======

ProtocolHandler：处理 低层次的网络通信和协议解析。它负责在网络层接收客户端的字节流数据，将其转换为特定协议的内容，例如 HTTP 请求头、URI、HTTP 方法等具体信息。这是 Tomcat 中对不同协议（如 HTTP、AJP 等）的适配器，用于确保 Tomcat 支持多种协议。
Adapter：关注 高层次的请求和响应封装，它是 Tomcat 内部的一个桥梁，负责将 ProtocolHandler 解析出的请求内容，转化为 Tomcat 容器（如 Engine）可以理解和处理的标准请求、响应对象。Adapter 还将容器处理完成的 Response 结果再传递回 ProtocolHandler。

ProtocolHandler：负责读取、解析低层协议数据流，例如将原始 HTTP 字节流解析为 HTTP 请求的头部、方法、路径、参数等内容。对于 ProtocolHandler 来说，它的主要任务是将底层网络协议转化为 Tomcat 标准的 Request 和 Response 对象。
Adapter：进一步将 Request 和 Response 对象转发给 Engine，调用容器的实际业务逻辑。这包含了 Tomcat 的核心业务处理过程，例如将请求分发到相应的 servlet 等。处理完后，Adapter 将 Engine 生成的响应返回给 ProtocolHandler。

ProtocolHandler 是协议相关的：它包含特定协议的实现，比如 HTTP、AJP 等。不同协议会有不同的 ProtocolHandler 实现，以支持协议的不同特性。
Adapter 是协议无关的：Adapter 主要关注如何将标准的 Request 和 Response 对象传递给容器处理，而无需关心请求来自 HTTP 还是 AJP 协议。

ProtocolHandler：处于请求处理的最底层。ProtocolHandler 负责接收并解析客户端的网络请求，建立 Request 和 Response 对象。
Adapter：位于 ProtocolHandler 和 Engine（容器）之间。它负责从 ProtocolHandler 中获取 Request，并将其传递给 Engine 进行实际的业务处理。  

工作流程的具体分工  
请求进入：ProtocolHandler 接收来自客户端的网络请求，并解析字节流。  
请求解析：ProtocolHandler 将字节流解析成协议级别的内容，创建 Request 和 Response 对象。   
交由 Adapter 处理：Adapter 获取 Request 和 Response 对象，调用 Engine 的业务逻辑。  
业务处理完成：Engine 处理完后，Adapter 将结果返回给 ProtocolHandler。  
响应发送：ProtocolHandler 将处理后的响应内容转化为客户端协议可以识别的格式（如 HTTP 响应格式），并返回给客户端。

然后执行endpoint的init方法，进入AbstractEndpoint的init方法，里面的bindWithCleanup方法用来初始化服务器套接字，调用的是NioEndpoint的bind来开启Nio的socket服务器。