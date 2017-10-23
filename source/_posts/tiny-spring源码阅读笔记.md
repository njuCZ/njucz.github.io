title: tiny-spring源码阅读笔记
date: 2016-08-20 20:08:45
tags: [java, spring]
---
要阅读spring的源码，理解spring中的核心概念,比如ioc,aop的实现，比如ApplicationContext和AutowireCapableBeanFactory的具体职责，在未深入了解前总觉得很虚幻。要想具体了解它们的职责，同时避开spring庞杂的系统代码，可以推荐[tiny-spring](https://github.com/code4craft/tiny-spring)项目,该项目实现一个微型的spring，spring中最重要最基础的功能都包含在内，可以有效的帮助快速入门，理解spring的精髓。

tiny-spring项目把重要的一些历史版本都进行了tag管理。共分为了十个版本，接下来我将记录下每个里程碑版本的一些理解。

#### step-3-inject-bean-with-property

首先理清下bean的概念：

> The objects that form the backbone of your application and that are managed by the Spring IoC container are called beans

bean指的是任何通过spring管理生成的object，和普通的object的唯一区别便是是否由spring的容器所管理。在spring中，每个bean都是以BeanDefinition的形式存在的。大家都知道ioc的核心实现是通过反射完成的，要通过反射获得一个object，那么该object的class，该object的成员变量，以及表示该object的name都是必需的属性。所以大家可以看到BeanDefinition包含以下属性：
```
private Object bean;
private Class beanClass;
private String beanClassName;
private PropertyValues propertyValues;
```
BeanDefinition对象在创建时只需知道beanClass，beanClassName和propertyValues属性，根据这些属性创建对应的对象则是由工厂类负责创建.

工厂类负责构造对象，所以会包含ConcurrentHashMap<String, BeanDefinition>，其中的key是BeanDefinition的beanClassName。子类AutowireCapableBeanFactory负责具体的创建bean的过程，即创建对象，注入属性。

#### step-5-inject-bean-to-bean
step-4和step-5主要做一件事，读取解析xml，完成bean的构造。
- Resource相关的类用于定义资源文件
- BeanDefinitionReader相关类定义了获得beanDefinition的过程，所以AbstractBeanDefinitionReader类会包含两个成员变量，一个是代表获取资源文件，另一个是内部保存读取的beanDefinition
- AutowireCapableBeanFactory类负责对象创建，属性注入（普通属性以及对象属性）

XmlBeanDefinitionReader负责对xml文件进行解析,获取定义的bean的name，class以及property，在property这里会根据是value属性还是ref属性来判断到底是内置类型变量还是引用类型变量。BeanReference代表的就是引用类型变量，之后在对bean的成员变量初始化的时候就是判断PropertyValue的value是否为BeanReference类型来分别进行赋值。

AbstractBeanFactory的registerBeanDefinition方法是为了将XmlBeanDefinitionReader读取到的beanDefinition注册进来。getbean方法的流程是
1. 从map中根据bean的name获取BeanDefinition
2. 如果BeanDefinition不存在就抛出异常
3. 获取beanDefinition的成员变量bean;
4. 如果bean不是null直接返回，如果是null就调用子类AutowireCapableBeanFactory进行对象创建

AutowireCapableBeanFactory负责具体的bean创建工作，即利用反射创建新对象，然后设置属性。在设置属性的时候如果发现是BeanReference的类型，那么重新调用getbean方法

#### step-6-invite-application-context
在step-5中有两大部分：Reader和Factory，而且他们之间是有调用关系的。而ApplicationContext就是负责协调Reader和factory，提供对外的接口。
- 对外提供getbean方法（调用factory的getbean方法）
- 对内的refresh方法（初始化reader，读文件，注册到factory中）

#### 聊聊AOP
在聊聊该项目的具体实现之前，我们先来了解一下aopalliance这个项目。
> AOP Alliance intends to facilitate and standardize the use of AOP to enhance existing middleware environments (such as J2EE), or development environements (e.g. JBuilder, Eclipse)

简而言之，aopalliance是为了统一aop的接口，为不同的aop实现提供更好的兼容性。这个项目主要规范了一些接口。主要的接口如下图所示：
![](/img/aopalliance.png)

该图中所示的类为aop概念中核心的关键类，下面简单介绍下每个类的概念及其作用。

- advice: describes a class of functions which modify other functions when the latter are run; it is a certain function, method or procedure that is to be applied at a given join point of a program.
- Interceptor: 继承于advice，细化advice的概念为拦截器。子接口有MethodInterceptor, ConstructorInterceptor, FieldInterceptor
- Join point: a point during the execution of a program, such as the execution of a method or the handling of an exception.此接口是运行时joinpoint的通用形式。该接口声明了三个方法：
  - Object proceed() throws Throwable; 触发拦截链上下一个拦截器
  - AccessibleObject getStaticPart(); 程序运行到joinpoint时要被代理的本身的那段代码，即所谓的静态部分
  - Object getThis(); 返回持有当前运行时joinpoint的实际对象
- Invocation: a joinpoint and can be intercepted by an interceptor. 运行时的要被代理的一个调用点。子接口类中分为两个ConstructorInvocation和MethodInvocation，不论调用方法还是构造函数，都需要明确参数，所以该接口定义了方法： Object[] getArguments();
- MethodInvocation： 即调用哪个函数，定义的新方法：Method getMethod();
- MethodInterceptor： 如图所示，MethodInterceptor继承Interceptor接口，并调用MethodInvocation。

了解这些核心概念，有利于在实现aop时更好的理解各个参数和方法的确切含义，并加以运用。

#### step-7-method-interceptor-by-jdk-dynamic-proxy
JDk提供的的核心代理类是Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h), 返回一个代理的实例，任何对该实例的调用都会经过InvocationHandler的处理。InvocationHandler则提供了一个invoke方法，用户可以在invoke方法里调用原本的方法，并添加其他的处理逻辑。

JdkDynamicAopProxy就是负责上述的逻辑。因为此处JdkDynamicAopProxy是一个通用的代理类，需要做的额外的处理都各不相同，所以在invoke方法的实现引入了MethodInterceptor和MethodInvocation。MethodInterceptor由用户自己的逻辑产生，程序需要自己定义MethodInvocation。

AdvisedSupport则提供一些代理的元数据：拦截的方法，目标对象，对象的class。

***注意此时的invoke顺序是JdkDynamicAopProxy.invoke() -> MethodInterceptor.invoke() -> MethodInvocation.invoke() -> original mehtod()***
```
// 设置被代理对象(Joinpoint)
AdvisedSupport advisedSupport = new AdvisedSupport();
TargetSource targetSource = new TargetSource(helloWorldService, HelloWorldService.class);
advisedSupport.setTargetSource(targetSource);

// 设置拦截器(Advice)
TimerInterceptor timerInterceptor = new TimerInterceptor();
advisedSupport.setMethodInterceptor(timerInterceptor);

// 创建代理(Proxy)
JdkDynamicAopProxy jdkDynamicAopProxy = new JdkDynamicAopProxy(advisedSupport);
HelloWorldService helloWorldServiceProxy = (HelloWorldService) jdkDynamicAopProxy.getProxy();

// 基于AOP的调用
helloWorldServiceProxy.helloWorld();
```

***多参考test case: JdkDynamicAopProxyTest***

#### step-9-auto-create-aop-proxy
为了能够自动的创建代理，那么应该在factory的createBean时完成相关工作。在这里，我们先引入BeanPostProcessor接口来代表完成bean初始化的工作，AspectJAwareAdvisorAutoProxyCreator就是负责在初始化时自动对对象进行代理。

首先来看看一个advisor类的表示
```
<bean id="aspectjAspect" class="us.codecraft.tinyioc.aop.AspectJExpressionPointcutAdvisor">
  <property name="advice" ref="timeInterceptor"></property>
  <property name="expression" value="execution(*us.codecraft.tinyioc.*.*(..))"></property>
</bean>
```
在AspectJAwareAdvisorAutoProxyCreator的postProcessAfterInitialization会首先获得所有的advisor类，然后遍历AspectJExpression是不是匹配当前的class。若是匹配，就会将当前的bean包装成一个JdkDynamicAopProxy的代理，并设置相关的代理方法。

因为bean包含了普通的对象，拦截器对象还有BeanPostProcessor对象，那么这些对象的初始化有没有什么顺序呢？
首先看看AbstractApplicationContext的refresh()方法：
1. loadBeanDefinitions() 读取xml中bean的定义
2. registerBeanPostProcessors() 会从bean列表中找到BeanPostProcessor类型的bean，*并完成初始化*，然后保存在beanFactory的列表中
3. 初始化其他的bean(初始化每个bean的时候都会依次调用每个BeanPostProcessor的两个方法的)

#### step-10-invite-cglib-and-aopproxy-factory
首先我们来聊聊cglib和JDk内置的proxy的异同:
> JDK Dynamic proxy can only proxy by interface (so your target class needs to implement an interface, which will also be implemented by the proxy class).

> CGLIB (and javassist) can create a proxy by subclassing. In this scenario the proxy becomes a subclass of the target class. No need for interfaces.

此外，使用JDK Dynamic proxy对所有的方法访问都会经过代理，而cglib可以选择对部分方法进行代理，效率也更高。

接下来来看看cglib如何实现代理的。
```
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(advised.getTargetSource().getTargetClass());
enhancer.setInterfaces(advised.getTargetSource().getInterfaces());
enhancer.setCallback(new DynamicAdvisedInterceptor(advised));
Object enhanced = enhancer.create();
```
DynamicAdvisedInterceptor是对net.sf.cglib.proxy.MethodInterceptor的一个实现，也就是在指定的CglibMethodInvocation处调用advised的methodInterceptor

至此，一个大概的支持ioc和aop的微型spring就完成了。
