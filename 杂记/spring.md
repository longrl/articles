### spring基于事件驱动
applicationcontext初始化时
注册事件广播器
广播器ApplicationEventMulticaster和监听器ApplicationListener

采用的就是设计模式中的观察者模式，具体可以看AbstractApplicationContext中的refresh函数
```java
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			//初始化上下文信息
			prepareRefresh();
			//初始化初级容器BeanFactory,并解析XML文件
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			//对Spring容器BeanFactory做一些准备工作
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				//用于注册特殊的后处理器来加载特殊的一些bean
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				//执行BeanFactory的后处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				//在容器中注册bean的后处理器
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				//在容器中初始化消息源MessageSource
				initMessageSource();

				// Initialize event multicaster for this context.
				//初始化事件广播器
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				//空实现   用于实例化bean之前，做一些其他初始化bean的工作
				onRefresh();

				// Check for listener beans and register them.
				//在容器中初始化各种监听器
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				//预先实例化非延迟加载的单例
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				//初始化生命周期处理器，并发出相应的事件进行通知
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}    
```

### bean加载
bean的信息从配置文件解析出来，设置到BeanDefinition之中，然后注册到spring容器之中；

bean加载的入口：getBean()
1、别名解析并转换
`final String beanName = transformedBeanName(name);`
去除前缀中的'&'

通过三级缓存解决循环依赖
singl![三级缓存](https://user-images.githubusercontent.com/49087641/159655003-77149902-28b5-4aa7-9c66-cd5ee8591044.png)
etonObjects、earlySingletonObjects、singletonFactories

Spring默认是解决了setter注入的循环依赖的，构造方法循环依赖问题，是在反射创建bean时就会发生的，此时Spring是没有办法提前获取到早期单例bean的，因为早期单例bean得要经过反射创建才能获取到

