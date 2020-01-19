---
title: 谈谈SpringBoot的监听器
date: 2020-01-17 14:55:00
tags: 
    - SpringBoot
    - 源码
categories: SpringBoot
---
&ensp;&ensp;&ensp;&ensp;最近在看SpringBoot的源码，在SpringBoot项目启动的过程中，监听器在不同阶段都会监听相应的事件，今天我们就来谈谈SpringBoot启动过程中的监听器。
## 一、监听器扫盲  
&ensp;&ensp;&ensp;&ensp;监听器模式，顾名思义就是某个对象监听某个或某些事件的触发，然后做出相应的操作。这句话可以看出，监听器模式包含几个特定的元素：  
- 事件，即监听什么（Event）
- 监听者，即谁来监听（Listener）
- 广播器，即谁来发布事件（Multicaster）
- 事件触发机制，即事件什么时候发布   

整体的原理如下所示。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/%E7%9B%91%E5%90%AC%E5%99%A8%E6%A8%A1%E5%BC%8F.png" width="100%" height="100%">  
</div>
&ensp;&ensp;&ensp;&ensp;当系统运行在某些关键节点的时候，会通过广播器去发布一些事件，而系统中存在着一些监听器，对某些事件感兴趣，去订阅这些事件。当这些事件被发布出去之后，监听器监听到这些事件，会触发一些行为。  
&ensp;&ensp;&ensp;&ensp;这就是监听器的简单解释，那么在SpringBoot中监听器是如何实现的呢？接下来我们就来看看吧！

## 二、揭开面纱 深入肌理
&ensp;&ensp;&ensp;&ensp;在SpringBoot中，系统<font color=#008000 >监听器</font>是 ApplicationListener，可以看到源码的注释，通过实现这个接口来实现监听器。   
{% codeblock ApplicationListener.java lang:java %}
package org.springframework.context;

import java.util.EventListener;

/**
 * Interface to be implemented by application event listeners.
 * 
 * 通过实现这个接口来实现监听器
 * 这个接口是按照监听器模式的标准来设计的
 * <p>Based on the standard {@code java.util.EventListener} interface
 * for the Observer design pattern.
 * 
 * 在Spring 3.0之后，一个应用监听器通常可以定义自己感兴趣的事件。当注册到Spring容器之后，当程序运行到一些关键节点时，
 * 会发出这些事件，并根据对应事件筛选出感兴趣的监听器进行触发。
 * <p>As of Spring 3.0, an {@code ApplicationListener} can generically declare
 * the event type that it is interested in. When registered with a Spring
 * {@code ApplicationContext}, events will be filtered accordingly, with the
 * listener getting invoked for matching event objects only.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @param <E> the specific {@code ApplicationEvent} subclass to listen to
 * @see org.springframework.context.ApplicationEvent
 * @see org.springframework.context.event.ApplicationEventMulticaster
 * @see org.springframework.context.event.EventListener
 */
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}
{% endcodeblock %}  
这个接口继承自 EventListener 接口，看源码可知 EventListener 接口就是一个接口定义，声明这是一个事件监听的接口  
{% codeblock EventListener.java lang:java %}
package java.util;

/**
 * A tagging interface that all event listener interfaces must extend.
 * @since 1.1
 */
public interface EventListener {
}
{% endcodeblock %}  
&ensp;&ensp;&ensp;&ensp;上面的 ApplicationListener 接口还有一个泛型，继承自 ApplicationEvent，这就是说在实现这个接口的时候，可以声明自己感兴趣的事件。系统在触发这个系统监听器的时候会根据其感兴趣的事件做一个过滤。这个接口定义了一个 onApplicationEvent 方法，是当它监听到事件发生的时候，会去做什么事情。  
&ensp;&ensp;&ensp;&ensp;接下来我们看一下监听器模式的<font color=#008000 >【广播器】</font>在SpringBoot中的实现。  
&ensp;&ensp;&ensp;&ensp;系统广播器是 ApplicationEventMulticaster ，实现这个接口来管理一些应用监听器，并且广播事件。其中定义了添加、删除监听器的方法以及广播事件的方法。   
{% codeblock ApplicationEventMulticaster.java lang:java %}
/**
 * 实现这个接口来管理一些应用监听器，并且广播事件
 * Interface to be implemented by objects that can manage a number of
 * {@link ApplicationListener} objects and publish events to them.
 *
 * <p>An {@link org.springframework.context.ApplicationEventPublisher}, typically
 * a Spring {@link org.springframework.context.ApplicationContext}, can use an
 * {@code ApplicationEventMulticaster} as a delegate for actually publishing events.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Stephane Nicoll
 * @see ApplicationListener
 */
public interface ApplicationEventMulticaster {
    /**
	 * Add a listener to be notified of all events.添加监听器
	 * @param listener the listener to add
	 */
	void addApplicationListener(ApplicationListener<?> listener);

	/**
	 * Add a listener bean to be notified of all events.
	 * @param listenerBeanName the name of the listener bean to add
	 */
	void addApplicationListenerBean(String listenerBeanName);

	/**
	 * Remove a listener from the notification list.删除监听器
	 * @param listener the listener to remove
	 */
	void removeApplicationListener(ApplicationListener<?> listener);

	/**
	 * Remove a listener bean from the notification list.
	 * @param listenerBeanName the name of the listener bean to remove
	 */
	void removeApplicationListenerBean(String listenerBeanName);

	/**
	 * Remove all listeners registered with this multicaster.移除所有监听器
	 * <p>After a remove call, the multicaster will perform no action
	 * on event notification until new listeners are registered.
	 */
	void removeAllListeners();

	/**
	 * Multicast the given application event to appropriate listeners.广播事件
	 * <p>Consider using {@link #multicastEvent(ApplicationEvent, ResolvableType)}
	 * if possible as it provides better support for generics-based events.
	 * @param event the event to multicast
	 */
	void multicastEvent(ApplicationEvent event);

	/**
	 * Multicast the given application event to appropriate listeners.
	 * <p>If the {@code eventType} is {@code null}, a default type is built
	 * based on the {@code event} instance.
	 * @param event the event to multicast
	 * @param eventType the type of event (can be {@code null})
	 * @since 4.2
	 */
	void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);

}
{% endcodeblock %}  
&ensp;&ensp;&ensp;&ensp;系统<font color=#008000 >事件</font>在SpringBoot中的类图如下图所示。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/SpringApplicationEvent%E7%B1%BB%E5%9B%BE.png" width="100%" height="100%">  
</div>  
&ensp;&ensp;&ensp;&ensp;最顶层是 EventObject，它代表的是一个事件对象，接着 ApplicationEvent 继承它，代表这是一个应用事件，之后SpringApplicationEvent代表了这是 Spring 中的系统事件，ApplicationStartedEvent、ApplicationFailedEvent等都是 SpringApplicationEvent 的子类。  
&ensp;&ensp;&ensp;&ensp;上图提到了这么多的事件，那在 SpringBoot 中这些事件的发送顺序是怎样的呢？  
&ensp;&ensp;&ensp;&ensp;下面是SpringBoot启动过程中涉及的事件触发流程图： 
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/%E4%BA%8B%E4%BB%B6%E8%A7%A6%E5%8F%91%E9%A1%BA%E5%BA%8F.png" width="100%" height="100%">  
</div>   
&ensp;&ensp;&ensp;&ensp;根据上述介绍的 SpringBoot 的事件相关的接口，我们可以自己定义一些监听器，然后注册到 SpringBoot 容器中。SpringBoot本身也有一些监听器的实现，上面我们已经提到，那么这些监听器是如何注册到 SpringBoot 容器中的呢？  
&ensp;&ensp;&ensp;&ensp;监听器注册的简明释义如下图所示，也是很容易理解的。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/%E7%9B%91%E5%90%AC%E5%99%A8%E6%B3%A8%E5%86%8C%E6%B5%81%E7%A8%8B.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;具体代码是如何实现的呢？
{% codeblock SpringApplication.java lang:java %}
/**
 * Create a new {@link SpringApplication} instance. The application context will load
 * beans from the specified primary sources (see {@link SpringApplication class-level}
 * documentation for details. The instance can be customized before calling
 * {@link #run(String...)}.
 * @param resourceLoader the resource loader to use
 * @param primarySources the primary bean sources
 * @see #run(Class, String[])
 * @see #setSources(Set)
 */
@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
	this.resourceLoader = resourceLoader;
	Assert.notNull(primarySources, "PrimarySources must not be null");
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
	// 根据 classpath 判断 web 应用类型
	this.webApplicationType = WebApplicationType.deduceFromClasspath();
	// 初始化 initializers 属性
	setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
	// 初始化 listeners 属性
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	// 获得是调用了哪个 main 方法
	this.mainApplicationClass = deduceMainApplicationClass();
}
{% endcodeblock %}  
&ensp;&ensp;&ensp;&ensp;可以看到，在 SpringApplication 的构造方法中，调用 getSpringFactoriesInstances 方法获取 ApplicationListener 的实现，然后使用 setListener 方法设置监听器到 SpringBoot 容器中。  
{% codeblock Game.java lang:java %}
// 获得指定类对应的对象们
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
	return getSpringFactoriesInstances(type, new Class<?>[] {});
}
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,Object... args) {
	ClassLoader classLoader = getClassLoader();
	// Use names and ensure unique to protect against duplicates
	// 加载指定类型对应的，在 `META-INFO/spring.factories` 里的类名的数组
	Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
	// 创建对象们
	List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
	// 排序对象们
	AnnotationAwareOrderComparator.sort(instances);
	return instances;
}
{% endcodeblock %}  
&ensp;&ensp;&ensp;&ensp;先通过 spring.factory 获得实现的类名，然后依次实例化，之后进行排序，返回结果。我们来看一下 spring.factory 中监听器的描述。 
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/springfactories-listener.png" width="100%" height="100%">  
</div>
&ensp;&ensp;&ensp;&ensp;在 getSpringFactoriesInstances 方法中打断点，可以清楚地看到，通过 spring-factories 加载这些监听器的实现的类名
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/springfactories%E7%9B%91%E5%90%AC%E5%99%A8debug.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;监听器模式的4个要素，上面我们已经看了3个，还差一个<font color=#008000 >事件触发机制</font>，我们来看一下源码吧。  
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/run-listener%E6%97%B6%E6%9C%BA.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;上图圈中的部分，通过 SpringApplicationRunListener 数组 listeners 直接或进入方法触发事件。下面我们来具体看一下第一个 starting 事件。  
&ensp;&ensp;&ensp;&ensp;进入 starting 方法内部，可以看到它是遍历调用 SpringApplicationRunListener 的 starting方法。
{% codeblock SpringApplicationRunListener.java lang:java %}
void starting() {
	for (SpringApplicationRunListener listener : this.listeners) {
		listener.starting();
	}
}
{% endcodeblock %}  
这个 SpringApplicationRunListener 中定义了各个阶段的事件，比如 starting、environmentPrepared、contextPrepared等等。
{% codeblock SpringApplicationRunListener.java lang:java %}
/**
 * Listener for the {@link SpringApplication} {@code run} method.
 * {@link SpringApplicationRunListener}s are loaded via the {@link SpringFactoriesLoader}
 * and should declare a public constructor that accepts a {@link SpringApplication}
 * instance and a {@code String[]} of arguments. A new
 * {@link SpringApplicationRunListener} instance will be created for each run.
 *
 * @author Phillip Webb
 * @author Dave Syer
 * @author Andy Wilkinson
 * @since 1.0.0
 */
public interface SpringApplicationRunListener {

	/**
	 * Called immediately when the run method has first started. Can be used for very
	 * early initialization.
	 */
	default void starting() {
	}

	/**
	 * Called once the environment has been prepared, but before the
	 * {@link ApplicationContext} has been created.
	 * @param environment the environment
	 */
	default void environmentPrepared(ConfigurableEnvironment environment) {
	}

	/**
	 * Called once the {@link ApplicationContext} has been created and prepared, but
	 * before sources have been loaded.
	 * @param context the application context
	 */
	default void contextPrepared(ConfigurableApplicationContext context) {
	}

	/**
	 * Called once the application context has been loaded but before it has been
	 * refreshed.
	 * @param context the application context
	 */
	default void contextLoaded(ConfigurableApplicationContext context) {
	}

	/**
	 * The context has been refreshed and the application has started but
	 * {@link CommandLineRunner CommandLineRunners} and {@link ApplicationRunner
	 * ApplicationRunners} have not been called.
	 * @param context the application context.
	 * @since 2.0.0
	 */
	default void started(ConfigurableApplicationContext context) {
	}

	/**
	 * Called immediately before the run method finishes, when the application context has
	 * been refreshed and all {@link CommandLineRunner CommandLineRunners} and
	 * {@link ApplicationRunner ApplicationRunners} have been called.
	 * @param context the application context.
	 * @since 2.0.0
	 */
	default void running(ConfigurableApplicationContext context) {
	}

	/**
	 * Called when a failure occurs when running the application.
	 * @param context the application context or {@code null} if a failure occurred before
	 * the context was created
	 * @param exception the failure
	 * @since 2.0.0
	 */
	default void failed(ConfigurableApplicationContext context, Throwable exception) {
	}

}
{% endcodeblock %} 
&ensp;&ensp;&ensp;&ensp;因为这个类定义了 SpringBoot 启动过程中各个阶段的事件，所以只用调用这个类的不同方法就可以在相应的节点触发对应的事件。
在 starting 方法内部，其实也很简单，就是调用了广播器的 multicastEvent 方法发送一个相应的 ApplicationStartingEvent 事件。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/starting%E8%B0%83%E7%94%A8%E5%B9%BF%E6%92%AD%E5%99%A8.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;SpringBoot容器通过这种机制，使监听器的内部实现与外部调用隔离开来。SpringBoot 容器在运行阶段，只需要调用这个类的各个关键方法就可以了，不需要 SpringBoot 容器自己去构造相应的事件来发送。
&ensp;&ensp;&ensp;&ensp;我们进入广播器的 multicastEvent 方法内部。
{% codeblock SimpleApplicationEventMulticaster.java lang:java %}
	@Override
	public void multicastEvent(ApplicationEvent event) {
		multicastEvent(event, resolveDefaultEventType(event));
	}

	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		//对 eventType 做了一层包装
        ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		//获取线程池
        Executor executor = getTaskExecutor();
        //获取对当前事件感兴趣的监听器列表，然后遍历
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}

	private ResolvableType resolveDefaultEventType(ApplicationEvent event) {
		return ResolvableType.forInstance(event);
	}

{% endcodeblock %}  
&ensp;&ensp;&ensp;&ensp;上述方法中 getApplicationListeners 方法是获取对当前事件感兴趣的监听器列表。我们看一下源码，是如何实现的。
{% codeblock AbstractApplicationEventMulticaster.java lang:java %}
	/**
	 * Return a Collection of ApplicationListeners matching the given
	 * event type. Non-matching listeners get excluded early.
	 * @param event the event to be propagated. Allows for excluding
	 * non-matching listeners early, based on cached matching information.
	 * @param eventType the event type
	 * @return a Collection of ApplicationListeners
	 * @see org.springframework.context.ApplicationListener
	 */
	protected Collection<ApplicationListener<?>> getApplicationListeners(
			ApplicationEvent event, ResolvableType eventType) {

        //首先先获取事件的源 source，也就是 SpringApplication
		Object source = event.getSource();
        //获得source的class type
		Class<?> sourceType = (source != null ? source.getClass() : null);
        //通过sourceType和eventType构造一个缓存key。
        //目的是若当前已经获得过对当前事件感兴趣的监听器列表，则从缓存中读取，不必再重新进行计算哪些监听器对该事件感兴趣，提升了效率
		ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

		// Quick check for existing entry on ConcurrentHashMap...
		ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
        //第一次调用的话retriever为null
		if (retriever != null) {
			return retriever.getApplicationListeners();
		}

        //beanClassLoader为null，进入if条件内
		if (this.beanClassLoader == null ||
				(ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
						(sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
			// Fully synchronized building and caching of a ListenerRetriever
                //同步块
			synchronized (this.retrievalMutex) {
                    //进入同步块，先从缓存中获取retriever
			retriever = this.retrieverCache.get(cacheKey);
                //缓存中retriever不为null，直接返回retriever的获取监听器方法
			if (retriever != null) {
				return retriever.getApplicationListeners();
			}
                //缓存retriever为null，创建ListenerRetriever实例
			retriever = new ListenerRetriever(true);
                //调用retrieveApplicationListeners方法，检索监听器
			Collection<ApplicationListener<?>> listeners =
					retrieveApplicationListeners(eventType, sourceType, retriever);
                //将检索到的retriever放进缓存中        
			this.retrieverCache.put(cacheKey, retriever);
                //返回监听器列表
			return listeners;
			}
		}
		else {
			// No ListenerRetriever caching -> no synchronization necessary
			return retrieveApplicationListeners(eventType, sourceType, null);
		}
	}

{% endcodeblock %}  
&ensp;&ensp;&ensp;&ensp;retrieveApplicationListeners 方法是如何检索监听器呢？我们继续来看。
{% codeblock AbstractApplicationEventMulticaster.java lang:java %}
	/**
	 * Actually retrieve the application listeners for the given event and source type.
	 * @param eventType the event type
	 * @param sourceType the event source type
	 * @param retriever the ListenerRetriever, if supposed to populate one (for caching purposes)
	 * @return the pre-filtered list of application listeners for the given event and source type
	 */
	private Collection<ApplicationListener<?>> retrieveApplicationListeners(
			ResolvableType eventType, @Nullable Class<?> sourceType, @Nullable ListenerRetriever retriever) {

		List<ApplicationListener<?>> allListeners = new ArrayList<>();
		Set<ApplicationListener<?>> listeners;
		Set<String> listenerBeans;
		synchronized (this.retrievalMutex) {
            //获得默认的applicationListeners和applicationLIstenerBeans
            //applicationListeners就是上文提到spring.factories加载进来的listener的实现
			listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
			listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
		}

		// Add programmatically registered listeners, including ones coming
		// from ApplicationListenerDetector (singleton beans and inner beans).
        //遍历监听器
		for (ApplicationListener<?> listener : listeners) {
            //依次判断，当前监听器是否对该事件感兴趣
			if (supportsEvent(listener, eventType, sourceType)) {
                //若感兴趣，会加入到集合中
				if (retriever != null) {
					retriever.applicationListeners.add(listener);
				}
				allListeners.add(listener);
			}
		}

		// Add listeners by bean name, potentially overlapping with programmatically
		// registered listeners above - but here potentially with additional metadata.
		if (!listenerBeans.isEmpty()) {
			ConfigurableBeanFactory beanFactory = getBeanFactory();
			for (String listenerBeanName : listenerBeans) {
				try {
					if (supportsEvent(beanFactory, listenerBeanName, eventType)) {
						ApplicationListener<?> listener =
								beanFactory.getBean(listenerBeanName, ApplicationListener.class);
						if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
							if (retriever != null) {
								if (beanFactory.isSingleton(listenerBeanName)) {
									retriever.applicationListeners.add(listener);
								}
								else {
									retriever.applicationListenerBeans.add(listenerBeanName);
								}
							}
							allListeners.add(listener);
						}
					}
					else {
						// Remove non-matching listeners that originally came from
						// ApplicationListenerDetector, possibly ruled out by additional
						// BeanDefinition metadata (e.g. factory method generics) above.
						Object listener = beanFactory.getSingleton(listenerBeanName);
						if (retriever != null) {
							retriever.applicationListeners.remove(listener);
						}
						allListeners.remove(listener);
					}
				}
				catch (NoSuchBeanDefinitionException ex) {
					// Singleton listener instance (without backing bean definition) disappeared -
					// probably in the middle of the destruction phase
				}
			}
		}

        //对监听器通过order值进行排序
		AnnotationAwareOrderComparator.sort(allListeners);
		if (retriever != null && retriever.applicationListenerBeans.isEmpty()) {
			retriever.applicationListeners.clear();
			retriever.applicationListeners.addAll(allListeners);
		}
        //将集合返回
		return allListeners;
	}

{% endcodeblock %}  
&ensp;&ensp;&ensp;&ensp;上面方法中 supportsEvent 方法是如何判断是否感兴趣呢？我们来打断点看一下。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/supportEvent-cloudFoundry.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;首先进入的是 CloudFoundryVcapEnvironmentPostProcessor，该类不是 GenericApplicationListener 的子类，所以会为该类创建一个GenericApplicationListenerAdapter，作为smartListener。断点进入GenericApplicationListenerAdapter中，可以看到，delegate就是CloudFoundryVcapEnvironmentPostProcessor，接着会调用resolveDeclaredEventType方法，来计算当前代理类对哪个事件感兴趣。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/listenerAdapter-cloudFoundry.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;这个resolveDeclaredEventType方法，是spring内部实现的一个泛型解析器，会根据类定义获得该类声明的事件类型，由于CloudFoundryVcapEnvironmentPostProcessor类实现ApplicationListener，泛型是ApplicationPreparedEvent。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/cloudFoundry%E6%B3%9B%E5%9E%8B.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;所以该方法就会获得 ApplicationPreparedEvent。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/resolveType-cloudFoundry.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;接着会调用 smartListener 的 supportsEventType 方法，判断是否支持该事件。进入方法，首先判断该类是否是 SmartApplicationListener 的子类，通过上面 CloudFoundryVcapEnvironmentPostProcessor 的定义，它不是 SmartApplicationListener 的子类，所以会进入 else 中。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/supportEventType-cloudFoundry.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;上面得到 declaredEventType 是 ApplicationPreparedEvent ，所以不为null。当前 eventType 是 ApplicationStartingEvent，显然“||”后半部分结果也是false，所以 smartListener.supportsEventType 是false，因为supportsEvent方法后面条件是“&&”，所以后面的内容不必在意，可以直接确定该监听器不支持该事件。  
&ensp;&ensp;&ensp;&ensp;我们接着调试，接下来进入的是 ConfigFileApplicationListener，按照上述同样的方法进行计算，由于它是 SmartApplicationListener 的子类。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/configFile%E5%AE%9A%E4%B9%89.png" width="100%" height="100%">  
</div> 
&ensp;&ensp;&ensp;&ensp;所以会进入 if 条件内
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/configFile%E8%BF%9B%E5%85%A5if.png" width="100%" height="100%">  
</div>
&ensp;&ensp;&ensp;&ensp;这里会调用实现类的 supportEvent 方法，即 ConfigFileApplicationListener 类的 supportsEventType 方法。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/configFile%E5%86%85%E9%83%A8supportEventType.png" width="100%" height="100%">  
</div>
&ensp;&ensp;&ensp;&ensp;判断当前这个event是否是 ApplicationEnvironmentPreparedEvent，或者是否是 ApplicationPreparedEvent，都不是的话，会返回false，也就是说对该事件不感兴趣，所以不会将 ConfigFileApplicationListener 添加到上述的感兴趣监听器列表中。
&ensp;&ensp;&ensp;&ensp;通过以上的方法，就检索出了哪些监听器对当前事件感兴趣。
&ensp;&ensp;&ensp;&ensp;之后在遍历这些监听器的过程中，会调用 invokeListener 方法
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/multicastEvent-else.png" width="100%" height="100%">  
</div>
&ensp;&ensp;&ensp;&ensp;invokeListener方法中会调用doInvokeListener方法，最终我们可以看到listener.onApplicationEvent，这里会进入具体事件的触发
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/doInvokeListener%E6%96%AD%E7%82%B9.png" width="100%" height="100%">  
</div>
&ensp;&ensp;&ensp;&ensp;以 ConfigFileApplicationListener 为例，会根据不同的event触发不同的事件。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/configFile-onApplicationEvent.png" width="100%" height="100%">  
</div>
&ensp;&ensp;&ensp;&ensp;通过上面的分析，总结一下获取监听器列表逻辑，如下图所示。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/%E8%8E%B7%E5%8F%96%E7%9B%91%E5%90%AC%E5%99%A8%E5%88%97%E8%A1%A8%E6%B5%81%E7%A8%8B.png" width="40%" height="60%">  
</div>
&ensp;&ensp;&ensp;&ensp;其中supportsEvent流程如下图所示。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/%E8%A7%A6%E5%8F%91%E6%9D%A1%E4%BB%B6%E6%B5%81%E7%A8%8B.png" width="100%" height="100%">  
</div>  

## 三、总结
&ensp;&ensp;&ensp;&ensp;以上关于SpringBoot中监听器的主要核心方法的实现我们已经一点点看过了，看上去很复杂，但是实际上还是按照标准的监听器模式实现的。其中将一系列事件的触发方法封装在一个runListener中，降低了系统的耦合度，使得调用的时候也变得很轻松，这一点很值得我们学习。