---
layout:      post
title:       "SpringBoot启动流程"
subtitle:    "SpringBoot启动流程"
author:      "Ekko"
header-img:  "img/bg/bg-20240706.jpg"
catalog:     true
tags:
  - 学习笔记
  - SpringBoot
---

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

AbstractApplicationContext#refresh

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        // 初始化前的预处理
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 获取 BeanFactory ，加载所有 bean 的定义信息
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
