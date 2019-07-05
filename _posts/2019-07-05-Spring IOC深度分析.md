---
layout:     post
title:      spring ioc 原理
keywords:   spring, IOC, 源码分析
category:   spring
description: Spring IOC 源码深度分析，并总结其设计理念与思想
tags:		[java, spring]
---

# Spring IOC容器实现与源码分析
## 1、IOC位置
![](/images/spring/spring-overview.png)

-  核心容器包括spring-core、spring-beans、spring-context、spring-context-support、spring-expression，其中spring-core、spring-bean模块提供整个框架的基本功能，包含了IOC和依赖注入特性。本文也主要结合源码分析该部分，来看IOC设计的思想。

- 从该图可以体现分层架构思想。

## 2、 如何阅读源码

- 带着问题思考阅读IOC容器源码
  - 1、IOC容器解决什么问题？自己如何实现IOC，有哪些难点?
  - 2、IOC容器如何管理依赖注入的对象
  - 3、核心数据结构
  - 4、如何加载Bean的定义的，如何解析的
  - 5、两个Bean在不同的XML里，如何做解析的
  - 6、XML配置的Bean，依赖注入的Bean是通过注解初始化化的，又是如何解决依赖的
  - 7、如何考虑扩展性，如何对Bean的不同应用场景进行扩展
  - 8、IOC的构成体系、设计体系、主要思想是什么？
  - 9、BeanFactory如何考虑GC的？
  - 10、BeanFactory与FactoryBean区别？

## 3、设计与实现
### 3.1、工作流程
- ![](/images/spring/ioc工作原理.png)

### 3.2、接口概况
#### 3.2.1、容器接口
![](/images/spring/IOC.jpg)

#### 3.2.2、Context继承关系
![](/images/spring/context_class.jpg)

#### 3.2.3、Bean定义类图
![](/images/spring/BeanDefine.jpg)

#### 3.2.4、Bean解析类图
![](/images/spring/BeanParser.jpg)

### 3.3 接口设计理解
#### 3.3.1 BeanFactory方法概览
```java
public interface BeanFactory {

	String FACTORY_BEAN_PREFIX = "&";

	Object getBean(String name) throws BeansException;

	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	<T> T getBean(Class<T> requiredType) throws BeansException;

	Object getBean(String name, Object... args) throws BeansException;

	boolean containsBean(String name);

	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
	
	boolean isTypeMatch(String name, Class<?> targetType) throws NoSuchBeanDefinitionException;

	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	String[] getAliases(String name);
}
```
#### 3.3.2 基础容器
- 从BeanFactory--->HierarchicalBeanFactory--->ConfigurableBeanFactory该设计是容器的基础功能，也是容器的标准实现的规范。BeanFactory提供getBean基本方法，HierarchicalBeanFactory增加了getParentBeanFactory获取父工厂方法，联想Java的类加载机制，这不是双亲委派么？ConfigurableBeanFactory提供了容器的配置功能，如setParentBeanFactory设置双亲委派，addBeanPostProcessor添加后置处理器等。
 - HierarchicalBeanFactory为什么只有设置父容器方法？

#### 3.3.3 高级容器
- 从BeanFactory--->ListableBeanFactory---->ApplicationContext为一条设计路线，该设计主要是面对开发者使用，屏蔽底层基础容器的实现。ListableBeanFactory提供可返回管理对列表的能力，ApplicationContex同时实现了资源加载、国际化、事件监听机制等高级特性

### 3.4 主要思想
源码学习的目的在于学习借鉴别人的设计思路，使自己的思路开阔与升华，通过学习总结以下相关思想观点
- 依赖注入,通过配置文件和注解的方式来解决该问题
- 工厂模式
- 模板模式
- 分层设计
- 单一职责
- Spring IOC容器表现形式为BeanFactory与ApplicationContext,前者是最基本的容器，提供基础功能，后者是容器的高级形态，提供应用环境的适配与高级功能



## 4、源码分析
### 4.1 Bean加载深度分析
![](/images/spring/classpathContext时序.jpg)

#### 4.1.1、构造方法
```java
    public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
                throws BeansException {
            super(parent);
            setConfigLocations(configLocations);
            if (refresh) {
                refresh();
            }
        }
``` 
- 构造参数：
    - 1、资源数组路径
    - 2、是否刷新上下文
    - 3、父上下文对象
- 构造方法工作：
    - 1、初始化资源解析器 
    - 2、设置资源信息
    - 3、解析刷新上下文（核心流程）

#### 4.1.2、Refresh源码
refresh是AbstractApplicationContext的方法，该抽象类定义了IOC容器的一系列基础方法与抽象方法（模板设计模式），refresh定义了加载与初始化的流程。可以看到该方法内部有synchronized，主要是避免多线程同时刷新上下文，不是加在整个方法体而是对象startupShutdownMonitor上呢？该方法设计的流程清晰

```java
    	synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}
		}
```

##### 4.1.2.1 obtainFreshBeanFactory方法
obtainFreshBeanFactory用来刷新Spring工厂的上下文，代码如下
```java
    	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```
- 其中refreshBeanFactory是核心实现，该方法是抽象方法又两个子类分别进行了实现AbstractRefreshableApplicationContext和GenericApplicationContext，从上面的时序图得知目前调用的AbstractRefreshableApplicationContext的实现，代码如下

```java
    protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```
- 此方法有两个核心实现一个是创建DefaultListableBeanFactory，一个是loadBeanDefinitions加载Bean的定义，内部的核心

### 4.2 主要流程总结
- 1、IOC容器主要以下几个阶段：
   - 1)、bean的定义阶段 
   - 2)、bean的解析阶段 
   - 3)、bean的实例化阶段
   - 4)、使用Bean阶段

- 2、Bean的定义阶段主要经历几个步骤：
   - 1）读取资源文件存放到一个数组里面 
   - 2）初始化xmlreder器 
   - 3）装载bean定义文件 
   - 4）通过流读入资源文件转换为docment对象 
   - 5）注册Bean定义，也就是真正解析XML，分为解析默认和解析自定义

 - 3、双亲委派的意义，应用场景？
      回顾类加载的双亲委派机制，每个加载器加载相应的class对象，避免多次加载同一个对象，减少内存开销（类A中使用了List，类B也会使用List，都各自加载各自的就悲剧了）。同样如此Bean对象之间会相互依赖同一个对象。

