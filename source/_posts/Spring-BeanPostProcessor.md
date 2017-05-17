---
title: Spring基础 - BeanPostProcessor
date: 2017-05-15 20:40:40
tags: Spring
---

# 接口简介

**BeanPostProcessor**是Spring提供的一个钩子(*Hook*)接口，可用于实现对新的bean实例进行自定义修改，比如说，检测bean是否实现了某些特定接口从而可以对其进行进一步操作，或者可以将原实例进行包装而实际返回代理对象。其内部主要声明了*postProcessBeforeInitialization*和*postProcessAfterInitialization*两个回调方法，具体的接口声明如下所示：

```java
public interface BeanPostProcessor {
  	//在bean进行初始化行为之前被调用
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
  	//在bean进行初始化行为之后被调用
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

需要注意的是这里所说的bean初始化行为主要是指调用**InitializingBean**的*afterPropertiesSet*或者一个自定义的*init-method*。

<!--more-->

# 触发时机

根据Spring提供的文档注释，不难获知**BeanPostProcessor**提供的两个方法会分别在bean进行初始化的前后被触发调用。抱着好奇心，也是为了更深入的理解，我通过IDE查看这两个方法的调用链，然后发现原来抽象类**AbstractAutowireCapableBeanFactory**的*initializeBean*方法会间接调用这两者：

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged(new PrivilegedAction<Object>() {
			@Override
			public Object run() {
				invokeAwareMethods(beanName, bean);
				return null;
			}
		}, getAccessControlContext());
	}
	else {
		invokeAwareMethods(beanName, bean);
	}
	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}
	try {
		invokeInitMethods(beanName, wrappedBean, mbd);
	} catch (Throwable ex) {
		throw new BeanCreationException(
          (mbd != null ? mbd.getResourceDescription() : null),
          	beanName, "Invocation of init method failed", ex);
	}
	if (mbd == null || !mbd.isSynthetic()) {
    	wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```

方法*applyBeanPostProcessorsBeforeInitialization*和*applyBeanPostProcessorsAfterInitialization*最终会分别调用已经注册的**BeanPostProcessor**的这两个方法，而且是在方法*invokeInitMethods*调用的前后。而方法*invokeInitMethods*实质就是对bean进行判定，来确定是否可以转换成**InitializingBean**进而调用接口方法*afterPropertiesSet*或者是否存在自定义的*init-method*进行调用。 

# 注册方式

通过**ApplicationContext**和**BeanFactory**进行注册的方式稍微有些不同。对于**ApplicationContext**来说，只要定义在其配置文件里(或注解方式)，它就会自动检测到**BeanPostProcessor**实现类 ，并将其注册；对于普通的**BeanFactory**来说，那么就需要显式以编程方式来调用方法*addBeanPostProcessor*进行注册。

假设项目开启了注解扫描，那么只需加上注解，就能实现**ApplicationContext**方式的自动注册：

```java
@Configuration
public class MyBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        System.out.println("Enter MyBeanPostProcessor postProcessBeforeInitialization");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        System.out.println("Enter MyBeanPostProcessor postProcessAfterInitialization");
        return bean;
    }

}
```

# 调用顺序

在实际项目中，我们可能需要注册不止一个**BeanPostProcessor**的实现类，而不管是采用**BeanFactory**还是**ApplicationContext**，实际调用顺序都是取决于注册顺序。由于**BeanFactory**是显式编程方式调用，因此可以人为决定注册顺序；而采用**ApplicationContext**的自动检测方式，其注册顺序由以下的规则确定：

1. 首先调用实现了**PriorityOrdered**接口的**BeanPostProcessor**实现类，如果不止一个类实现了**PriorityOrdered**接口，那么会按照方法*getOrder*返回的值的大小，越大就会优先调用。
2. 接着调用实现了**Ordered**接口的**BeanPostProcessor**实现类，如果不止一个类实现了**Ordered**接口，那么会按照*getOrder*返回的值的大小，值越大就会优先调用。
3. 然后调用既没有实现**PriorityOrdered**也没有实现**Ordered**的**BeanPostProcessor**实现类，这个调用顺序往往是无序的，因为由于没有在注册之前进行排序操作。
4. 最后是调用实现**MergedBeanDefinitionPostProcessor**(通常供框架内部使用)的实现类。

感兴趣的话，可以看一下**PostProcessorRegistrationDelegate**的*registerBeanPostProcessors*方法，其内部实现了对**BeanPostProcessor**的注册(这也是**AbstractApplicationContext**的*refresh*过程之一)：

```java
	public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(beanFactory, orderedPostProcessors);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		sortPostProcessors(beanFactory, internalPostProcessors);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```

# 简单实践

现在考虑项目中有这样一个配置，它通过读取配置文件**app.properties**，可以设置规则属性：

```java
@Configuration
@PropertySource("classpath:app.properties")
public class MyConfig {

    @Value("${app.rule}")
    private String rule;

    public String getRule() {
        return rule;
    }
    
}
```

现在我们需要对设置的规则进行校验，可以通过实现自定义的**BeanPostProcessor**来解决：

```java
@Configuration
public class RuleNameBeanPostProcessor implements BeanPostProcessor {

    private static final List<String> rules = Arrays.asList("ruleA", "ruleB", "ruleC");

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        if (bean instanceof MyConfig) {
            MyConfig config = (MyConfig) bean;
            if (!rules.contains(config.getRule())) {
                throw new BeanInitializationException("Invalid rule");
            }
        }
        return bean;
    }

}
```

如果我们在**app.properties**文件中配置了预定义规则之外的规则，那么程序在启动的时候就会抛出异常

![异常信息](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/BeanInitializationException.png)

通过启动阶段的检测机制，我们能够快速发现可能存在的问题，从而避免在程序运行期间发现错误。当然**BeanPostProcessor**在实际项目中肯定可以发挥更多的作用，这有待于我们进一步去挖掘。另外，如果对于Spring源码感兴趣的话，这个接口以及框架内部已有的实现类也是我们必须要了解的。

# 参考

[BeanPostProcessor javadoc](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/beans/factory/config/BeanPostProcessor.html)