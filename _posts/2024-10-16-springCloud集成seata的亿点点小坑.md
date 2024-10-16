---
title: springCloud集成seata的亿点点小坑
author: leopold hu
date: 2024-10-16
category: Jekyll
layout: post
---

背景：两个springCloud项目之间有调用插入数据库的业务，需要使用分布式事务确保数据的一致性，由于原来服务发现就是使用了阿里的nacos，所以分布式事务框架也选择回阿里的seata。

## 两个系统框架的版本如下
|框架名字|版本|
|--|--|
|jdk|17|
|springboot|3.2.7|
|org.springframework.cloud|2023.0.3|
|com.alibaba.cloud|2023.0.1.2|

上手第一件事当然是找官方文档看看有没有quick start啦，被我找到了，
[spring-cloud-alibaba-seata](https://github.com/alibaba/spring-cloud-alibaba/blob/2023.x/spring-cloud-alibaba-examples/seata-example/readme-zh.md)
照着文档所描述，先启动好seata-server2.0，连上了nacos，然后把配置上传到nacos配置中心里面，然后在系统里面引入好seata的依赖，加上配置文件。启动一切正常没有报错。好让我试一下触发一下请求，我的业务请求流程是这样子的。
1. 服务A先新增数据到数据库
2. 服务A调用feign请求服务B的接口
3. 服务B新增数据到数据库
4. 服务B回调信息
5. 服务A接收回调信息
6. 服务A执行下面的业务

我尝试在第六步的时候抛出一个RuntimeException，看看服务A和服务B数据会不会回滚。结果是服务A回滚了，服务B没有回滚。

初步想法是服务A的回滚只是本地事务触发了，而不是seata引起的。
在我反复对照官方demo和自己的配置没有任何差别以后，开始了排查之路。

## 了解SEATA的事务机制
因为我的数据库都是mysql，存储引擎是InnoDB，支持本地ACID事务，所以用的是AT模式。先回顾一下AT模式是怎么工作的，首先AT模式分成了两阶段提交

- 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
- 二阶段：提交异步化，非常快速地完成。 回滚通过一阶段的回滚日志进行反向补偿。

通过以上的描述，我首先是打开了seata的控制台http://localhost:7091，观察一下seata的事务运行得怎么样，于是我发现在调用请求的时候，全局事物有信息产生了
![image](https://github.com/user-attachments/assets/7a294b9e-783a-41ec-9675-4427d60bf0c6)
但是全局锁却没有信息
![image](https://github.com/user-attachments/assets/d5c940be-16a6-457b-8a7a-dda6657a5275)
于是我开始想，会不会是因为没有生成各自的分支事务导致他们没办法回滚。生成了全局事务信息，表明TC是正常工作的，那么有没有可能是RM没有正常工作？于是开始调试初始化的过程。设置seata的日志为debug，看到了seata的netty日志
```
10:50:37.730 [NettyClientSelector_TMROLE_1_1] DEBUG i.s.c.r.n.AbstractNettyRemotingClient - will send ping msg,channel [id: 0x932a5a1d, L:/192.168.3.2:56227 - R:/192.168.3.88:8091]
10:50:37.730 [NettyClientSelector_TMROLE_1_1] DEBUG i.s.c.r.netty.AbstractNettyRemoting - write message:services ping, channel:[id: 0x932a5a1d, L:/192.168.3.2:56227 - R:/192.168.3.88:8091],active?true,writable?true,isopen?true
10:50:37.730 [NettyClientSelector_RMROLE_1_1] DEBUG i.s.c.r.n.AbstractNettyRemotingClient - will send ping msg,channel [id: 0xa0076d16, L:/192.168.3.2:56230 - R:/192.168.3.88:8091]
10:50:37.730 [NettyClientSelector_RMROLE_1_1] DEBUG i.s.c.r.netty.AbstractNettyRemoting - write message:services ping, channel:[id: 0xa0076d16, L:/192.168.3.2:56230 - R:/192.168.3.88:8091],active?true,writable?true,isopen?true
10:50:37.730 [NettyClientSelector_TMROLE_1_1] DEBUG i.s.c.r.netty.AbstractNettyRemoting - io.seata.core.rpc.netty.TmNettyRemotingClient@56fcb842 msgId:25, body:services pong
10:50:37.730 [NettyClientSelector_RMROLE_1_1] DEBUG i.s.c.r.netty.AbstractNettyRemoting - io.seata.core.rpc.netty.RmNettyRemotingClient@4838668b msgId:25, body:services pong
10:50:37.731 [NettyClientSelector_TMROLE_1_1] DEBUG i.s.c.r.p.c.ClientHeartbeatProcessor - received PONG from /192.168.3.88:8091
10:50:37.731 [NettyClientSelector_RMROLE_1_1] DEBUG i.s.c.r.p.c.ClientHeartbeatProcessor - received PONG from /192.168.3.88:8091
```
netty工作是正常的，那就是RM是正常工作的，那么TC 、RM、TM之间是正常沟通的，但是为什么事务会不生效的呢？
后来再查阅相关的文章，发现原来这个seata的控制台里面的全局锁信息菜单，显示的其实是哪些资源被锁定了，这个锁定是为了防止并发事务出现事务不一致的问题，在当前的业务里面，仅仅只是各自的数据库新插入一条，TC根据全局事务和其他分支事务的状态，决定是否授予锁，并将决定返回给 RM，有可能是TC认为不需要加锁(为了优化速度)而导致在控制台没有看到有全局锁的信息。

## 排查undo_log表
接下来开始聚焦在undo_log表上，数据是根据undo_log表里面的记录进行回滚的，那么我得看一看undo_log表里面有没有记录到信息，重新调试业务，发现undo_log表里面并没有记录到数据。经查阅，记录undo_log表是在BaseTransactionalExecutor.prepareUndoLog()方法执行的，调试发现，在执行
```
if (beforeImage.getRows().isEmpty() && afterImage.getRows().isEmpty()) {
            return;
}
```
的时候直接return了，因为beforeImage和afterImage的rows都是0，这就很奇怪了，新插入数据的话，beforeImage是0的话正常，但是afterImage应该会有新插入的记录才对，于是接着往上查，afterImage是在AbstractDMLBaseExecutor.executeAutoCommitFalse()里面的TableRecords afterImage = afterImage(beforeImage);生成的，进入这个方法看到fpkValues返回的是id=0，但是seata如果想获取新增后的记录的话他是应该要根据新增的id获取记录，然后把记录放到undo_log里面的。
![image](https://github.com/user-attachments/assets/ca01ee2b-892b-4e31-8f12-beb100112833)
再进入这个方法里面查看，检查isContainsPk返回的是true，进入的是getPkValuesByColumn()方法去获取主键的id
![image](https://github.com/user-attachments/assets/bec3a325-5c33-49de-9348-df893098af8c)
接着往下看到了BaseInsertExecutor.parsePkValuesFromStatement()方法
![image](https://github.com/user-attachments/assets/5368646b-821e-4694-8270-81328f8ba435)
调试到这一步开始发现问题了，这个insert语句里面statementProxy已经准备好了id和id对应的值0，然后后面就直接拿了id里面的这个0值了，所以我推测在上一步里面getPkValuesByColumn()方法就是说这个语句里面已经包含了主键的信息，直接拿这里面的主键值，那么，为什么这里是0值而不是自增后的新的id值呢？

带着疑问接着看statementProxy是如何帮我们初始化的，我一直追溯到DefaultSqlSession.update()，可以看到这是一个mybatis的类，首先获取的是MappedStatement，这是一开始存储了SQL详细参数的信息，比如说sql语句在哪个位置，里面有什么类型参数之类的，然后到SimpleExecutor.doUpdate()，这时候会生成一个StatementHandler，这是准备执行SQL的对象，用来转换成JDBC可执行的东西，通过StatementHandler又生成了一个Statement，一直调试到这里，都可以看到插入的id依旧是0。
![image](https://github.com/user-attachments/assets/09f11f93-84e7-45bf-914f-d4f9ee4154f5)
直到执行到PreparedStatementHandler.update(Statement statement)这个方法，在执行execute的时候发现进入了PreparedStatementProxy.execute()里面，可以看到这里已经由mybatis转到seata了，证明seata已经正常自动代理了。这时候就能看出来问题了，mybatis把自增主键set回去实体类里面的是在PreparedStatementHandler执行完操作后，而seata代理进行记录undo_log表的操作是在PreparedStatementHandler执行操作的过程中，才导致了seata想获取新增主键值，但是值还是0。
![image](https://github.com/user-attachments/assets/ceefcd12-8574-4485-b992-1e07bf5a1244)

## 解决主键不自增的问题
同样的调试在serverA再试一次，我惊奇地发现，serverA是能正常在记录undo_log表的时候能获取到自增后的id的，这到底是怎么回事呢？观察PreparedStatement后发现里面的sql语句是不包含id的！
![image](https://github.com/user-attachments/assets/1a64ee05-771a-495b-9288-603538b12440)
对比一下两边的新增实体，serverA的数据是这样插入的
![image](https://github.com/user-attachments/assets/57260347-0d2a-4e06-9412-01b1f18d8359)
而serverB的数据，由于是serverA调用接口插入的不是自身系统new出来的对象插入，并且实体类里面id的类型是int，所以会存在默认值0。看到这里，会不会就是这个原因导致的？我尝试一下在serverA里面插入数据的时候显式set一个id=0，结果serverA依然能获取到自增后的id。
![image](https://github.com/user-attachments/assets/d5ffd06d-07ab-4815-91c0-c7abaf330ed5)
然后我试了一下在serverB里面显式地把id置为null。结果，serverB可以获取到自增后的id了，并且全局事务两个server都能正常回滚了。

## 继续查找导致两者区别的原因
我想知道，到底是什么原因导致明明是一样用mybatisplus插入，为什么两边却呈现两种不一样的结果？带着疑问继续调试源码，找一下是在哪里生成的targetSQL语句，在SimpleExecutor.doUpdate(MappedStatement ms, Object parameter)方法里面，看到了把parameter参数带进去newStatementHandler里面，而parameter里面是有字段和值的信息的，里面的new RoutingStatementHandler可以看到构造器里面会根据不同的类型生成不同的处理器，里面会一直调用到父类BaseStatementHandler的构造器，其中就看到了在这个地方生成了boundSql。
![image](https://github.com/user-attachments/assets/cfe44109-458f-4b9d-be83-09f7bdec75b5)
而这个generateKeys(parameterObject)方法看上去非常像是生成自增值的方法，再点进去看，wtf？
![image](https://github.com/user-attachments/assets/ed0058bf-74f3-4723-a7ba-1b9828e0297d)
经过查阅，原来这个插入前生成主键mysql不支持，Jdbc3KeyGenerator是在插入操作后才生成主键。所以这个generateKeys(parameterObject);并没有任何作用。那就直接看下面一行方法getBoundSql，就在
    `BoundSql boundSql = sqlSource.getBoundSql(parameterObject);`
这一个方法里面，是把插入的参数和值换成sql的，我发现了这个地方在serverB系统里面是一下这样的
![image](https://github.com/user-attachments/assets/22783bb4-fb53-4bd4-92bb-2ee2f07e3e81)
![image](https://github.com/user-attachments/assets/94061601-8005-4a11-b231-0b8b09542bd6)
但是在serverA系统里面是以下这样的
![image](https://github.com/user-attachments/assets/ecc72930-b647-404b-ba2e-10fc94ec7f95)
![image](https://github.com/user-attachments/assets/32454253-32ae-447d-b7e9-567e0e2550c0)
所以这里为什么parameterObject里面同样都存在id:0，但是经过getBoundSql以后，得到的boundSql一个是有id的，一个是没有id的。仔细一看原来是sqlSource的实现类不一样。serverA用的是RawSqlSource，而serverB用的是DynamicSqlSource。

## 探究sqlSource
先查阅一下资料，RawSqlSource是用来处理静态SQL语句，而DynamicSqlSource是用来处理动态SQL语句，所以RawSqlSource是在启动应用的时候mybatis就自动初始化好了，追踪到MybatisXMLScriptBuilder,parseScriptNode()方法，看到parseDynamicTags生成出来的对象就会设置好这个sql是否属于动态sql，再往下看parseDynamicTags(XNode node)方法，在textSqlNode.isDynamic()里面parser.parse(text)这个方法就是用来判断并且标记是不是动态sql的，查看里面的逻辑
![image](https://github.com/user-attachments/assets/80f0e2ca-43f0-4038-92d1-4373be6ae89c)
原来是看sql语句里面有没有包含${}这种动态SQL语句来判断的。
然后调试serverB，发现压根没有进来parseScriptNode()方法，然后再看MybatisXMLMapperBuilder.configurationElement(XNode context)的时候，发现serverB里面Mybatisplus并没有自动帮我生成一个insert语句，而serverA里面自动帮我生成了一个insert和一个update语句
![image](https://github.com/user-attachments/assets/e7ddd391-782f-4cee-9c0b-e7e723750481)
再次去查证，结果发现是serverA的xml文件里面写了insert方法和update方法，所以才会在MybatisSqlSessionFactoryBean.buildSqlSessionFactory()的时候就能扫描出来，然后后面的insert就是使用写好的这个语句了，才没有id！这才是导致两个server会有两种不同表现的原因。

## 怎么解决填充自增ID的步骤在执行seata之后？
经过上面的探索，首先明确了：1、无法插入undo_log是由于获取主键值异常导致seata select不到新增的记录数据。 2、造成一个服务正常获取主键值，一个服务不正常获取主键值的原因是正常获取的xml里面写了insert语句，mybatis调用的是xml里面的语句，而不是Mybatis-plus自动生成的。
然后我开始查找到底mybatis-plus是怎么帮我新增insert语句的，经过调试，启动程序的时候会调用DefaultSqlInjector.getMethodList初始化，其中一行
`.add(new Insert(dbConfig.isInsertIgnoreAutoIncrementColumn()))`
正正就是用来决定是否由mybatis-plus去掉主键字段的
![image](https://github.com/user-attachments/assets/52f85d8d-a286-4a90-82b5-0275cec5f8cc)
点进去看到注释，默认是false，也就是不去除，再结合我的实体类主键id的类型是int，会有默认值0，导致mybatis-plus没有去掉主键，而是塞了0值进来，再到seata查询到这个SQL语句里面已经包含了主键了，误以为已经把id放进来了，就直接使用了里面的0值来select，最终当然是查询不到，导致无法插入undo_log表。所以只需要把设置false换成true即可
![image](https://github.com/user-attachments/assets/135bf18d-3b07-445f-981e-77093265b9ca)
结束。