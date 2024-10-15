1、首先了解spring bean是个啥玩意
bean其实就是对象，是一个可以让spring进行管理的对象，他在代码里面是叫BeanDefinition，他是通过反射出来的

2、bean的实例化
实例化就是在JVM堆内存内申请空间，反射创建对象，然后设置好默认的属性
重点类：AbstractAutowireCapableBeanFactory
这个类负责几个关键任务：

- bean的创建
- 属性注入
- 初始化
- 代理处理

在方法createBeanInstance中进行创建，获取类的构造器，然后通过构造器的newInstance获取

3、实例化完了之后开始对属性进行赋值
3.1、自定义属性赋值
在populateBean方法里面进行set操作，首先从RootBeanDefinition中获取所有需要注入的属性值PropertyValues

3.2、容器对象属性赋值、
检查所有实现了Aware接口的类，把相关的容器对象set到对应实现了Aware的属性里面

4、得到了bean对象后再对其进行扩展
关键类：BeanPostProcessor