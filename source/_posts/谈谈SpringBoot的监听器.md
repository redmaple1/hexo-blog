---
title: 谈谈SpringBoot的监听器
date: 2020-01-17 14:55:00
tags: 
    - SpringBoot
    - 源码
categories: SpringBoot
---
&ensp;&ensp;&ensp;最近在看SpringBoot的源码，在SpringBoot项目启动的过程中，监听器在不同阶段都会监听相应的事件，今天我们就来谈谈SpringBoot启动过程中的监听器。
## 一、监听器扫盲  
&ensp;&ensp;&ensp;监听器模式，顾名思义就是某个对象监听某个或某些事件的触发，然后做出相应的操作。这句话可以看出，监听器模式包含几个特定的元素：  
- 事件，即监听什么（Event）
- 监听者，即谁来监听（Listener）
- 广播器，即谁来发布事件（Multicaster）
- 事件触发机制，即事件什么时候发布   

整体的原理如下所示。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/%E7%9B%91%E5%90%AC%E5%99%A8%E6%A8%A1%E5%BC%8F.png" width="100%" height="100%">  
</div>
&ensp;&ensp;&ensp;当系统运行在某些关键节点的时候，会通过广播器去发布一些事件，而系统中存在着一些监听器，对某些事件感兴趣，去订阅这些事件。当这些事件被发布出去之后，监听器监听到这些事件，会触发一些行为。  
&ensp;&ensp;&ensp;这就是监听器的简单解释，那么在SpringBoot中监听器是如何实现的呢？接下来我们就来看看吧！

## 二、揭开面纱
&ensp;&ensp;&ensp;在SpringBoot中，系统监听器是 ApplicationListener，可以看到源码的注释，通过实现这个接口来实现监听器。   
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
上面的 ApplicationListener 接口还有一个泛型，继承自 ApplicationEvent，这就是说在实现这个接口的时候，可以声明自己感兴趣的事件。系统在触发这个系统监听器的时候会根据其感兴趣的事件做一个过滤。这个接口定义了一个 onApplicationEvent 方法，是当它监听到事件发生的时候，会去做什么事情。  
接下来我们看一下监听器模式的【广播器】在SpringBoot中的实现。  
系统广播器是 ApplicationEventMulticaster ，实现这个接口来管理一些应用监听器，并且广播事件。其中定义了添加、删除监听器的方法以及广播事件的方法。   
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
系统事件在SpringBoot中的类图如下图所示。
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/SpringApplicationEvent%E7%B1%BB%E5%9B%BE.png" width="100%" height="100%">  
</div>  
最顶层是 EventObject，它代表的是一个事件对象，接着 ApplicationEvent 继承它，代表这是一个应用事件，之后SpringApplicationEvent代表了这是 Spring 中的系统事件，ApplicationStartedEvent、ApplicationFailedEvent等都是 SpringApplicationEvent 的子类。  
上图提到了这么多的事件，那在 SpringBoot 中这些事件的发送顺序是怎样的呢？  
下面是SpringBoot启动过程中涉及的事件触发流程图： 
<img src="https://hexo-rxy.oss-cn-beijing.aliyuncs.com/spring-boot/listener/%E4%BA%8B%E4%BB%B6%E8%A7%A6%E5%8F%91%E9%A1%BA%E5%BA%8F.png" width="100%" height="100%">  
</div>   

