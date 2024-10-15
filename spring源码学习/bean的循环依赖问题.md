**springboot2.6以后allowCircularReferences默认是false，也就是默认不支持用户使用自动解决循环依赖的功能，需要用户自己去理清依赖关系，建议通过重构代码、采取@Lazy注解等方式进行解决。可在配置文件中显式设置为true**

重要类:
DefaultSingletonBeanRegistry

singletonObjects（一级 new ConcurrentHashMap<>(256)）、singletonFactories（三级 new ConcurrentHashMap<>(16)）、earlySingletonObjects（二级 new HashMap<>(16)）三个map变量就是三级缓存

三级缓存里面放的是ObjectFactory<?>，函数式接口。可以将lambda表达式作为参数放到方法的实参中，在方法执行的时候，并不会实际的调用当前的lambda表达式，只有在调用getObject方法的时候才会去调用lambda表达式
@FunctionalInterface

一级就是完全创建好的bean（成品）
二级就是实例化完但是还没初始完的（半成品）
三级就是() -> getEarlyBeanReference(beanName, mbd, bean)，是一个lambda表达式
获取三级缓存里面的表达式getObject的时候有可能将代理对象换原始对象

代理对象的创建是在初始化过程的扩展阶段，而属性的赋值是在生成代理对象之前执行的，需要在前置过程的时候判断是否需要生成代理对象。
