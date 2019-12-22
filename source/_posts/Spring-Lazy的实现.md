---
title: Spring @Lazy的实现
date: 2019-12-22 15:26:12
tags:
- Java
- 源码
---

# @Lazy的用途
```
/**
 * Indicates whether a bean is to be lazily initialized.
 *
 * <p>May be used on any class directly or indirectly annotated with {@link
 * org.springframework.stereotype.Component @Component} or on methods annotated with
 * {@link Bean @Bean}.
 *
 * <p>If this annotation is not present on a {@code @Component} or {@code @Bean} definition,
 * eager initialization will occur. If present and set to {@code true}, the {@code @Bean} or
 * {@code @Component} will not be initialized until referenced by another bean or explicitly
 * retrieved from the enclosing {@link org.springframework.beans.factory.BeanFactory
 * BeanFactory}. If present and set to {@code false}, the bean will be instantiated on
 * startup by bean factories that perform eager initialization of singletons.
 *
 * <p>If Lazy is present on a {@link Configuration @Configuration} class, this
 * indicates that all {@code @Bean} methods within that {@code @Configuration}
 * should be lazily initialized. If {@code @Lazy} is present and false on a {@code @Bean}
 * method within a {@code @Lazy}-annotated {@code @Configuration} class, this indicates
 * overriding the 'default lazy' behavior and that the bean should be eagerly initialized.
 *
 * <p>In addition to its role for component initialization, this annotation may also be placed
 * on injection points marked with {@link org.springframework.beans.factory.annotation.Autowired}
 * or {@link javax.inject.Inject}: In that context, it leads to the creation of a
 * lazy-resolution proxy for all affected dependencies, as an alternative to using
 * {@link org.springframework.beans.factory.ObjectFactory} or {@link javax.inject.Provider}.
```

官方注释写的也挺清楚，简单概括一下：
1. 写在`@Component`或者`@Bean`之前用于声明bean是否延迟初始化
2. 写在`@Configuration`之前用于声明这个`Configuration`内的所有`Bean`是否延迟初始化
3. 写在构造注入或者`@Autowired`之前用于声明是否延迟注入 此时注入的实际是一个`lazy-resolution proxy`

# 分别学习@Lazy在这几个场景下如何实现

## 声明Bean延迟初始化 
这个场景是比较容易理解，也比较容易想到实现的 

org.springframework.context.annotation.ClassPathBeanDefinitionScanner#doScan 部分代码
```java
String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
if (candidate instanceof AbstractBeanDefinition) {
    postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
}
if (candidate instanceof AnnotatedBeanDefinition) {
    // 这里做了注解的识别和处理
    AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
}
```

org.springframework.context.annotation.AnnotationConfigUtils#processCommonDefinitionAnnotations 部分代码
```java
// 本质上还是setLazyInit属性
AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
if (lazy != null) {
    abd.setLazyInit(lazy.getBoolean("value"));
}
else if (abd.getMetadata() != metadata) {
    lazy = attributesFor(abd.getMetadata(), Lazy.class);
    if (lazy != null) {
        abd.setLazyInit(lazy.getBoolean("value"));
    }
}
```

## 申明延迟注入
demo代码：（这2个类都放在`com.example.demo`包下 
```java
@Component
public class TestComponent {
    @Lazy
    @Autowired
    private AnotherComponent anotherComponent;

    public void test() {
        System.out.println(anotherComponent.hello());
    }

    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext("com.example.demo");
        TestComponent testComponent = (TestComponent) ac.getBean("testComponent");
        testComponent.test();
    }
}

@Component
public class AnotherComponent {
    public String hello() {
        return "hi, I am another.";
    }
}
```
上面也提到了，本质上inject的实际是一个`lazy-resolution proxy`，并不是原来的bean本身。分2部分看下源码:
* 扫描并register`testComponent`时对lazy @Autowired的处理
1. 在beanFactory初始化之后会预实例化 
singleton:org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization
```java
// Allow for caching all bean definition metadata, not expecting further changes.
beanFactory.freezeConfiguration();

// Instantiate all remaining (non-lazy-init) singletons.
beanFactory.preInstantiateSingletons();
```
2. 实例化testComponent时 会注入它的依赖 
org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveDependency
```java
descriptor.initParameterNameDiscovery(this.getParameterNameDiscoverer());
if (Optional.class == descriptor.getDependencyType()) {
    return this.createOptionalDependency(descriptor, requestingBeanName);
} else if (ObjectFactory.class != descriptor.getDependencyType() && ObjectProvider.class != descriptor.getDependencyType()) {
    if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        return (new DefaultListableBeanFactory.Jsr330Factory()).createDependencyProvider(descriptor, requestingBeanName);
    } else {
        // 这里判断和注入LazyResolutionProxy
        Object result = this.getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(descriptor, requestingBeanName);
        if (result == null) {
            result = this.doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
        }
        return result;
    }
} else {
    return new DefaultListableBeanFactory.DependencyObjectProvider(descriptor, requestingBeanName);
}
```
3. 判断和注入LazyResolutionProxy
```java
// 是否是lazy依赖
protected boolean isLazy(DependencyDescriptor descriptor) {
    for (Annotation ann : descriptor.getAnnotations()) {
        Lazy lazy = AnnotationUtils.getAnnotation(ann, Lazy.class);
        if (lazy != null && lazy.value()) {
            return true;
        }
    }
    MethodParameter methodParam = descriptor.getMethodParameter();
    if (methodParam != null) {
        Method method = methodParam.getMethod();
        if (method == null || void.class == method.getReturnType()) {
            Lazy lazy = AnnotationUtils.getAnnotation(methodParam.getAnnotatedElement(), Lazy.class);
            if (lazy != null && lazy.value()) {
                return true;
            }
        }
    }
    return false;
}

// 创建proxy
protected Object buildLazyResolutionProxy(final DependencyDescriptor descriptor, final @Nullable String beanName) {
    Assert.state(getBeanFactory() instanceof DefaultListableBeanFactory,
            "BeanFactory needs to be a DefaultListableBeanFactory");
    final DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) getBeanFactory();
    // TargetSource是一个接口 用于提供实际要代理的target 
    // 这里给到的实现是从beanFactory取得target
    TargetSource ts = new TargetSource() {
        @Override
        public Class<?> getTargetClass() {
            return descriptor.getDependencyType();
        }
        @Override
        public boolean isStatic() {
            return false;
        }
        @Override
        public Object getTarget() {
            // 这里从beanFactory中获取真正的bean
            Object target = beanFactory.doResolveDependency(descriptor, beanName, null, null);
            if (target == null) {
                Class<?> type = getTargetClass();
                if (Map.class == type) {
                    return Collections.emptyMap();
                }
                else if (List.class == type) {
                    return Collections.emptyList();
                }
                else if (Set.class == type || Collection.class == type) {
                    return Collections.emptySet();
                }
                throw new NoSuchBeanDefinitionException(descriptor.getResolvableType(),
                        "Optional dependency not present for lazy injection point");
            }
            return target;
        }
        @Override
        public void releaseTarget(Object target) {
        }
    };
    ProxyFactory pf = new ProxyFactory();
    pf.setTargetSource(ts);
    Class<?> dependencyType = descriptor.getDependencyType();
    if (dependencyType.isInterface()) {
        pf.addInterface(dependencyType);
    }
    // 这里生成proxy
    return pf.getProxy(beanFactory.getBeanClassLoader());
}
```
4. 生成proxy 分为jdk(JdkDynamicAopProxy)和cglib(CglibAopProxy) 
org.springframework.aop.framework.DefaultAopProxyFactory#createAopProxy
```java
if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
    return new JdkDynamicAopProxy(config);
} else {
    Class<?> targetClass = config.getTargetClass();
    if (targetClass == null) {
        throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
    } else {
        return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));
    }
}
```
> 复习下：spring的动态代理在没给具体类或者面向接口时使用jdk，其他使用cglib (因为cglib的本质是动态生成被代理类的子类)  
5. proxy中对方法的拦截 这里jdk和cglib行为相似 
关键在于这里调用了`targetSource.getTarget()`来从beanFactory获取bean从而触发了init 核心代码:

org.springframework.aop.framework.JdkDynamicAopProxy#invoke
```java
target = targetSource.getTarget();
Class<?> targetClass = target != null ? target.getClass() : null;
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
if (chain.isEmpty()) {
    Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
    retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
} else {
    MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
    retVal = invocation.proceed();
}
```

org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor#intercept
```java
target = targetSource.getTarget();
Class<?> targetClass = target != null ? target.getClass() : null;
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
Object retVal;
if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
    Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
    retVal = methodProxy.invoke(target, argsToUse);
} else {
    retVal = (new CglibAopProxy.CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy)).proceed();
}
```

# 小结
@Lazy可用于bean的延迟初始化(asm获取类注解)和bean依赖的延迟注入(动态代理)