---
categories:
  - Technology
tags:
  - Spring
---
# 一 Junit
## 1.1 JUnit简介
单元测试，白盒测试  
JUnit的设计使用以Patterns Generate Architectures的方式来架构系统。其设计思想是通过从零开始来应用设计模式，然后一个接一个，直至你获得最终合适的系统架构。

## 1.2 JUnit简析

### 1.2.1 JUnitCore
[JUnitCore源码分析](http://blog.csdn.net/neven7/article/details/46675845)参考上面链接中的JUnitCore源码时序图

@RunWith中配置的值就是这里取到的。

重要的一段代码：

```java
public Result run(Runner runner) {
    Result result = new Result();
    RunListener listener = result.createListener();
    notifier.addFirstListener(listener);
    try {
        notifier.fireTestRunStarted(runner.getDescription());
        runner.run(notifier);
        notifier.fireTestRunFinished(result);
    } finally {
        removeListener(listener);
    }
    return result;
}
```

### 1.2.2 RunNotifier
通知注册到其中的Listener执行相关的操作。比如：单元测试的结果输出到终端上；IDE里面的测试执行状态的改变，成功或者失败的标识等等

### 1.2.3 TestListener
```java
public interface TestListener {
    public void addError(Test test, Throwable e);
    public void addFailure(Test test, AssertionFailedError e);
    public void endTest(Test test);
    public void startTest(Test test);
}
```

### 1.2.4 BlockJUnit4ClassRunner

[默认执行器 blockjunit4classrunner 源码分析](https://testerhome.com/topics/3427)

JUnitCore使用了Facade外观模式，内部可以选择不同的单元测试的执行者，Junit4默认提供的就是BlockJUnit4ClassRunner，SpringTest也是继承BlockJUnit4ClassRunner来实现的。

### 1.2.5 Statement
装饰模式？链表？

### 1.2.6 执行过程run

class层面：
* 获取Statement
>classBlock
>1. Statement statement = childrenInvoker(notifier);首先组织一个会调用(runChildren)所有@Test标注方法进行测试的测试单元的Statement
>2. 组织好Statement，也就是附加@BeforeClass和@AfterClass等方法

* statement.evaluate();

method层面，runChild：
* 获取Statement
>methodBlock
>1. 实例化对象（比如例化SpringTest时这一步添加DI等相关工作）
>2. 组织好Statement，也就是附加@Before和@After等方法

* RunLeaf
```java
RunNotifier.fireTestStarted()
statement.evaluate();
RunNotifier.fireTestFinished()
```

## 1.3 支持的注解
* @RunWith
 >用来说明此测试类的运行者，JUnit4中如果不指定默认使用的是BlockJUnit4ClassRunner,Spring中是SpringJUnit4ClassRunner
* @Before
* @After
* @BeforeClass
* @AfterClass
* @Test
  >@Test(expected=XXXException.class)
  @Test(timeout=...)
* @Ignore
* @Repeat


## 1.4 JUnit中的设计模式
* Facade外观模式
* Decorator装饰模式
* 责任链模式
* Command命令模式
>将请求封装成对象，是回调函数的面向对象版本
>Test接口

* Adapter模式
* Composite组合模式
>TestSuit
>Template Method模板方法

* 观察者模式


# 二 SpringTest

## 2.1 一个例子

```java
@Transactional
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:app.xml")
public abstract class AbstractTest extends AbstractTransactionalJUnit4SpringContextTests {
    @Resource
    private Cache cache;
    @Resource
    private IMedis medis;

    @BeforeClass
    public static void before(){
        System.out.println("beforeClass");
    }

    @AfterClass
    public static void after(){
        System.out.println("afterClass");
    }

    @After
    public void cleanCache() {
        cache.clear();
        medis.flushAll();
    }
}
```

```java
public class TestImpl extends AbstractTest {

    @BeforeClass
    public static void before1(){
        System.out.println("beforeClass1");
    }

    @AfterClass
    public static void after1(){
        System.out.println("afterClass1");
    }
    @Test
    public void test() {
        System.out.print("test");
    }

    @Test
    public void test2() {
        System.out.print("test2");
    }
}
```

## 2.2 如何从JUnit4进入SpringTest

@RunWith

## 2.3 SpringJUnit4ClassRunner

SpringJUnit4ClassRunner是对BlockJUnit4ClassRunner进行封装，比如对@IfProfileValue注解的支持

## 2.3.1 执行流程

先看前面提到的methodBlock方法在SpringJUnit4ClassRunner中的实现：
```java
protected Statement methodBlock(FrameworkMethod frameworkMethod) {
  Object testInstance;
  try {
    testInstance = new ReflectiveCallable() {
      @Override
      protected Object runReflectiveCall() throws Throwable {
        return createTest();
      }
    }.run();
  }
  catch (Throwable ex) {
    return new Fail(ex);
  }

  Statement statement = methodInvoker(frameworkMethod, testInstance);
  statement = possiblyExpectingExceptions(frameworkMethod, testInstance, statement);
  statement = withBefores(frameworkMethod, testInstance, statement);
  statement = withAfters(frameworkMethod, testInstance, statement);
  statement = withRulesReflectively(frameworkMethod, testInstance, statement);
  statement = withPotentialRepeat(frameworkMethod, testInstance, statement);
  statement = withPotentialTimeout(frameworkMethod, testInstance, statement);
  return statement;
}
```

其中的createTest方法中建立起来了Spring的TestContextManager，关于TestContextManager后面会有更多的介绍

```java
@Override
protected Object createTest() throws Exception {
  Object testInstance = super.createTest();
  getTestContextManager().prepareTestInstance(testInstance);
  return testInstance;
}
```

## 2.3.2 几种Statement
- RunPrepareTestInstanceCallbacks
- RunBeforeTestClassCallbacks
- RunBeforeTestMethodCallbacks：在JUnit的Before方法执行前执行
- RunAfterTestMethodCallbacks
- RunAfterTestClassCallbacks

## 2.4 TestContextManager

主要包含下面两个成员对象，操作都是移交给testExecutionListeners来执行的。
- private final TestContext testContext;
- private final List<TestExecutionListener> testExecutionListeners = new ArrayList<TestExecutionListener>();

## 2.5 TestExecutionListener

TestExecutionListener 接口包括下面几种方法：
- void beforeTestClass(TestContext testContext) throws Exception;
- void prepareTestInstance(TestContext testContext) throws Exception;
- void beforeTestMethod(TestContext testContext) throws Exception;
- void afterTestMethod(TestContext testContext) throws Exception;
- void afterTestClass(TestContext testContext) throws Exception;

Spring提供了6种默认的TestExecutionListener：
* ServletTestExecutionListener: configures Servlet API mocks for a WebApplicationContext
* DirtiesContextBeforeModesTestExecutionListener: handles the @DirtiesContext annotation for before modes
* DependencyInjectionTestExecutionListener: provides dependency injection for the test instance
* DirtiesContextTestExecutionListener: handles the @DirtiesContext annotation for after modes
* TransactionalTestExecutionListener: provides transactional test execution with default rollback semantics
* SqlScriptsTestExecutionListener: executes SQL scripts configured via the @Sql annotation

SpringTest依赖注入的实现就是在DependencyInjectionTestExecutionListener中完成的：
```java
protected void injectDependencies(final TestContext testContext) throws Exception {
  Object bean = testContext.getTestInstance();
  AutowireCapableBeanFactory beanFactory = testContext.getApplicationContext().getAutowireCapableBeanFactory();
  beanFactory.autowireBeanProperties(bean, AutowireCapableBeanFactory.AUTOWIRE_NO, false);
  beanFactory.initializeBean(bean, testContext.getTestClass().getName());
  testContext.removeAttribute(REINJECT_DEPENDENCIES_ATTRIBUTE);
}
```


## 2.6 支持的注解
* @ActiveProfiles("test")：用来配置不同的环境，SpringJUnit4ClassRunner中进行过滤
* @BeforeTransaction
* @AfterTransaction

# 三 一个完整的流程

```
环境IDE: idea 15
程序入口：com.intellij.rt.execution.junit.JUnitStarter
SpringJUnit4ClassRunner
负责创建TestContextManager
……
```