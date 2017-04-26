---
categories:
  - Technology
tags:
  - Spring
---
# 一 IoC的本质
## 1.1 本质目的：___解耦___
假设有A,B,C,D,E几个class, 其中A对象依赖于B,C,D,E,如下随着对象的增多或者对象的修改，这种代码之间的耦合很恐怖。
```Java
class A{
  B b;
  C c;
  D d;
  public A(){
    b = new B();
    c = new C();
    d = new D();
  }
}
```
## 1.2 本质实现：___工厂类+反射___

以上面的class A，想一下，如何不用b = new B()来实现给A对象中的b成员赋值？

### 1.2.1 new的替代者：工厂类
* 初级：为A,B,C,D每个class都来一个工厂类
* 高级：抽象的统一工厂类实现，查找A,B,C,D的定义，根据信息生成对象（Spring IoC容器-->Bean对象）

### 1.2.2 赋值的替代者：反射
* 依赖注入(Dependency Injection, DI): 被动的接收对象, 是IoC容器实现的控制反转的主要方式
* 依赖查找(Dependency Lookup,DL): 主动在需要的时间通过调用框架提供的方法来获取对象

如何实现DI
* interface
* set方法（@Autowired的本质）
* 构造函数

## 1.3 为什么叫控制反转(IoC)？
对象依赖关系的满足由主动实现到被动接受的转变，就是所谓的控制反转了。

# 二 Spring IoC的实现
下面的所有代码均以Spring 4.2.4.RELEASE为例说明，会有部分代码实例，如果想直接看结论可以跳到最后的总结部分。
## 2.1 前序：从Main函数到Spring Context
我们项目使用的是jetty，以我们的例子为流程简单说下Spring IoC启动的前序内容，整体上讲主要包括下面两个步骤
1. Bootstrap: main函数的入口，寻找项目中的jetty.xml文件
2. WebAppContext：org.eclipse.jetty.webapp.WebAppContext,jetty.xml中注册的handler,负责寻找解析项目路径下的web.xml文件，web.xml中的内容一般如下：
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/spring-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

## 2.2 找到Spring合适的ApplicationContext
根据web.xml按照listener → filter → servlet的顺序加载，首先是org.springframework.web.context.ContextLoaderListener

![ContextLoaderListener的继承关系](http://o9l56z0kf.bkt.clouddn.com/image/blog/spring-context-loader-listener.png)

ContextLoaderListener实现javax.servlet.ServletContextListener接口，实现了其中的public void contextInitialized(ServletContextEvent event)方法，并最终调用继承自ContextLoader中的WebApplicationContext initWebApplicationContext(ServletContext servletContext)方法。

下面几个步骤(没有按照代码栈执行顺序查找，而基本是实际运行的顺序)是Spring查找ApplicationContext类的过程：

 1. Spring目录中包含ContextLoader.properties文件中，其内容是org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext
 2. contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());这句执行结果默认就是取到的ContextLoader.properties文件中的XmlWebApplicationContext
 3. ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());这句根据上面得到的ApplicationContext的类名字得到类
 4. (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

Spring除了提供的XmlWebApplicationContext还有其他几种ApplicationContext如
* AnnotationConfigWebApplicationContext
* ClassPathXmlApplicationContext
* FileSystemXmlApplicationContext
* StaticWebApplicationContext
* EmbeddedWebApplicationContext
* GroovyWebApplicationContext

## 2.3 建立起BeanFactory
下面的介绍主要参考了[http://wiki.sankuai.com/pages/viewpage.action?pageId=449740666](http://wiki.sankuai.com/pages/viewpage.action?pageId=449740666)，前方大量代码，代码恐惧症者绕行~,可直接跳到后面的结论部分

从上面提到过ContextLoader回到这里来，在public WebApplicationContext initWebApplicationContext(ServletContext servletContext)中:
```java
...略...
//得到上面说的XmlWebApplicationContext
this.context = createWebApplicationContext(servletContext);
...略...
//
configureAndRefreshWebApplicationContext(cwac, servletContext);
...略...
```

进入configureAndRefreshWebApplicationContext
```Java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
  if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
    // The application context id is still set to its original default value
    // -> assign a more useful id based on available information
    //获取到web.xml中配置的context-param，我们的例子中就是applicationContext.xml，后续的bean定义主要就来源于此处。
    String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
    if (idParam != null) {
      wac.setId(idParam);
    }
    else {
      // Generate default id...
      wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
          ObjectUtils.getDisplayString(sc.getContextPath()));
    }
  }

  wac.setServletContext(sc);
  String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
  if (configLocationParam != null) {
    wac.setConfigLocation(configLocationParam);
  }

  // The wac environment's #initPropertySources will be called in any case when the context
  // is refreshed; do it eagerly here to ensure servlet property sources are in place for
  // use in any post-processing or initialization that occurs below prior to #refresh
  ConfigurableEnvironment env = wac.getEnvironment();
  if (env instanceof ConfigurableWebEnvironment) {
    ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
  }

  customizeContext(sc, wac);
  wac.refresh();
}
```

继续进入最后的wac.refresh();
```Java
@Override
public void refresh() throws BeansException, IllegalStateException {
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

首先从obtainFreshBeanFactory追踪下去会进入AbstractRefreshableApplicationContext的refreshBeanFactory方法：
```Java
/**
 * This implementation performs an actual refresh of this context's underlying
 * bean factory, shutting down the previous bean factory (if any) and
 * initializing a fresh bean factory for the next phase of the context's lifecycle.
 */
@Override
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
这里面得到了一个DefaultListableBeanFactory就是我们第一章节中说到的factory，也就是Spring的IoC容器，所谓IoC容器就是指生成Bean的Factory。

## 2.4 收集所有的BeanDefinition
追踪了这么久代码终于追踪到了一个小高潮，下面紧接着另一个loadBeanDefinitions(beanFactory);这个就是加载BeanDefinition的地方，也就是对Bean的定义。
```Java
/**
 * Loads the bean definitions via an XmlBeanDefinitionReader.
 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 * @see #initBeanDefinitionReader
 * @see #loadBeanDefinitions
 */
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
  // Create a new XmlBeanDefinitionReader for the given BeanFactory.
  XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

  // Configure the bean definition reader with this context's
  // resource loading environment.
  beanDefinitionReader.setEnvironment(getEnvironment());
  beanDefinitionReader.setResourceLoader(this);
  beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

  // Allow a subclass to provide custom initialization of the reader,
  // then proceed with actually loading the bean definitions.
  initBeanDefinitionReader(beanDefinitionReader);
  loadBeanDefinitions(beanDefinitionReader);
}
```
这地方不再详细展开进入代码了，其流程基本如下：
* 定义XmlBeanDefinitionReader类，以Resource形式读取Spring的xml配置文件(一个问题，java代码实现的配置@Configurable配置文件如果实现的？)
* 将XML文件转成DOM树，通过SAX方式来解析xml，得到所有的BeanDefinition,对xml文件的解析可以有默认的命名空间和自定义的命名空间
* 解析XML得到的所有BeanDefinition会通过DefaultListableBeanFactory的registerBeanDefinition方法放入到一个Map中，key为beanName，value为beanDefinition
上面几个步骤就完成了所有BeanDefinition的寻找和注册，然后下面就该轮到DI上场了，从而实现对Bean的依赖注入。

## 2.5 注入Bean

回到前面的wac.refresh()调用中来到finishBeanFactoryInitialization地方，里面会调用beanFactory.preInstantiateSingletons()这里完成了Bean的实例化及依赖注入(记住这里是PreInstantiate)。
```Java
@Override
public void preInstantiateSingletons() throws BeansException {
  if (this.logger.isDebugEnabled()) {
    this.logger.debug("Pre-instantiating singletons in " + this);
  }

  // Iterate over a copy to allow for init methods which in turn register new bean definitions.
  // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
  List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

  // Trigger initialization of all non-lazy singleton beans...
  for (String beanName : beanNames) {
    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
      if (isFactoryBean(beanName)) {
        final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
        boolean isEagerInit;
        if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
          isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
            @Override
            public Boolean run() {
              return ((SmartFactoryBean<?>) factory).isEagerInit();
            }
          }, getAccessControlContext());
        }
        else {
          isEagerInit = (factory instanceof SmartFactoryBean &&
              ((SmartFactoryBean<?>) factory).isEagerInit());
        }
        if (isEagerInit) {
          getBean(beanName);
        }
      }
      else {
        getBean(beanName);
      }
    }
  }

  // Trigger post-initialization callback for all applicable beans...
  for (String beanName : beanNames) {
    Object singletonInstance = getSingleton(beanName);
    if (singletonInstance instanceof SmartInitializingSingleton) {
      final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
      if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
          @Override
          public Object run() {
            smartSingleton.afterSingletonsInstantiated();
            return null;
          }
        }, getAccessControlContext());
      }
      else {
        smartSingleton.afterSingletonsInstantiated();
      }
    }
  }
}
```
因为Spring的强大，Bean不一定只是在xml文件中定义，也可以通过@Configurable来注解，也可以通过annotation driven自动发现@Service等等多种类型的Bean。还有一种FactoryBean(注意与BeanFactory区别)，是用来产生工厂类的Bean。

在DefaultListableBeanFactory的基类AbstractAutowireCapableBeanFactory中可以找到doCreateBean方法，其中会调用createBeanInstance、populateBean、initializeBean三个方法:
* createBeanInstance:根据BeanDefinition创建Bean实例，这是一个很复杂的过程，根据不同的策略能反射构造，能CGLIB...，另外还有AUTOWIRE_BY_NAME，AUTOWIRE_BY_TYPE区别等。
* populateBean:会深度递归搜索，设置Bean之间的依赖关系
* initializeBean:我们可以深入Bean生命周期的方法大都是这个方法中调用的，如BeanNameAware, initMethod等。

下面附一张Bean的生命周期图

![Spring的Bean的生命周期图](http://o9l56z0kf.bkt.clouddn.com/image/blog/spring-bean-life-cycle.png)

## 2.6 总结
第一部分抽象的说了下IoC的本质及实现方式，第二部分以Spring为例说明了Spring中的IoC实现，流程比较复杂啰嗦，但本质上是一致的：
1. Spring本身的Context建立起来之后开始构造出BeanFactory，也就是第一部分中提到的Factory类
2. 为每个Bean提供工厂类显然不现实，Spring构造出的BeanFactory通过收集Bean的BeanDefinition来统一为所有Bean提供工厂类，因此第二部分主要是收集Bean的Definition
3. 完成收集Bean的定义之后，工厂类就可以根据BeanDefinition类实例化Bean对象


# 三 参考文献
* [技术交流笔记——SPRING IOC 原理](http://winters1224.blog.51cto.com/3021203/1327881)
* [依赖倒置(DIP),控制反转(IoC)与依赖注入(DI)](http://openwares.net/java/dip_ioc_di.html)
* [Spring学习笔记之从web.xml看Ioc容器的加载](http://wiki.sankuai.com/pages/viewpage.action?pageId=449740666)
