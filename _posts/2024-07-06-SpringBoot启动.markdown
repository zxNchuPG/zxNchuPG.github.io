---
layout:      post
title:       "SpringBoot启动流程"
subtitle:    "SpringBoot启动流程"
author:      "Ekko"
header-img:  "img/bg/bg-20240706.png"
catalog:     true
tags:
  - 学习笔记
  - SpringBoot
---

> 参考资料 [SpringBoot启动流程](https://www.cnblogs.com/hefeng2014/p/17770183.html)

[TOC]

### SPI（Service provider interface） 

是Java提供的一种为服务框架提供服务实现的机制。它允许框架在运行时动态地发现服务的实现，从而实现模块化设计。在Java中，SPI机制主要用于解耦API和实现，使得应用程序可以在不修改代码的情况下替换或扩展功能

核心思想是通过配置文件声明服务提供者的实现，Java运行时环境在需要时通过这个配置文件找到并加载相应的服务实现

工作原理：

1. 定义服务接口：定义一个服务接口（或者抽象类）
2. 提供服务实现：提供该服务接口的一个或多个实现类
3. 创建服务提供者配置文件：在META-INF/services目录下创建一个以服务接口完全限定类名命名的文件，文件内容为实现该接口的具体类名
4. 加载服务实现：通过java.util.ServiceLoader来加载并使用这些实现

---

### SpringBoot启动流程分析

### 1、main方法进入

```java
@SpringBootApplication
public class Application {
    SpringApplication.run(Application.class);
}


// SpringApplication.run 内部
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return (new SpringApplication(primarySources)).run(args);
}
```

### 2、new SpringApplication()构造器

```java
/**
 * @params primarySources 主启动类
 */
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.sources = new LinkedHashSet();
    this.bannerMode = Mode.CONSOLE;
    this.logStartupInfo = true;
    this.addCommandLineProperties = true;
    this.addConversionService = true;
    this.headless = true;
    this.registerShutdownHook = true;
    this.additionalProfiles = new HashSet();
    this.isCustomEnvironment = false;
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    
    // 保存 主启动类
    this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
    
    // 判断当前应用环境，从classpath下判断当前SpringBoot应用应该使用哪种环境启动
    // servlet 同步阻塞、reactivi 异步非阻塞
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    
    // 设置初始化器(会将一组类型为ApplicationContextInitializer的初始化器放入SpringApplication中)
    this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
    
    // 设置监听器(加载类型为ApplicationListener的监听器，和上面初始化器一样)
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));

    // 确定主配置类(找到有main方法的类)
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```

#### 2.1 ApplicationContextInitializer 初始化器

```java
this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
```

SpringApplication构造器中，会加载ApplicationContextInitializer列表，ApplicationContextInitializer的作用是用于在刷新容器之前初始化Spring ConfigurableApplicationContext 的回调接口，我们也可以实现 ApplicationContextInitializer接口 来在容器刷新之前做一些我们的初始化逻辑

比如 NacosBootStrapContextInitializer 会去加载 bootstrap.properties 配置

---

this.getSpringFactoriesInstances() 内部

通过SPI机制从META-INFO下的spring.factories加载所有类型为ApplicationContextInitializer并且创建对应的实例返回

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = this.getClassLoader();
    Set<String> names = new LinkedHashSet(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = this.createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
    if (result != null) {
        return result;
    } else {
        try {
            Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
            ...
            return result;
        } catch (IOException var13) {
            IOException ex = var13;
            throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", ex);
        }
    }
}
```
---

在spring-boot中一共配置了6个ApplicationContextInitializer

1. ConfigurationWarningsApplicationContextInitializer：报告IOC容器的一些常见的错误配置
2. ContextIdApplicationContextInitializer：设置Spring应用上下文的ID
3. DelegatingApplicationContextInitializer：加载 application.properties 中 context.initializer.classes 配置的类
4. ServerPortInfoApplicationContextInitializer：将内置servlet容器实际使用的监听端口写入到 Environment 环境属性中
5. SharedMetadataReaderFactoryContextInitializer：创建一个 SpringBoot 和ConfigurationClassPostProcessor 共用的 CachingMetadataReaderFactory 对象
6. ConditionEvaluationReportLoggingListener：将 ConditionEvaluationReport 写入日志

---

#### 2.2 ApplicationListener 监听器

和 ApplicationContextInitializer 加载一致，也是通过SPI机制从META-INFO下的spring.factories

spring配置的内部监听器

1. ClearCachesApplicationListener：应用上下文加载完成后对缓存做清除工作
2. ParentContextCloserApplicationListener：监听双亲应用上下文的关闭事件并往自己的子应用上下文中传播
3. FileEncodingApplicationListener：检测系统文件编码与应用环境编码是否一致，如果系统文件编码和应用环境的编码不同则终止应用启动
4. AnsiOutputApplicationListener：根据 spring.output.ansi.enabled 参数配置 AnsiOutput
5. ConfigFileApplicationListener：从常见的那些约定的位置读取配置文件
6. DelegatingApplicationListener：监听到事件后转发给 application.properties 中配置的 context.listener.classes 的监听器
7. ClasspathLoggingApplicationListener：对环境就绪事件
8. ApplicationEnvironmentPreparedEvent 和应用失败事件 ApplicationFailedEvent 做出响应
9. LoggingApplicationListener：配置 LoggingSystem。使用 logging.config 环境变量指定的配置或者缺省配置
10. LiquibaseServiceLocatorApplicationListener：使用一个可以和 SpringBoot 可执行jar包配合工作的版本替换 LiquibaseServiceLocator
11. BackgroundPreinitializer：使用一个后台线程尽早触发一些耗时的初始化任务

---

### 3、SpringApplicaion.run方法

```java
public ConfigurableApplicationContext run(String... args) {
    // 创建 创建时间性能监控器
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    // 创建空的IOC容器
    ConfigurableApplicationContext context = null;

    // 异常报告器
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();

    // 配置与awt相关的信息
    this.configureHeadlessProperty();

    // 获取SpringApplicationRunListeners
    SpringApplicationRunListeners listeners = this.getRunListeners(args);
    
    // listeners 调用 starting方法（回调机制）
    listeners.starting();

    Throwable ex;
    Collection exceptionReporters;
    try {
        // 将 main 方法的 args 参数（比如启动参数）封装到对象中
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

        // 准备运行时环境
        ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);

        // 如果有配置 spring.beaninfo.ignore ，则将该配置设置进系统参数
        this.configureIgnoreBeanInfo(environment);

        // 打印 banner
        Banner printedBanner = this.printBanner(environment);

        // 创建 ApplicationContext （创建IOC容器）
        context = this.createApplicationContext();

        // 初始化异常报告器
        exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);

        // 初始化IOC容器
        this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);

        // 刷新容器上下文
        this.refreshContext(context);
        
        // 刷新后的处理
        this.afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
        }

        // 发布started时间
        listeners.started(context);
        
        // 运行器回调
        this.callRunners(context, applicationArguments);
    } catch (Throwable var10) {
        ex = var10;
        this.handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        listeners.running(context);
        return context;
    } catch (Throwable var9) {
        ex = var9;
        this.handleRunFailure(context, ex, exceptionReporters, (SpringApplicationRunListeners) null);
        throw new IllegalStateException(ex);
    }
}
```
---

#### 3.1 new StopWatch()

就是一个计时器，用来监控启动耗时的

#### 3.2 创建IOC容器、异常报告器

SpringBootExceptionReporter是启动错误报告的报告器，并且也是用SpringFactoriesLoader加载，实现类只有一个：FailureAnalyzers

#### 3.3 awt相关信息

这里从System中取出一个配置，然后又给设置回去了，这里大家可以不用关注，这里的作用是：设置应用在启动时，即使没有检测到显示器也允许其继续运行

#### 3.4 getRunListeners：获取SpringApplicationRunListeners

使用SPI机制加载SpringApplicationRunListener的实现类，默认加载的listener只有一个，类型为EventPublishingRunListener

#### 3.5 SpringApplicationRunListener的回调方法以及回调时机

```java
public interface SpringApplicationRunListener {
    
    // 首次启动run方法时立即调用，可用于非常早期的初始化
    void starting();

    // 准备好环境（Environment构建完成），但是在创建 ApplicationContext 之前调用
    void environmentPrepared(ConfigurableEnvironment environment);

    // 构建和构造 ApplicationContext 之后，加载之前调用
    void contextPrepared(ConfigurableApplicationContext context);

    // ApplicationContext 已加载，刷新之前调用
    void contextLoaded(ConfigurableApplicationContext context);

    // ApplicationContext 已刷新，应用程序已经启动，但是尚未调用 CommandLineRunners 和 ApplicationRunners
    void started(ConfigurableApplicationContext context);

    // 在运行方法彻底完成之前立即调用，刷新 ApplicationContext 并调用所有 CommandLineRunners 和 ApplicationRunner
    void running(ConfigurableApplicationContext context);

    // 在运行应用程序失败时调用
    void failed(ConfigurableApplicationContext context, Throwable exception);
    
}
```

#### 3.6 prepareEnvironment：准备运行时环境

这里可以把Environment理解为IOC容器的运行环境，包括Profile和Properties两大部分，他可由一个到几个激活的Profile共同配置，他的配置可在应用级bean中获取


```java
private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
    // Create and configure the environment
    // 创建运行时环境
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    
    // 配置运行时环境
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    
    // 回调监听器（Environment构建完成，但是在创建 ApplicationContext 之前调用）
    listeners.environmentPrepared(environment);
    
    // 环境与应用绑定
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader())
                .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

##### 3.6.1 getOrCreateEnvironment 创建运行时环境

默认SpringBoot环境下会创建StandardServletEnvironment

```java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    switch (this.webApplicationType) {
    case SERVLET:
        return new StandardServletEnvironment();
    case REACTIVE:
        return new StandardReactiveWebEnvironment();
    default:
        return new StandardEnvironment();
    }
}
```

##### 3.6.2 configureEnvironment 配置运行时环境

1. ConversionService是类型转换的根接口，它是用来在SpringWebMvc中用来做参数转换的
2. 配置PropertySource和Profile

```java
protected void configureEnvironment(ConfigurableEnvironment environment,
			String[] args) {
    if (this.addConversionService) {
        ConversionService conversionService = ApplicationConversionService
                .getSharedInstance();
        environment.setConversionService(
                (ConfigurableConversionService) conversionService);
    }
    configurePropertySources(environment, args);
    configureProfiles(environment, args);
}
```

##### 3.6.3 bindToSpringApplication 环境与应用绑定

作用是：把配置内容绑定到指定的属性配置类中(类似@ConfigurationProperties)

```java
protected void bindToSpringApplication(ConfigurableEnvironment environment) {
    try {
        Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));
    }
    catch (Exception ex) {
        throw new IllegalStateException("Cannot bind to SpringApplication", ex);
    }
}
```

#### 3.7 configureIgnoreBeanInfo 设置系统参数

控制是否跳过BeanInfo类的搜索(spring.beaninfo.ignore)，默认是true

```java
private void configureIgnoreBeanInfo(ConfigurableEnvironment environment) {
    if (System.getProperty(
            CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME) == null) {
        Boolean ignore = environment.getProperty("spring.beaninfo.ignore",
                Boolean.class, Boolean.TRUE);
        System.setProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME,
                ignore.toString());
    }
}
```

#### 3.8 createApplicationContext 创建IOC容器

根据不同的环境创建不同的IOC容器，但是创建的都是基于Annotation的ApplicationContext

IOC容器类型：

1. Servlet - StandardServletEnvironment - AnnotationConfigServletWebServerApplicationContext
2. Reactive - StandardReactiveWebEnvironment - AnnotationConfigReactiveWebServerApplicationContext
3. None - StandardEnvironment - AnnotationConfigApplicationContext

```java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
            case SERVLET:
                contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                break;
            case REACTIVE:
                contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                break;
            default:
                contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, "
                            + "please specify an ApplicationContextClass",
                    ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```
---

#### 3.9 prepareContext 初始化IOC容器

```java
private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
    // 将创建好的应用环境设置到IOC容器中
    context.setEnvironment(environment);
    
    // IOC容器的后置处理
    postProcessApplicationContext(context);
    
    // 执行初始化
    applyInitializers(context);
    
    // 回调监听器
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
                .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    // Load the sources
    // 加载启动类
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    
    // 注册启动类
    load(context, sources.toArray(new Object[0]));
    
    // 回调监听器
    listeners.contextLoaded(context);
}
```

---

##### 3.9.1 postProcessApplicationContext IOC容器的后置处理

```java
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
    // 注册 BeanName 生成器 
    if (this.beanNameGenerator != null) {
        context.getBeanFactory().registerSingleton(
                AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
                this.beanNameGenerator);
    }
    
    // 设置资源加载器和类加载器
    if (this.resourceLoader != null) {
        if (context instanceof GenericApplicationContext) {
            ((GenericApplicationContext) context)
                    .setResourceLoader(this.resourceLoader);
        }
        if (context instanceof DefaultResourceLoader) {
            ((DefaultResourceLoader) context)
                    .setClassLoader(this.resourceLoader.getClassLoader());
        }
    }
    
    // 设置类型转化器
    if (this.addConversionService) {
        context.getBeanFactory().setConversionService(
                ApplicationConversionService.getSharedInstance());
    }
}
```

##### 3.9.2 applyInitializers 执行初始化

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
    for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
                initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```

---

##### 3.9.3 load 注册主启动类

```java
protected void load(ApplicationContext context, Object[] sources) {
    if (logger.isDebugEnabled()) {
        logger.debug(
                "Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    }
    
    // getBeanDefinitionRegistry 获取IOC容器
    // 最终拿到的IOC容器是 AnnotationConfigServletWebServerApplicationContext
    BeanDefinitionLoader loader = createBeanDefinitionLoader(
            getBeanDefinitionRegistry(context), sources);
    
    // 设置 BeanName 生成器，通过Debug发现此时它还没有被注册
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    
    // 设置资源加载器
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    
    // 设置运行环境
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    
    // BeanDefinitionLoader.load 方法
    loader.load();
}
```
---

###### 3.9.3.1 BeanDefinitionLoader.load() 方法

```java
public int load() {
    int count = 0;
    
    // 主启动器就一个，继续 调用重载的 load 方法
    for (Object source : this.sources) {
        count += load(source);
    }
    return count;
}

// 根据传入的source类型，来决定用哪种方式加载，这里我们的主启动类是属于Class （Application.class）类型，所以继续调用重载的lead方法
private int load(Object source) {
    Assert.notNull(source, "Source must not be null");
    if (source instanceof Class<?>) {
        return load((Class<?>) source);
    }
    if (source instanceof Resource) {
        return load((Resource) source);
    }
    if (source instanceof Package) {
        return load((Package) source);
    }
    if (source instanceof CharSequence) {
        return load((CharSequence) source);
    }
    throw new IllegalArgumentException("Invalid source type " + source.getClass());
}

// class 类型的 load
// @SpringBootApplication 组合了一个 @SpringBootConfiguration
// 它又组合了一个 @Configuration 注解，@Configuration 的底层就是一个 @Component
private int load(Class<?> source) {
    if (isGroovyPresent()
            && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
        // Any GroovyLoaders added in beans{} DSL can contribute beans here
        GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source,
                GroovyBeanDefinitionSource.class);
        load(loader);
    }
    
    // 如果它是一个 Component，则用注解解析
    if (isComponent(source)) {
        this.annotatedReader.register(source);
        return 1;
    }
    return 0;
}
```
---

annotatedReader.register 再解读

```java
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
                        @Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {

    // 包装成 BeanDefinition
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setInstanceSupplier(instanceSupplier);
    
    // 解析 Scope信息，决定作用域
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    
    // 生成 Bean 的名称
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    // 解析 BeanDefinition 的注解
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    
    // 使用定制器修改这个 BeanDefinition
    for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
        customizer.customize(abd);
    }

    // 使用 BeanDefinitionHolder ，将 BeanDefinition 注册到 IOC 容器中
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    
    // 注册到 IOC 容器
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

---

#### 3.10 refreshContext 刷新容器上下文

```java
private void refreshContext(ConfigurableApplicationContext context) {
    
    // 直接调用
    refresh(context);
    
    if (this.registerShutdownHook) {
        try {
            // 注册一个关闭的钩子
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}

protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    // 强转成AbstractApplicationContext并调用其refresh方法
    // IOC容器启动最核心的方法
    ((AbstractApplicationContext) applicationContext).refresh();
}
```

---

#### 3.11 afterRefresh 刷新IOC容器后的处理

```java
// 空实现
protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {
}
```

---

####  3.12 listeners.started 发布started事件

```java
// 发布ApplicationReadyEvent事件
@Override
public void started(ConfigurableApplicationContext context) {
    context.publishEvent(new ApplicationStartedEvent(this.application, this.args, context));
}
```

---

####  3.13 callRunners 运行器回调

回调CommandLineRunner和ApplicationRunner的run方法，这个也是回调使用的，可以实现它用来扩展

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<>();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<>(runners)) {
        if (runner instanceof ApplicationRunner) {
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            callRunner((CommandLineRunner) runner, args);
        }
    }
}
```

---

### 4 applicationContext.refresh 容器核心实现

org.springframework.context.support.AbstractApplicationContext#refresh

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        // 1、初始化前的预处理
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 2、获取 BeanFactory ，加载所有 bean 的定义信息
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        // 3、BeanFactory 的预处理配置
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 4、准备 BeanFactory 完成后进行的后置处理，是个抽象
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            // 5、执行 BeanFactory 创建后的后置处理器，他会执行所有的 BeanDefinitionRegistryPostProcessor 和 BeanFactoruPostProcessor
            // 主要工作是包扫描、解析配置类等
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            // 6、注册 BeanPostProcessor（注册所有的 BeanPostProcessor ）
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            // 7、初始化 MessageSource 初始化国际化组件
            initMessageSource();

            // Initialize event multicaster for this context.
            // 8、初始化事务派发器
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // 9、子类的多态 onRefresh ，子类扩展的刷新动作
            onRefresh();

            // Check for listener beans and register them.
            // 10、注册监听器
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // 11、初始化剩余的单实例 Bean
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            // 12、完成容器的创建工作
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

---

#### 4.1 prepareRefresh 初始化前的预处理

记录启动时间，标记IOC容器状态，之后初始化属性

保存容器的早期事件(这个时候事件广播器还没初始化，所以需要保存需要广播的事件)

```java
/**
 * Prepare this context for refreshing, setting its startup date and
 * active flag as well as performing any initialization of property sources.
 */
protected void prepareRefresh() {
    // Switch to active.
    // 记录启动时间，切换IOC容器的状态
    this.startupDate = System.currentTimeMillis();
    
    // 标记 IOC 容器的关闭状态=fasle
    this.closed.set(false);
    
    // 标记 IOC 容器已激活
    this.active.set(true);

    if (logger.isDebugEnabled()) {
        if (logger.isTraceEnabled()) {
            logger.trace("Refreshing " + this);
        }
        else {
            logger.debug("Refreshing " + getDisplayName());
        }
    }

    // Initialize any placeholder property sources in the context environment.
    // 初始化属性配置
    initPropertySources();

    // Validate that all properties marked as required are resolvable:
    // see ConfigurablePropertyResolver#setRequiredProperties
    // 属性校验
    getEnvironment().validateRequiredProperties();

    // Store pre-refresh ApplicationListeners...
    // 初始化早期事件
    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    }
    else {
        // Reset local application listeners to pre-refresh state.
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    // Allow for the collection of early ApplicationEvents,
    // to be published once the multicaster is available...
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

---

#### 4.2 obtainFreshBeanFactory 获取BeanFactory，加载所有bean定义信息

先刷新后获取

基于 xml 配置文件的 ApplicationContext 中，在该步骤会解析 xml 配置文件，封装 BeanDefinition

但是，基于注解配置类的 ApplicationContext 就不太一样

GenericApplicationContext 的 refreshBeanFactory 方法中，有一个 CAS 判断的动作，它控制着 GenericApplicationContext 不能反复刷新

> xml配置文件的 ApplicationContext 可以反复刷新
> 注解配置类的 ApplicationContext 只能刷新一次

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}
```

---

##### 4.2.1 GenericApplicationContext.refreshBeanFactory

refreshBeanFactory是一个抽象方法，留给子类重写

对于XML配置的IOC容器和注解配置的IOC容器非别有一种实现 GenericApplicationContext 和 AbstractRefreshableApplicationContext 重写了它

AnnotationConfigServletWebServerApplicationContext 继承了 GenericApplicationContext

AbstractRefreshableApplicationContext就不进行分析了，因为现在都是基于注解驱动，很少用xml配置了

```java
// 简单设置了序列化ID
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
    }
    this.beanFactory.setSerializationId(getId());
}
```

---

#### 4.3 prepareBeanFactory BeanFactory的预处理配置

```java
/**
 * Configure the factory's standard context characteristics,
 * such as the context's ClassLoader and post-processors.
 * @param beanFactory the BeanFactory to configure
 */
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());

    // 给 BeanFactory 设置类加载器，表达式解析器
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    
    // 编码注册 ApplicationContextAwareProcessor
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    // 以下几个 Aware 接口的回调注入不由 BeanFactory 负责
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    // 以下几个类型的自动注入会直接绑定
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    // 由编码注册监听器钩子
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    // load Time weaver 相关的部分
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    // 注册默认的运行时环境 Environment
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

---

##### 4.3.3 new ApplicationContextAwareProcessor()

beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

注册了ApplicationContextAwareProcessor到IOC容器，并且忽略了一些Ware接口。我们发现注入的后置处理作用就是在挨个判断Ware类型，然后注入需要的数据

```java
@Override
@Nullable
public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
    AccessControlContext acc = null;

    // 支持 6 个 aware 接口的回调
    if (System.getSecurityManager() != null &&
            (bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
                    bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
                    bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
        acc = this.applicationContext.getBeanFactory().getAccessControlContext();
    }

    if (acc != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareInterfaces(bean);
            return null;
        }, acc);
    }
    else {
        invokeAwareInterfaces(bean);
    }

    return bean;
}

private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof EnvironmentAware) {
            ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
        }
        if (bean instanceof EmbeddedValueResolverAware) {
            ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
        }
        if (bean instanceof ResourceLoaderAware) {
            ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
        }
        if (bean instanceof ApplicationEventPublisherAware) {
            ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
        }
        if (bean instanceof MessageSourceAware) {
            ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
        }
        if (bean instanceof ApplicationContextAware) {
            ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
    }
}
```

##### 4.3.4 new ApplicationListenerDetector()

beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

注册了一个ApplicationListenerDetector的后置处理器，这个后置处理器会收集所有的监听器，把监听器挂载到applicationContext中，这样这些监听器后续才可以被事件广播器广播到

```java
@Override
// 把这些监听器挂进 ApplicationContext 中，这样这些监听器才可以被事件广播器使用
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean instanceof ApplicationListener) {
        // potentially not detected as a listener by getBeanNamesForType retrieval
        Boolean flag = this.singletonNames.get(beanName);
        if (Boolean.TRUE.equals(flag)) {
            // singleton bean (top-level or inner): register on the fly
            this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
        }
        else if (Boolean.FALSE.equals(flag)) {
            if (logger.isWarnEnabled() && !this.applicationContext.containsBean(beanName)) {
                // inner bean with other scope - can't reliably process events
                logger.warn("Inner bean '" + beanName + "' implements ApplicationListener interface " +
                        "but is not reachable for event multicasting by its containing ApplicationContext " +
                        "because it does not have singleton scope. Only top-level listener beans are allowed " +
                        "to be of non-singleton scope.");
            }
            this.singletonNames.remove(beanName);
        }
    }
    return bean;
}
```

#### 4.4 postProcessBeanFactory BeanFactory的后置处理

这个方法是个抽象方法，子类AnnotationConfigServletWebServerApplicationContext重写了这个方法

```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    super.postProcessBeanFactory(beanFactory);
    // 包扫描
    if (!ObjectUtils.isEmpty(this.basePackages)) {
        this.scanner.scan(this.basePackages);
    }
    if (!this.annotatedClasses.isEmpty()) {
        this.reader.register(ClassUtils.toClassArray(this.annotatedClasses));
    }
}
```

---

##### 4.4.1 super.postProcessBeanFactory

首先调用了父类ServletWebServerApplicationContext#postProcessBeanFactory方法

这里往IOC容器中注册了WebApplicationContextServletContextAwareProcessor，并且忽略了ServletContextAware

```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    beanFactory.addBeanPostProcessor(
            new WebApplicationContextServletContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    registerWebApplicationScopes();
}
```

###### 4.4.1.1 registerWebApplicationScopes()

逻辑就是把Web的request域和session域注册到IOC容器中，让IOC容器知道这两种作用域(Bean的作用域)

```java
private void registerWebApplicationScopes() {
    ExistingWebApplicationScopes existingScopes = new ExistingWebApplicationScopes(
            getBeanFactory());
    WebApplicationContextUtils.registerWebApplicationScopes(getBeanFactory());
    existingScopes.restore();
}
```

##### 4.4.2 包扫描

这里如果debug的话if是不进来的

后续的refresh#invokeBeanFactoryPostProcessors会调用到，这里先讲一下ClassPathBeanDefinitionScanner#scan方法执行流程

```java
// 包扫描
if (!ObjectUtils.isEmpty(this.basePackages)) {
    this.scanner.scan(this.basePackages);
}
```

---

###### 4.4.2.1 ClassPathBeanDefinitionScanner#scan

这里发现首先会调用findCandidateComponents方法，这里返回的是beanDefinition

然后后续对beanDefinition做一些处理然后调用registerBeanDefinition注册到IOC容器中

```java
/**
 * Perform a scan within the specified base packages.
 * @param basePackages the packages to check for annotated classes
 * @return number of beans registered
 */
// scan入口
public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

    doScan(basePackages);

    // Register annotation config processors, if necessary.
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }

    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}

/**
 * Perform a scan within the specified base packages,
 * returning the registered bean definitions.
 * <p>This method does <i>not</i> register an annotation config processor
 * but rather leaves this up to the caller.
 * @param basePackages the packages to check for annotated classes
 * @return set of beans registered if any for tooling registration purposes (never {@code null})
 */
// 执行 scan
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    // 真正的包扫描动作
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            // 处理 scope（默认情况下是 singleton）
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            
            // 生成 bean 的名称
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            
            // 处理 bean 中的 @Lazy、@Primary 等注解
            if (candidate instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                // 设置 AOP 相关的属性
                definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                
                // 注册进 BeanDefinitionRegistry
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
```

---

####### 4.4.2.1.1 findCandidateComponents(basePackage)

```java
/**
 * Scan the class path for candidate components.
 * @param basePackage the package to check for annotated classes
 * @return a corresponding Set of autodetected bean definitions
 */
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    }
    else {
        return scanCandidateComponents(basePackage);
    }
}

/**
 * 首先它将要扫描的包和一些前缀进行拼接
 * 前缀是 classpath*: ，后缀默认扫 **/.class
 * 中间部分调了一个 resolveBasePackage 方法，这个方法其实不看也能猜出来是把这个包名转换成文件路径
 * 拼接出来的包路径是： classpath:/com/example/demo/xx/x.class
 * 后面会调用getResources方法执行包扫描逻辑
 */
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        // 此处可处理 /**/service/*Service.class 这样的表达式
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        
        // 包扫描
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
        boolean traceEnabled = logger.isTraceEnabled();
        boolean debugEnabled = logger.isDebugEnabled();
        for (Resource resource : resources) {
            if (traceEnabled) {
                logger.trace("Scanning " + resource);
            }
            if (resource.isReadable()) {
                try {
                    // 加载 .class 字节码
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                    
                    // 如果符合匹配规则，则封装成 ScannedGenericBeanDefinition
                    if (isCandidateComponent(metadataReader)) {
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                        sbd.setResource(resource);
                        sbd.setSource(resource);
                        if (isCandidateComponent(sbd)) {
                            if (debugEnabled) {
                                logger.debug("Identified candidate component class: " + resource);
                            }
                            candidates.add(sbd);
                        }
                        else {
                            if (debugEnabled) {
                                logger.debug("Ignored because not a concrete top-level class: " + resource);
                            }
                        }
                    }
                    else {
                        if (traceEnabled) {
                            logger.trace("Ignored because not matching any filter: " + resource);
                        }
                    }
                }
                catch (Throwable ex) {
                    throw new BeanDefinitionStoreException(
                            "Failed to read candidate component class: " + resource, ex);
                }
            }
            else {
                if (traceEnabled) {
                    logger.trace("Ignored because not readable: " + resource);
                }
            }
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
    }
    return candidates;
}
```

在 scanCandidateComponents 方法中，获取所有的Resources

那么后续它使用了一个 MetadataReader 来解析.class文件，它就可以读取这个class的类定义信息、注解标注信息

之后要用 MetadataReader 来判断这个class是否为一个 Component

isCandidateComponent 方法会判断class是否被 @Component/@ManagedBean 标注，如果判断为 Component 后，会将这个 class 封装成 beanDefinition

![img.png](/asserts/images/2024-07-06-SpringBoot启动/scanCandidateComponents.png)


生成了BeanDefinition。然后后续就是遍历这些BeanDefinition然后生成BeanName，处理一些bean的元信息，然后注册到IOC容器中

---

#### 4.5 invokeBeanFactoryPostProcessors 创建BeanFactory创建后的后置处理器

org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors

BeanFactoryPostProcessor是一个接口

它提供一个方法，可以在BeanDefinition已经被加载，但没有Bean被实例化，可以对BeanFactory进行后置处理

还有一个字接口BeanDefinitionRegistryPostProcessor

执行时机是在所有Bean的定义信息即将被加载但未实例化时，也就是 BeanFactoryPostProcessor，可以实现往容器中添加BeanDefinition。

```java
/**
 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
 * respecting explicit order if given.
 * <p>Must be called before singleton instantiation.
 */
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    
    // go 执行 BeanFactory 后置处理器
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}
```

---

##### 4.2.1 PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors：回调后置处理器

org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, java.util.List<org.springframework.beans.factory.config.BeanFactoryPostProcessor>)

首先遍历一次，把BeanDefinitionRegistryPostProcessor的后置处理器单独挑出来，直接回调postProcessBeanDefinitionRegistry方法

把BeanFactory中所有的BeanDefinitionRegistryPostProcessor分成三部分：实现 PriorityOrdered 接口的、实现 Ordered 接口的，普通的。下面的流程是先进行筛选，然后进行排序，然后注册，最后执行回调，再清除

最后也会筛选出BeanFactoryPostProcessor，执行和上面BeanDefinitionRegistryPostProcessor的所有逻辑

```java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    // 首先调用 BeanDefinitionRegistryPostProcessor
    Set<String> processedBeans = new HashSet<>();

    // 判断 BeanFactory 类型，默认 SpringBoot 创建的 BeanFactory 是 DefaultListableBeanFactory
    // 实现了 BeanDifinitionRegistry
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        // 区分不同的后置处理器，并划分成不同的集合中
        // 将 BeanFactoryPostProcessor 与 BeanDefinitionRegistryPostProcessor 分离开
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.
        // 不要在这里初始化 BeanFactory：我们需要保留所有未初始化的常规 Bean，以便让 bean 工厂后处理器应用到它们
        // 独立实现 PriorityOrdered、Ordered 和其他的 BeanDefinitionRegistryPostProcessor 之间
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
        // 这部门实际上想表达的意思是，在创建 Bean 之前，要先执行这些
        // BeanDefinitionRegistryPostProcessor 的后置处理方法，并且实现了
        // PriorityOrdered 排序接口或实现了 Ordered 接口的 Bean 需要优先被加载
        
        // 下面一段是从 BeanFactory 中取出所有 BeanDefinitionRegistryPostProcessor 类型的全限定名（String[]）
        // 放到下面遍历，还要判断这些类里是否有实现 priorityOrdered 接口
        // 如果有，存到集合中，之后进行排序、统一回调这些后置处理器
        
        // 首先。调用实现 PriorityOrdered 接口的 BeanDefinitionRegistryPostProcessors
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
        // 接下来，执行实现了 Ordered 接口的 BeanDefinitionRegistryPostProcessor 
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
        // 最后，执行所有其他的 BeanDefinitionRegistryPostProcessor
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }

        // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
        // 先回调 BeanDefinitionRegistryPostProcessor 的 postProcessBeanFactory 方法
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        
        // 再回调 BeanFactoryPostProcessor 的 postProcessBeanFactory 方法 
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // Invoke factory processors registered with the context instance.
        // 如果 BeanFactory 没有实现 BeanDefinitionRegistry 接口，则进入下面的代码流程
        
        // 调用在上下文实例中注册的工厂处理器
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    // 下面的部分是回调 BeanFactoryPostProcessor，思路与上面的几乎一致
    String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // Finally, invoke all other BeanFactoryPostProcessors.
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, e.g. replacing placeholders in values...
    // 清理缓存
    beanFactory.clearMetadataCache();
}
```

---

###### 4.2.1.1 registryProcessor.postProcessBeanDefinitionRegistry

org.springframework.context.annotation.ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry

在上面回调各种BeanFactory的后置处理器，在spring内部有一个很重要的BeanFactoryPostProcessor是ConfigurationClassPostProcessor

他的作用是用于@Configuration类的扫描加载处理

**执行逻辑：**

取出BeanFactory的id，判断是否已经被调用过了

如果没有，会在被调用过的集合中加上当前BeanFactory的id

之后会调用processConfigBeanDefinitions方法

```java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    int registryId = System.identityHashCode(registry);
    if (this.registriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
                "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
    }
    if (this.factoriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
                "postProcessBeanFactory already called on this post-processor against " + registry);
    }
    this.registriesPostProcessed.add(registryId);

    // 解析配置类
    processConfigBeanDefinitions(registry);
}

/**
 * Build and validate a configuration model based on the registry of
 * {@link Configuration} classes.
 * 首先获取所有的配置类和组件，然后对配置类排序
 * 构造默认的Bean名称的生成器
 * 调用ConfigurationClassParser#parse方法解析配置类
 * 调用ConfigurationClassBeanDefinitionReader#loadBeanDefinitions加载配置类中的Bean定义信息，然后注册到IOC容器中
 */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    String[] candidateNames = registry.getBeanDefinitionNames();

    // 筛选出来所有的配置类
    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
                ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }

    // Return immediately if no @Configuration classes were found
    if (configCandidates.isEmpty()) {
        return;
    }

    // Sort by previously determined @Order value, if applicable
    // 配置类排序
    configCandidates.sort((bd1, bd2) -> {
        int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
        int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
        return Integer.compare(i1, i2);
    });

    // Detect any custom bean name generation strategy supplied through the enclosing application context
    // 构造默认的 BeanNameGenerator bean 的名称生成器
    SingletonBeanRegistry sbr = null;
    if (registry instanceof SingletonBeanRegistry) {
        sbr = (SingletonBeanRegistry) registry;
        if (!this.localBeanNameGeneratorSet) {
            BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
            if (generator != null) {
                this.componentScanBeanNameGenerator = generator;
                this.importBeanNameGenerator = generator;
            }
        }
    }

    if (this.environment == null) {
        this.environment = new StandardEnvironment();
    }

    // Parse each @Configuration class
    // 真正解析配置类的组件 ConfigurationClassParser
    ConfigurationClassParser parser = new ConfigurationClassParser(
            this.metadataReaderFactory, this.problemReporter, this.environment,
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);

    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
    do {
        
        // 解析配置类
        parser.parse(candidates);
        parser.validate();

        Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
        configClasses.removeAll(alreadyParsed);

        // Read the model and create bean definitions based on its content
        if (this.reader == null) {
            this.reader = new ConfigurationClassBeanDefinitionReader(
                    registry, this.sourceExtractor, this.resourceLoader, this.environment,
                    this.importBeanNameGenerator, parser.getImportRegistry());
        }
        
        // 加载配置类的内容 在这里注册所有的 BeanDefinition
        this.reader.loadBeanDefinitions(configClasses);
        alreadyParsed.addAll(configClasses);

        candidates.clear();
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            String[] newCandidateNames = registry.getBeanDefinitionNames();
            Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
            Set<String> alreadyParsedClasses = new HashSet<>();
            for (ConfigurationClass configurationClass : alreadyParsed) {
                alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
            }
            for (String candidateName : newCandidateNames) {
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                            !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());

    // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
    if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
        sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
    }

    if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
        // Clear cache in externally provided MetadataReaderFactory; this is a no-op
        // for a shared cache since it'll be cleared by the ApplicationContext.
        ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
    }
}
```

---

####### 4.2.1.1.1 解析配置类

org.springframework.context.annotation.ConfigurationClassParser#parse(java.util.Set<org.springframework.beans.factory.config.BeanDefinitionHolder>)

遍历每一个BeanDefinition，并根据类型来决定如何解析，SpringBoot通常使用的注解配置会进入第一个if结构

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }

    this.deferredImportSelectorHandler.process();
}
```

---

重载 parse 方法

```java
protected final void parse(@Nullable String className, String beanName) throws IOException {
    Assert.notNull(className, "No bean class name for configuration class bean definition");
    MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className);
    processConfigurationClass(new ConfigurationClass(reader, beanName));
}
```
---

org.springframework.context.annotation.ConfigurationClassParser#processConfigurationClass

在processConfigurationClass判断下是否已经重复处理了，最终会调用processConfigurationClass方法

```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }

    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    if (existingClass != null) {
        // 如果配置类已经 @import 过了，则跳过
        if (configClass.isImported()) {
            if (existingClass.isImported()) {
                existingClass.mergeImportedBy(configClass);
            }
            // Otherwise ignore new imported config class; existing non-imported class overrides it.
            return;
        }
        else {
            // Explicit bean definition found, probably replacing an import.
            // Let's remove the old one and go with the new one.
            this.configurationClasses.remove(configClass);
            this.knownSuperclasses.values().removeIf(configClass::equals);
        }
    }

    // Recursively process the configuration class and its superclass hierarchy.
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        // 真正干活的
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    this.configurationClasses.put(configClass, configClass);
}
```

---

####### 4.2.1.1.2 doProcessConfigurationClass

org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass

```java
/**
 * Apply processing and build a complete {@link ConfigurationClass} by reading the
 * annotations, members and methods from the source class. This method can be called
 * multiple times as relevant sources are discovered.
 * @param configClass the configuration class being build
 * @param sourceClass a source class
 * @return the superclass, or {@code null} if none found or previously processed
 */
@Nullable
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {

    // 处理 @Component 注解
    if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
        // Recursively process any member (nested) classes first
        processMemberClasses(configClass, sourceClass);
    }

    // Process any @PropertySource annotations
    // 处理 @PropertySource 注解
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Process any @ComponentScan annotations
    // 处理 @ComponentScan 注解
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        // 如果有 @ComponentScans，则要取出里面所有的 @ComponentScan 依次扫描
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            // 借助 ComponentScanAnnotationParser 扫描
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            // 是否扫描到了其他的注解配置类
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    // 如果扫描到了递归解析
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    // 处理 @Import 注解
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // Process any @ImportResource annotations
    // 处理 @ImportResource 注解
    // 注解配置类上标注 @ImportResource 可以导入 xml 配置文件
    // 而解析这些 @importResource 的逻辑就在这里
    // 先把配置文件的路径放入了 configClass，ConfigurationClassPostProcessor 的后置处理 processConfigBeanDefinitions 方法中，parse 方法下面有 loadBeanDefinitions
    AnnotationAttributes importResource =
            AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    // 处理 @Bean 注解，这里也只是存起来，处理实在 retrieveBeanMethodMetadata 方法中
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces
    // 处理父接口
    processInterfaces(configClass, sourceClass);

    // Process superclass, if any
    // 父类的返回
    // 如果配置类存在父类的话，父类也应该一起加载，所以这里会取到配置类的父类，并继续递归处理
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java") &&
                !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    return null;
}
```

---

####### 4.2.1.1.3 loadBeanDefinitions 解析配置类中的内容

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitions

```java
/**
 * Read {@code configurationModel}, registering bean definitions
 * with the registry based on its contents.
 */
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
    TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
    for (ConfigurationClass configClass : configurationModel) {
        // 循环加载
        loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
    }
}

/**
 * Read a particular {@link ConfigurationClass}, registering bean definitions
 * for the class itself and all of its {@link Bean} methods.
 */
private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

    // 与条件装配有关
    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        return;
    }

    // 如果当前配置类型是被 @import 的，要把自己注册进 BeanFactory
    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    
    // 注册 @Bean 注解方法
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }

    // 注册来自 xml 配置文件的 Bean
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    
    // 注册来自 ImportBeanDefinitionRegistrar 的 Bean
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

---

#### 4.6 registerBeanPostProcessors 注册BeanPostProcessor

org.springframework.context.support.AbstractApplicationContext#registerBeanPostProcessors

首先从IOC容器中获取了所有实现了BeanPostProcessor的beanName

这里和之前逻辑一样，按照规则给所有后置处理器，首先注册所有实现了priorityOrdered接口的后置处理器，然后注册所有实现了Ordered接口的后置处理器，最后注册所有普通的后置处理器

```java
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

    // 从 bean 容器中获取所有实现了 BeanPostProcessor 的 bean 的 beanName
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

    // Register BeanPostProcessorChecker that logs an info message when
    // a bean is created during BeanPostProcessor instantiation, i.e. when
    // a bean is not eligible for getting processed by all BeanPostProcessors.
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

    // Separate between BeanPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    // 根据排序规则，给所有的后置处理器分类
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            // 注意此处， priorityOrdered 类型的后置处理器被提前初始化了
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
    // 首先，注册所有实现了 priorityOrdered 接口的后置处理器
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

    // Next, register the BeanPostProcessors that implement Ordered.
    // 其次，注册所有实现了 ordered 接口的后置处理器
    List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
    for (String ppName : orderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        orderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, orderedPostProcessors);

    // Now, register all regular BeanPostProcessors.
    // 最后，注册所有普通的后置处理器
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
    for (String ppName : nonOrderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        nonOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

    // Finally, re-register all internal BeanPostProcessors.
    // 注意此处，所有 MergedBeanDefinitionPostProcessor 又被注册了一次
    sortPostProcessors(internalPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, internalPostProcessors);

    // Re-register post-processor for detecting inner beans as ApplicationListeners,
    // moving it to the end of the processor chain (for picking up proxies etc).
    // 最后又注册了一个 ApplicationListenerDetector
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

---

##### 4.6.1 MergedBeanDefinitionPostProcessor

这里在上面的方法中会又单独注册了MergedBeanDefinitionPostProcessor

这个接口是运行时用于合并bean定义的后处理器回调接口

可以实现接口用于创建Bean实例的合并Bean定义

这个其中有一个很重要的实现就是AutowiredAnnotationBeanPostProcessor

##### 4.6.1.1 MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition

org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition

findAutowiringMetadata：先获取注入的依赖

InjectionMetadata#checkConfigMembers：再进行对象检查

```java
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
    metadata.checkConfigMembers(beanDefinition);
}

// 这里首先使用双重检查锁来保证线程安全，之后会调用buildAutowiringMetadata方法构建自动装配的metadata
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
    // Fall back to class name as cache key, for backwards compatibility with custom callers.
    String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
    // Quick check on the concurrent map first, with minimal locking.
    InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
    if (InjectionMetadata.needsRefresh(metadata, clazz)) {
        synchronized (this.injectionMetadataCache) {
            metadata = this.injectionMetadataCache.get(cacheKey);
            if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                if (metadata != null) {
                    metadata.clear(pvs);
                }
                metadata = buildAutowiringMetadata(clazz);
                this.injectionMetadataCache.put(cacheKey, metadata);
            }
        }
    }
    return metadata;
}

// 在这个while循环就是一步一步往父类上爬，判断条件是是否到了Object方法
// 循环体中，先是反射遍历当前类的属性，并判断上面是否有@Autowired等注解
// 之后有获取方法上的注解也保存进去
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
    List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
    Class<?> targetClass = clazz;

    // 循环获取父类信息
    do {
        final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

        // 循环获取类上的属性，并判断是否有 @Autowired 等注入类注解
        ReflectionUtils.doWithLocalFields(targetClass, field -> {
            AnnotationAttributes ann = findAutowiredAnnotation(field);
            if (ann != null) {
                if (Modifier.isStatic(field.getModifiers())) {
                    if (logger.isInfoEnabled()) {
                        logger.info("Autowired annotation is not supported on static fields: " + field);
                    }
                    return;
                }
                boolean required = determineRequiredStatus(ann);
                currElements.add(new AutowiredFieldElement(field, required));
            }
        });

        // 循环获取类上的方法，并判断是否有需要依赖的项
        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
            if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                return;
            }
            AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
            if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                if (Modifier.isStatic(method.getModifiers())) {
                    if (logger.isInfoEnabled()) {
                        logger.info("Autowired annotation is not supported on static methods: " + method);
                    }
                    return;
                }
                if (method.getParameterCount() == 0) {
                    if (logger.isInfoEnabled()) {
                        logger.info("Autowired annotation should only be used on methods with parameters: " +
                                method);
                    }
                }
                boolean required = determineRequiredStatus(ann);
                PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                currElements.add(new AutowiredMethodElement(method, required, pd));
            }
        });

        elements.addAll(0, currElements);
        targetClass = targetClass.getSuperclass();
    }
    while (targetClass != null && targetClass != Object.class);

    return new InjectionMetadata(clazz, elements);
}
```

---

##### 4.6.2 ApplicationListenerDetector

beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));

registerBeanPostProcessors最后会又添加了一个后置处理器ApplicationListenerDetector

这个后置处理器就是收集所有的监听器

在他的postProcessAfterInitialization方法中，会去把监听器类型的bean注册到IOC容器中

org.springframework.context.support.ApplicationListenerDetector#postProcessAfterInitialization

```java
/**
 * 用来关联所有的监听器饮用
 * 同样的，监听器在创建的时候，也需要 ApplicationListenerDetector
 * 把这些监听器挂在 ApplicationContext 中，这样这些监听器才可以被事件广播器使用
 */
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean instanceof ApplicationListener) {
        // potentially not detected as a listener by getBeanNamesForType retrieval
        Boolean flag = this.singletonNames.get(beanName);
        if (Boolean.TRUE.equals(flag)) {
            // singleton bean (top-level or inner): register on the fly
            this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
        }
        else if (Boolean.FALSE.equals(flag)) {
            if (logger.isWarnEnabled() && !this.applicationContext.containsBean(beanName)) {
                // inner bean with other scope - can't reliably process events
                logger.warn("Inner bean '" + beanName + "' implements ApplicationListener interface " +
                        "but is not reachable for event multicasting by its containing ApplicationContext " +
                        "because it does not have singleton scope. Only top-level listener beans are allowed " +
                        "to be of non-singleton scope.");
            }
            this.singletonNames.remove(beanName);
        }
    }
    return bean;
}
```

---

#### 4.7 initMessageSource 初始化MessageSource

org.springframework.context.support.AbstractApplicationContext#initMessageSource

这个组件是实现国际化接口的，他的默认实现是DelegatingMessageSource

功能：将字符串和参数数组格式化为一个国际化后的消息

```java
	/**
 * Initialize the MessageSource.
 * Use parent's if none defined in this context.
 */
protected void initMessageSource() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    
    // 检查是否存在 MessageSource 组件，如果有，直接赋值
    if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
        this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
        // Make MessageSource aware of parent MessageSource.
        if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
            HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
            if (hms.getParentMessageSource() == null) {
                // Only set parent context as parent MessageSource if no parent MessageSource
                // registered already.
                hms.setParentMessageSource(getInternalParentMessageSource());
            }
        }
        if (logger.isTraceEnabled()) {
            logger.trace("Using MessageSource [" + this.messageSource + "]");
        }
    }
    else {
        // Use empty MessageSource to be able to accept getMessage calls.
        // 如果没有，创建一个，并注册到 BeanFactory 中
        DelegatingMessageSource dms = new DelegatingMessageSource();
        dms.setParentMessageSource(getInternalParentMessageSource());
        this.messageSource = dms;
        beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
        if (logger.isTraceEnabled()) {
            logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
        }
    }
}
```

---

#### 4.8 initApplicationEventMulticaster 初始化事件派发器

org.springframework.context.support.AbstractApplicationContext#initApplicationEventMulticaster

首先会判断IOC容器中是否有名称为applicationEventMulticaster的bean

如果没有就默认注册一个applicationEventMulticaster

类型是SimpleApplicationEventMulticaster

这个组件是一个事件发布器，用来做事件的发布

```java
/**
 * Initialize the ApplicationEventMulticaster.
 * Uses SimpleApplicationEventMulticaster if none defined in the context.
 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
 */
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    }
    else {
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
        if (logger.isTraceEnabled()) {
            logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
                    "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
        }
    }
}
```

---

##### 4.8.1 SimpleApplicationEventMulticaster

最终会调用到ApplicationListener 的 onApplicationEvent方法，根据对应的事件匹配对应的listener

---

#### 4.9 onRefresh 子类扩展刷新(模版方法)

在springBoot中对这个方法进行了一个扩展

ServletWebServerApplicationContext#Onrefresh方法,这里发现它会创建一个WebServer也就是tomcat

org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#onRefresh

```java
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();
    if (webServer == null && servletContext == null) {
        // 这一步创建了嵌入式 Servlet 容器的工厂
        ServletWebServerFactory factory = getWebServerFactory();
        this.webServer = factory.getWebServer(getSelfInitializer());
    }
    else if (servletContext != null) {
        try {
            getSelfInitializer().onStartup(servletContext);
        }
        catch (ServletException ex) {
            throw new ApplicationContextException("Cannot initialize servlet context", ex);
        }
    }
    initPropertySources();
}

/**
 * Returns the {@link ServletWebServerFactory} that should be used to create the
 * embedded {@link WebServer}. By default this method searches for a suitable bean in
 * the context itself.
 * @return a {@link ServletWebServerFactory} (never {@code null})
 */
protected ServletWebServerFactory getWebServerFactory() {
    // Use bean names so that we don't consider the hierarchy
    // 获取 IOC 容器中类型为 ServletWebServerFactory 的 Bean
    // 默认的 Tomcat 创建工厂应该从这里取出： TomcatServletWebServerFactory ，实现了 ServletWebServerFactory 接口
    String[] beanNames = getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);
    if (beanNames.length == 0) {
        throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to missing "
                + "ServletWebServerFactory bean.");
    }
    if (beanNames.length > 1) {
        throw new ApplicationContextException("Unable to start ServletWebServerApplicationContext due to multiple "
                + "ServletWebServerFactory beans : " + StringUtils.arrayToCommaDelimitedString(beanNames));
    }
    return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
}
```

---

#### 4.10 registerListeners 注册监听器

这里监听器已经在IOC容器中注册好了，取出来后要放入事件广播器，以方便事件广播器广播事件

广播早期事件（在之前prepareRefresh方法中会使用earlyApplicationEventsb保存早期事件，这里是没有早期事件的，是留给开发者的，在后置处理器和监听器都被创建好，其余的单实例Bean还没有创建时，提供一个预留的时机来处理一些额外的事情）

```java
/**
 * Add beans that implement ApplicationListener as listeners.
 * Doesn't affect other listeners, which can be added without being beans.
 */
protected void registerListeners() {
    // Register statically specified listeners first.
    // 注册所有已经被创建出来的 ApplicationListener，绑定到 ApplicationEventMulticaster中
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    // 此处不会实例化 ApplicationListener，而只绑定 name
    // 此处那些没有初始化的 ApplicationListener 并没有被实例化，而是会等到下一步 finishBeanFactoryInitialization 方法中才会被创建出来
    // 不要再此处创建，因为框架希望让那些后置处理器取干预它们
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }

    // Publish early application events now that we finally have a multicaster...
    // 广播早期事件
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (earlyEventsToProcess != null) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}
```

---

#### 4.11 finishBeanFactoryInitialization 初始化单实例Bean

```java
/**
 * Finish the initialization of this context's bean factory,
 * initializing all remaining singleton beans.
 */
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Initialize conversion service for this context.
    // 初始化 ConversionService 这个 ConversionService 是用于类型转换的服务接口
    // 它的工作，是将配置文件 properties 中的数据，进行类型转换，得到 Spring 真正想要的数据类型
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    // 嵌入式解析起 EnbeddedValueResolver 的组件注册，它负责解析占位符和表达式
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    // 尽早初始化 LoadTimeWeaverAware 类型的 bean，以允许尽早注册其变化器
    // 这部分与 LoadTimeWeaverAware 部分有关，实际上与 AspectJ 有关
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    // 停用临时类加载器
    beanFactory.setTempClassLoader(null);

    // Allow for caching all bean definition metadata, not expecting further changes.
    // 允许缓存所以 bean 定义元数据
    beanFactory.freezeConfiguration();

    // Instantiate all remaining (non-lazy-init) singletons.
    // 初始化非懒加载的 bean
    beanFactory.preInstantiateSingletons();
}
```

---

##### 4.11.1 DefaultListableBeanFactory#preInstantiateSingletons

org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons

```java
@Override
public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
        logger.trace("Pre-instantiating singletons in " + this);
    }

    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    // 此处循环初始化剩余的非延迟加载的单例 bean
    for (String beanName : beanNames) {
        // 先合并 BeanDefinition
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        
        // 不是抽象的，不是延迟加载的 单实例 Bean 要初始化 
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                // factoryBean 默认不立即初始化，除非指定 isEagerInit = true
                if (bean instanceof FactoryBean) {
                    final FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                        ((SmartFactoryBean<?>) factory)::isEagerInit,
                                getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            }
            else {
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    // 这个初始化的最后结点，实在 SpringFramework 4.1 之后才有的，而支撑这个阶段的核心接口是 SmartInitializingSingleton
    // SmartInitializingSingleton 的回调时机，是所有的非延迟加载的单实例 bean 都初始化完成后，再挨个执行 afterSingletonsInstantiated 方法
    // 调用的方式非常简单，逐个判断就完事了
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```

--- 

###### 4.11.1.1 AbstractBeanFactory#getBean

org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String)

```java
@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

/**
 * Return an instance, which may be shared or independent, of the specified bean.
 * @param name the name of the bean to retrieve
 * @param requiredType the required type of the bean to retrieve
 * @param args arguments to use when creating a bean instance using explicit arguments
 * (only applied when creating a new instance as opposed to retrieving an existing one)
 * @param typeCheckOnly whether the instance is obtained for a type check,
 * not for actual use
 * @return an instance of the bean
 * @throws BeansException if the bean could not be created
 */
@SuppressWarnings("unchecked")
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    // 此处是解决别名 BeanName的映射，getBean时可以传入 bean 的别名，此处可以根据别名找到 BeanName
    final String beanName = transformedBeanName(name);
    Object bean;

    // Eagerly check singleton cache for manually registered singletons.
    // 先尝试从之前实例化好的 Bean 中找有没有这个 Bean，如果能找到，说明已经被实例化了，可以直接返回
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isTraceEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    // 上面 get 不到 bean
    else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        // 如果搜不到，但该 Bean 正在被创建，说明产生了循环引用且无法处理，只能抛出异常
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // Check if bean definition exists in this factory.
        // 检查这个 Bean 对应的 BeanDefinition 在 IOC容器中是否存在
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            // 如果检查不存在，看看父容器有没有（Web环境会存在父子容器现象）
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                        nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else if (requiredType != null) {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        // 走到这个地方，证明 Bean 确实要被创建了，标记 Bean 被创建
        // 该设计是防止多线程同时到这里，引发多次创建的问题
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            // 合并 BeanDefinition
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            // 处理当前 Bean 的 bean 依赖（@DependsOn注解的依赖）
            // 在创建一个 Bean 之前，可能这个 Bean 需要依赖其他的 Bean
            // 通过这个步骤，可以先递归的将这个 Bean 显式声明的需要的其他 Bean 先创建出来
            // 通过 bean 标签的 depends-on 属性或 @DependsOn 注解进行显式声明
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // Create bean instance.
            // 作用域为 singleton 单实例 bean 创建
            if (mbd.isSingleton()) {
                // 匿名内部类执行完成后的 getSingleton 调用
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            // 作用域为 prototype 类型
            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            // 作用域既不是 singleton，又不是 prototype 那就按照实际情况创建
            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                    "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // Check if required type matches the type of the actual bean instance.
    // 检查所需的类型是否与实际 bean 实例的类型匹配，类型不匹配则抛出异常
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        }
        catch (TypeMismatchException ex) {
            if (logger.isTraceEnabled()) {
                logger.trace("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

---

####### 4.11.1.1.1 getSingleton 尝试获取单实例Bean(解决循环依赖)

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)

这里就是首先尝试通过beanName从IOC容器中获取对应的bean，如果不存在才会走后续的实例化过程

这个流程是为了解决循环依赖，这个后续会做详细剖析

```java
/**
 * Return the (raw) singleton object registered under the given name.
 * <p>Checks already instantiated singletons and also allows for an early
 * reference to a currently created singleton (resolving a circular reference).
 * @param beanName the name of the bean to look for
 * @param allowEarlyReference whether early references should be created or not
 * @return the registered singleton object, or {@code null} if none found
 */
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

---

####### 4.11.1.1.2 isPrototypeCurrentlyInCreation 创建原型Bean时会校验的

org.springframework.beans.factory.support.AbstractBeanFactory#isPrototypeCurrentlyInCreation

检查循环依赖的方法：isPrototypeCurrentlyInCreation，这个方法是创建原型Bean时会校验的

如果当前线程中在创建一个scope=prototype的Bean，并且当前要创建的Bean跟这个线程中创建的Bean的name一致

则会认为出现了多实例Bean的循环依赖，会引发异常

---

####### 4.11.1.1.3 markBeanAsCreated 标记准备创建的Bean

org.springframework.beans.factory.support.AbstractBeanFactory#markBeanAsCreated

IOC容器会把所有创建过的Bean的name都缓存起来，这样下次会判断是否是重复创建

```java
/**
 * Mark the specified bean as already created (or about to be created).
 * <p>This allows the bean factory to optimize its caching for repeated
 * creation of the specified bean.
 * @param beanName the name of the bean
 */
protected void markBeanAsCreated(String beanName) {
    if (!this.alreadyCreated.contains(beanName)) {
        synchronized (this.mergedBeanDefinitions) {
            if (!this.alreadyCreated.contains(beanName)) {
                // Let the bean definition get re-merged now that we're actually creating
                // the bean... just in case some of its metadata changed in the meantime.
                clearMergedBeanDefinition(beanName);
                this.alreadyCreated.add(beanName);
            }
        }
    }
}
```

---

####### 4.11.1.1.4 getMergedBeanDefinition 合并BeanDefinition，处理显式依赖

这部分会解析@DependsOn注解标注声明的Bean，并预先的构建它，被依赖的Bean也是通过getBean方法来创建，思路一致

org.springframework.beans.factory.support.AbstractBeanFactory#getMergedBeanDefinition(java.lang.String)

---

####### 4.11.1.1.5 准备创建 Bean

createBean创建单实例Bean，这里createBean方法是通过getSingleton方法传入匿名内部类，调用的createBean方法

```java
sharedInstance = getSingleton(beanName, () -> {
    try {
        return createBean(beanName, mbd, args);
    }
    catch (BeansException ex) {
        // Explicitly remove instance from singleton cache: It might have been put there
        // eagerly by the creation process, to allow for circular reference resolution.
        // Also remove any beans that received a temporary reference to the bean.
        destroySingleton(beanName);
        throw ex;
    }
});
bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);

```
---

####### 4.11.1.1.6 getSingleton循环依赖

如果当前准备创建的Bean还没有在IOC容重，就标记一下它

把当前准备创建的beanName放入singletonsCurrentlyInCreation中

他的作用是解决循环依赖，这个后续会说。然后会调用singletonFactory.getObject()方法，这个会调用到createBean

```java
/**
 * Return the (raw) singleton object registered under the given name,
 * creating and registering a new one if none registered yet.
 * @param beanName the name of the bean
 * @param singletonFactory the ObjectFactory to lazily create the singleton
 * with, if necessary
 * @return the registered singleton object
 */
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        // 双重检锁再检查一次单实例 bean 的实例，判断是否有当前 BeanName 的 bean
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            if (this.singletonsCurrentlyInDestruction) {
                throw new BeanCreationNotAllowedException(beanName,
                        "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                        "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            
            // 标记当前 bean
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<>();
            }
            try {
                // createBean
                // 如果单实例 bean 的缓存中真的没有，那就返回对象，调用 createBean 方法
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (IllegalStateException ex) {
                // Has the singleton object implicitly appeared in the meantime ->
                // if yes, proceed with it since the exception indicates that state.
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw ex;
                }
            }
            catch (BeanCreationException ex) {
                if (recordSuppressedExceptions) {
                    for (Exception suppressedException : this.suppressedExceptions) {
                        ex.addRelatedCause(suppressedException);
                    }
                }
                throw ex;
            }
            finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                afterSingletonCreation(beanName);
            }
            
            // 新创建的单实例 bean 要存入单实例 bean 的缓存中
            if (newSingleton) {
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```

---

###### 4.11.1.2 AbstractAutowireCapableBeanFactory#createBean

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])

这里面有两个重要的部分，AOP的入口和真正创建Bean的入口

```java
/**
 * Central method of this class: creates a bean instance,
 * populates the bean instance, applies post-processors, etc.
 * @see #doCreateBean
 */
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    if (logger.isTraceEnabled()) {
        logger.trace("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
        // 前置校验
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        // 后置处理器拦截创建 bean 这里是给后置处理器一个机会来创建代理 bean 而不是目标 bean 实例
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }

    try {
        // 真正的创建 bean
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isTraceEnabled()) {
            logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        // A previously detected exception with proper bean creation context already,
        // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}
```

---

####### 4.11.1.2.1 resolveBeforeInstantiation（aop）

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation

```java
	/**
 * Apply before-instantiation post-processors, resolving whether there is a
 * before-instantiation shortcut for the specified bean.
 * @param beanName the name of the bean
 * @param mbd the bean definition for the bean
 * @return the shortcut-determined bean instance, or {@code null} if none
 */
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                // 执行所有 InstantiationAwareBeanPostProcessor
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        
        // 如果成功创建出了 bean ，则执行 BeanPostProcessor 的后置初始化
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}
```

---

####### 4.11.1.2.2 doCreateBean

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

createBeanInstance：创建Bean对象

addSingletonFactory：Bean放入缓存(涉及循环依赖)

populateBean：属性复制和自动注入

initializeBean：初始化后处理

```java
 protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, Object[] args) {
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
    }

    if (instanceWrapper == null) {
        instanceWrapper = this.createBeanInstance(beanName, mbd, args);
    }

    final Object bean = instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null;
    Class beanType = instanceWrapper != null ? instanceWrapper.getWrappedClass() : null;
    synchronized(mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            mbd.postProcessed = true;
        }
    }

    boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
    if (earlySingletonExposure) {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
        }

        this.addSingletonFactory(beanName, new ObjectFactory() {
            public Object getObject() throws BeansException {
                return AbstractAutowireCapableBeanFactory.this.getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    Object exposedObject = bean;

    try {
        this.populateBean(beanName, mbd, instanceWrapper);
        exposedObject = this.initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable var14) {
        Throwable ex = var14;
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException)ex).getBeanName())) {
            throw (BeanCreationException)ex;
        }

        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = this.getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                String[] dependentBeans = this.getDependentBeans(beanName);
                Set actualDependentBeans = new LinkedHashSet(dependentBeans.length);

                for(int i = 0; i < dependentBeans.length; ++i) {
                    String dependentBean = dependentBeans[i];
                    if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }

                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been " + "wrapped. This means that said other beans do not use the final version of the " + "bean. This is often the result of over-eager type matching - consider using " + "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
    return exposedObject;
}
```

--- 

#### 4.12 finishRefresh 完成容器的创建工作

org.springframework.context.support.AbstractApplicationContext#finishRefresh

清除资源缓存

初始化生命周期处理器也就是DefaultLifecycleProcessor

getLifecycleProcessor().onRefresh()会从IOC容器中找出所有的Lifecycle类型的Bean，遍历回调start方法

publishEvent(new ContextRefreshedEvent(this))发布ContextRefreshedEvent事件，代表IOC容器已经刷新完毕

```java
/**
 * Finish the refresh of this context, invoking the LifecycleProcessor's
 * onRefresh() method and publishing the
 * {@link org.springframework.context.event.ContextRefreshedEvent}.
 */
protected void finishRefresh() {
    // Clear context-level resource caches (such as ASM metadata from scanning).
    // 清除上下文级别的资源缓存
    clearResourceCaches();

    // Initialize lifecycle processor for this context.
    // 为当前 ApplicationContext 初始化一个生命周期处理器
    initLifecycleProcessor();

    // Propagate refresh to lifecycle processor first.
    // 将refresh的动作传播到生命周期处理器，
    getLifecycleProcessor().onRefresh();

    // Publish the final event.
    publishEvent(new ContextRefreshedEvent(this));

    // Participate in LiveBeansView MBean, if active.
    // 广播事件
    LiveBeansView.registerApplicationContext(this);
}
```
