---
title: [译]CGlib教程
date: 2017-06-30 13:56:04
tags: CGlib
---

[原文链接](https://github.com/cglib/cglib/wiki/Tutorial)

原始文章：[http://mydailyjava.blogspot.no/2013/11/cglib-missing-manual.html](http://mydailyjava.blogspot.no/2013/11/cglib-missing-manual.html)

## 增强器

让我们从`Enhancer`类开始，这可能是cglib库中最常被使用的类。一个增强器允许为没有接口的类型创建一个Java代理。`Enhancer`可以与java标准库的`Proxy`类做对比，后者是在Java 1.3时引入的。`Enhancer`动态地创建给定类型的子类而拦截所有的方法调用。与`Proxy`类不同的是，它对于类和接口都适用。接下来的例子和后续的一些例子都是基于这个简单的Java POJO：

```java
public class SampleClass {
  public String test(String input) {
    return "Hello world!";
  }
}
```

<!--more-->

使用cglib，可以很容易地利用`Enhancer`和`FixedValue`回调接口产生的值替换`test(String)`方法的返回值：

```java
@Test
public void testFixedValue() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new FixedValue() {
    @Override
    public Object loadObject() throws Exception {
      return "Hello cglib!";
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
}
```

在上述示例中，增强器会返回一个额外添加字节码的`SampleClass`子类(*an instrumented subclass of SampleClass*)的实例，对其所有的方法调用都会返回由上述匿名`FixedValue`实现生成的一个固定值。对象是由`Enhancer#create(Object...)`方法创建，此方法接收任意数量的参数以用于选择增强类中的任意一个构造器。(即使构造器仅仅是java字节码层面的方法，`Enhancer`也不能添加额外字节码到(*instrument*)构造器，而且它也不能添加额外字节码到(*instrument*) `static`或`final`的类。)如果你只想要创建一个类而不需要实例，`Enhancer#createClass`将创建一个可用于动态创建实例的`Class`实例。增强类中的所有构造器将以动态生成类的构造器作为委托供使用。

请注意，在上述例子中任意的方法调用都会被委托，也包括调用定义在`java.lang.Object`的方法。结果就是，调用`toString()`也将返回`"Hello cglib!"`。相反，对`proxy.hashCode()`的调用将会导致`ClassCastException`，这是因为`FixedValue`拦截器总是返回一个`String`，即使`Object#hashCode`签名提示了需要一个原始整型。

可以做出的的另一个观察是最终方法(*final method*)将不会被拦截。这种方法的一个例子是`Object#getClass`，当它被调用时会返回类似`SampleClass$$EnhancerByCGLIB$$e277c63c`的内容。这个类名是为了避免命名冲突而由cglib随机生成的。当你在程序代码里使用明确的类型时，请注意增强实例会有不同的类型。然而，由cglib生成的类会与被增强的类位于相同的包(因此能够覆盖包级私有方法)。同最终方法类似，子类化方法对增强最终类(*final class*)无能为力，因此像Hibernate这样的框架不能持久化最终类。

接下来，让我们看一个更强大的回调类，`InvocationHandler`，它也可以用于`Enhancer`：

```java
@Test
public void testInvocationHandler() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new InvocationHandler() {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable {
      if(method.getDeclaringClass() != Object.class && method.getReturnType() == String.class) {
        return "Hello cglib!";
      } else {
        throw new RuntimeException("Do not know what to do.");
      }
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
  assertNotEquals("Hello cglib!", proxy.toString());
}
```

这个回调接口允许我们对被调用的方法进行应答。然而，当随着`InvocationHandler#invoke`方法对代理对象调用方法时，你应该要小心。此方法的所有调用都会被分派到同一个`InvocationHandler`，可能因此导致无限循环。为了避免这种情况，我们可以另一个回调分派器：

```java
@Test
public void testMethodInterceptor() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy)
        throws Throwable {
      if(method.getDeclaringClass() != Object.class && method.getReturnType() == String.class) {
        return "Hello cglib!";
      } else {
        return proxy.invokeSuper(obj, args);
      }
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
  assertNotEquals("Hello cglib!", proxy.toString());
  proxy.hashCode(); // Does not throw an exception or result in an endless loop.
}
```

`MethodInterceptor`允许对被拦截的方法进行完全控制，并提供一些实用程序来调用处于原始状态的增强类的方法。既然如此，人们为什么还会想使用其他方法呢？因为其他方法更有效率，并且cglib经常用在效率起重要作用的极端情况框架(*edge case frameworks*)里。例如`MethodInterceptor`的创建和链接需要生成不同类型的字节代码，并创建一些`InvocationHandler`不需要的运行时对象。因此，还有其他类可以与`Enhancer`一同使用：

1.`LazyLoader`：尽管`LayzLoader`的唯一方法与`FixedValue`拥有相同的方法签名，但`LazyLoader`从根本上与`FiexedValue`拦截器是不同的。`LazyLoader`实际上应该返回一个增强类的子类实例。仅当调用增强类的方法时这个实例才会被请求，然后存储下来以供将来对已生成代理的调用。如果你的对象创建代价非常昂贵而且不知道是否会被使用，那么这么做是合理的。请注意，对于代理对象和懒加载对象，增强类的一些构造器都必须被调用。这样的话，需要确认有另一种代价较小的(可能是受保护的)构造器是可用的或者为代理使用接口类型。你可以通过为`Enhancer#create(Object...)`提供参数来选择被调用的构造器。

2.`Dispatcher`：`Dispatcher`和`LazyLoader`很类似但是它会在每个方法调用时被调用而不需要存储已加载对象。这允许更改类的实现而不用更改对它的引用。再次，请注意 ，对于代理对象和已生成对象，被增强类的一些构造器都必须被调用。

3.`ProxyRefDispatcher`：这个类在它的签名里携带了一个指向被调用的代理对象的引用。这样允许将一些方法调用委托给代理的另一个方法。请注意，这很容易引发无限循环，并且如果在`ProxyRefDispatcher#loadObject(Object)`里调用相同的方法，将总会引发无限循环。

4.`NoOp`：`NoOp`并不像它的名字所说的一样什么都不做。相反，它会把每个方法调用委托给被增强类的方法实现。 

到目前为止，最后两种拦截器可能对你没有意义。如果总是将方法委托给增强类，那么为什么还想着要增强一个类呢？你是对的，这些拦截器只能与`CallbackFilter`一起使用，就如下面代码片段所示：

```java
@Test
public void testCallbackFilter() throws Exception {
  Enhancer enhancer = new Enhancer();
  CallbackHelper callbackHelper = new CallbackHelper(SampleClass.class, new Class[0]) {
    @Override
    protected Object getCallback(Method method) {
      if(method.getDeclaringClass() != Object.class && method.getReturnType() == String.class) {
        return new FixedValue() {
          @Override
          public Object loadObject() throws Exception {
            return "Hello cglib!";
          };
        }
      } else {
        return NoOp.INSTANCE; // A singleton provided by NoOp.
      }
    }
  };
  enhancer.setSuperclass(MyClass.class);
  enhancer.setCallbackFilter(callbackHelper);
  enhancer.setCallbacks(callbackHelper.getCallbacks());
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
  assertNotEquals("Hello cglib!", proxy.toString());
  proxy.hashCode(); // Does not throw an exception or result in an endless loop.
}
```

`Enhancer`实例在它的`Enhancer#setCallbackFilter(CallbackFilter)`方法里接受一个`CallbackFilter`，并期望增强类的方法映射到`Callback`实例数组的数组索引。当在已创建的代理上调用一个方法时，`Enhancer`会选择相应的拦截器，并将被调用的方法分派给对应的`Callback`(这是至今为止所介绍的所有拦截器的标记接口)。为了使这个API不那么尴尬，cglib提供了一个`CallbackHelper`，它将代表一个`CallbackFilter` 并且会为你创建一个`Callback`的数组。上述增强对象将在功能上等同于`MethodInterceptor`例子中的对象，但是它允许你编写专门的拦截器，同时将调度逻辑与这些拦截器保持独立。

## 它是如何工作的?

当`Enhancer`创建一个类并在其创建之后，它将为注册为增强类的`Callback`的各个拦截器设置`private`字段。这意味着在由cglib创建的类定义在创建之后不能被重用，这是因为回调的注册没有成为已生成类的初始化阶段的一部分，而是当类已经被JVM初始化之后，由cglib手动准备。这也意味着由cglib创建的类在它们初始化之后从技术上讲并没有准备就绪，例如不能通过线路传送，因为在目标机器上加载的类不会存在回调接口。

依赖于已注册的拦截器，cglib可以注册额外的字段，例如像`MethodInterceptor`，在增强类或任何它的子类被拦截的每个方法都会注册两个私有静态字段(一个持有反射方法，另一个持有`MethodProxy`)。请注意，`MethodProxy`正在过度使用`FastClass`，其会触发额外的类创建并会在下面进一步描述。

```java
@Test
public void testFixedValue() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new FixedValue() {
    @Override
    public Object loadObject() throws Exception {
      return "Hello cglib!";
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
}
```

匿名的`FixValue`子类将很难从被增强的`SampleClass`引用，这样不论是匿名的`FixValue`实例还是持有@Test方法的类都不会被垃圾回收。这会在你的应用程序中引入令人讨厌的内容泄漏。因此，不要将cglib与非静态的内部类一起使用。(我仅在这个概述中这么使用是为了保持例子的简短)

最后，你不应该拦截`Object#finalize()`。由于cglib的子类化方式，拦截`finalize`是通过覆盖来实现，这通常是一个坏主意。拦截`finalize`的增强实例将会被垃圾收集器不同对待，也将会导致这些对象在JVM的终结队列(*finalization queue*)排队。另外，如果你在对被拦截的`finalize`调用中(意外地)创建了一个增强类的强引用(*a hard reference*)，那么你已经有效创建了一个不可回收的实例。这通常不是你想要的，因此需要注意最终方法永远不要被cglib拦截。`Object#wait`， `Object#notify` 和`Object#notifyAll`不会施加相同的问题。然而，请注意`Object#clone`可以被拦截可能是你不期望的事情。

## 不可变bean

cglib的`ImmutableBean`允许你创建一个类似于像`Collections#immutableSet`的不可变的包装器。底层bean的所有更改都会被`IllegalStateException`阻止(然而，并不是被Java API所推荐的`UnsupportedOperationException`)。看一下bean

```java
public class SampleBean {
  private String value;
  public String getValue() {
    return value;
  }
  public void setValue(String value) {
    this.value = value;
  }
}
```

我们可以使得这个bean不可变：

```java
@Test(expected = IllegalStateException.class)
public void testImmutableBean() throws Exception {
  SampleBean bean = new SampleBean();
  bean.setValue("Hello world!");
  SampleBean immutableBean = (SampleBean) ImmutableBean.create(bean);
  assertEquals("Hello world!", immutableBean.getValue());
  bean.setValue("Hello world, again!");
  assertEquals("Hello world, again!", immutableBean.getValue());
  immutableBean.setValue("Hello cglib!"); // Causes exception.
}
```

从示例中可以看出，不可变的bean通过抛出`IllegalStateException`来阻止所有的状态更改。然而，bean的状态可以通过更改原始对象而改变。所有这些变化都将由`ImmutableBean`反映出来。

## Bean生成器

`BeanGenerator`是cgbli的另一个bean实用工具。它会在运行时为你创建一个bean：

```java
@Test
public void testBeanGenerator() throws Exception {
  BeanGenerator beanGenerator = new BeanGenerator();
  beanGenerator.addProperty("value", String.class);
  Object myBean = beanGenerator.create();

  Method setter = myBean.getClass().getMethod("setValue", String.class);
  setter.invoke(myBean, "Hello cglib!");
  Method getter = myBean.getClass().getMethod("getValue");
  assertEquals("Hello cglib!", getter.invoke(myBean));
}
```

从示例中可以看出，`BeanGenerator`首先将一些属性作为名值对。在创建的时候，`BeanGenerator`为你创建访问器

```java
<type> get<name>()
void set<name>(<type>)
```

当另一个库希望通过反射解析bean，但在运行时你不知道这些bean，这是非常有用的。(一个例子是Apache Wicket，它有很多bean相关的工作)

## Bean复制器

`BeanCopier`是另一个实用工具，它通过属性值复制bean。考虑另一个与`SampleBean`有相似属性的bean：

```java
public class OtherSampleBean {
  private String value;
  public String getValue() {
    return value;
  }
  public void setValue(String value) {
    this.value = value;
  }
}
```

现在你可以把属性从一个bean复制到另一个bean而不受特定类型的限定：

```java
@Test
public void testBeanCopier() throws Exception {
  BeanCopier copier = BeanCopier.create(SampleBean.class, OtherSampleBean.class, false);
  SampleBean bean = new SampleBean();
  bean.setValue("Hello cglib!");
  OtherSampleBean otherBean = new OtherSampleBean();
  copier.copy(bean, otherBean, null);
  assertEquals("Hello cglib!", otherBean.getValue()); 
}
```

`BeanCopier#copy`调用一个(最终)可选的转换器，它允许在每个bean属性上做一些进一步操作。如果`BeanCopier`把第三个构造器参数设置为false而创建的，`Converter`会被忽略并且可以为空。

## 批量bean

`BulkBean`允许通过数组而不是方法调用的方式来使用一组指定的bean访问器：

```java
@Test
public void testBulkBean() throws Exception {
  BulkBean bulkBean = BulkBean.create(SampleBean.class,
      new String[]{"getValue"},
      new String[]{"setValue"},
      new Class[]{String.class});
  SampleBean bean = new SampleBean();
  bean.setValue("Hello world!");
  assertEquals(1, bulkBean.getPropertyValues(bean).length);
  assertEquals("Hello world!", bulkBean.getPropertyValues(bean)[0]);
  bulkBean.setPropertyValues(bean, new Object[] {"Hello cglib!"});
  assertEquals("Hello cglib!", bean.getValue());
}
```

`BulkBean`将一组getter名称、一组setter名称以及一组属性的类型作为构造器参数。最终生成的添加额外字节码的类(*the resulting instrumented class*)可以由`BulkBean#getPropertyValues(Object)`抽取为一个数组。类似的，一个bean的属性可以通过`BulkBean#setPropertyValues(Object, Object[])`设置。

## Bean映射

这是cglib最后一个bean的实用工具。`BeanMap`将bean所有的属性都转换成一个字符-对象映射的Java `Map`：

```java
@Test
public void testBeanGenerator() throws Exception {
  SampleBean bean = new SampleBean();
  BeanMap map = BeanMap.create(bean);
  bean.setValue("Hello cglib!");
  assertEquals("Hello cglib", map.get("value"));
}
```

另外，`BeanMap#newInstance(Object)`方法允许重用相同的`Class`来为其他bean创建映射。

## Key工厂

`KeyFactory`工厂允许动态创建由多个值组成的键，可以用在如`Map`这样的实现。为了做到这一点，`KeyFactory`需要接口来定义应该在这样的键中使用的值。这个接口必须包含名称为`newInstance`并且返回一个`Object`的单个方法。例如：

```java
public interface SampleKeyFactory {
  Object newInstance(String first, int second);
}
```

现在一个key的实例可以被这样创建：

```java
@Test
public void testKeyFactory() throws Exception {
  //Key.class --> SampleKeyFactory.class
  SampleKeyFactory keyFactory = (SampleKeyFactory) KeyFactory.create(Key.class);
  Object key = keyFactory.newInstance("foo", 42);
  Map<Object, String> map = new HashMap<Object, String>();
  map.put(key, "Hello cglib!");
  assertEquals("Hello cglib!", map.get(keyFactory.newInstance("foo", 42)));
}
```

`KeyFactory`将会确保`Object#equals(Object)`和`Object#hashCode`方法的正确实现，以便最终的键对象能被`Map`或者`Set`使用。`KeyFactory`也在cglib库内部使用了很多。

## 混合

有些人可能已经从其他编程语言如Ruby或者Scala(其中mixins被叫做traits)知道了`Mixin`类的概念。cglib的`Mixin`允许将几个对象组合成一个对象。然而，为了这样做，这些对象必须由接口支持：

```java
public interface Interface1 {
  String first();
}

public interface Interface2 {
  String second();
}

public class Class1 implements Interface1 {
  @Override
  public String first() {
    return "first";
  }
}

public class Class2 implements Interface2 {
  @Override
  public String second() {
    return "second";
  }
}
```

现在类`Class1`和`Class2`可以通过额外的接口组合成一个单独的类：

```java
public interface MixinInterface extends Interface1, Interface2 { /* empty */ }

@Test
public void testMixin() throws Exception {
  Mixin mixin = Mixin.create(new Class[]{Interface1.class, Interface2.class,
      MixinInterface.class}, new Object[]{new Class1(), new Class2()});
  MixinInterface mixinDelegate = (MixinInterface) mixin;
  assertEquals("first", mixinDelegate.first());
  assertEquals("second", mixinDelegate.second());
}
```

诚然，`Mixin` API是相当尴尬的，因为它需要用于混合的类来实现一些接口，但这样一来问题也可以通过不添加额外字节码(*non-instrumented*)的Java来解决。

## 字符串切换器

`StringSwitcher`模拟`String`到`int`的Java映射：

```java
@Test
public void testStringSwitcher() throws Exception {
  String[] strings = new String[]{"one", "two"};
  int[] values = new int[]{10, 20};
  StringSwitcher stringSwitcher = StringSwitcher.create(strings, values, true);
  assertEquals(10, stringSwitcher.intValue("one"));
  assertEquals(20, stringSwitcher.intValue("two"));
  assertEquals(-1, stringSwitcher.intValue("three"));
}
```

`StringSwitcher`允许在`String`上模拟一个switch命令就像从Java 7开始可以使用的内置的Java switch语句。如果在Java 6或更低版本中使用`StringSwitcher`或许仍然能给你的代码增加一些好处，然而我个人不会推荐使用它。

## 接口制造者

`InterfaceMaker`就如它名称所说：它能够动态创建一个新接口。

```java
@Test
public void testInterfaceMaker() throws Exception {
  Signature signature = new Signature("foo", Type.DOUBLE_TYPE, new Type[]{Type.INT_TYPE});
  InterfaceMaker interfaceMaker = new InterfaceMaker();
  interfaceMaker.add(signature, new Type[0]);
  Class iface = interfaceMaker.create();
  assertEquals(1, iface.getMethods().length);
  assertEquals("foo", iface.getMethods()[0].getName());
  assertEquals(double.class, iface.getMethods()[0].getReturnType());
}
```

与其他cglib类的公共API不同，`InterfaceMaker`依赖于ASM类型。在运行的应用程序中创建接口几乎没有意义，因为接口仅代表一种通过编译器检查的可以使用的类型。然而当你生成将在后续开发中使用的代码时，它是有意义的。

## 方法委托

`MethodDelegate`允许通过将一个方法调用绑定到某个接口来模拟一个特定方法的类似c#的委托。例如，下面的代码将`SampleBean#getValue`绑定到一个委托： 

```java
public interface BeanDelegate {
  String getValueFromDelegate();
}

@Test
public void testMethodDelegate() throws Exception {
  SampleBean bean = new SampleBean();
  bean.setValue("Hello cglib!");
  BeanDelegate delegate = (BeanDelegate) MethodDelegate.create(
      bean, "getValue", BeanDelegate.class);
  assertEquals("Hello world!", delegate.getValueFromDelegate());
}
```

然而有一些事情需要注意：

1. 工厂方法`MethodDelegate#create`只需要一个方法名称，并作为其第二个参数。这就是`MethodDelegate`将会为你代理的方法。
2. 作为传给工厂方法的第一个参数的对象必须要有一个无参方法。因此，`MethodDelegate`没有那么强大。
3. 第三个参数必须是只有一个参数的接口。`MethodDelegate`实现了它，并且可以转换成它。当方法被调用的时候，它会调用第一个参数对象上被代理的方法。

此外，要考虑到这些缺点：

1. cglib为每一个代理创建一个新类。最终，这会浪费你的永久代的堆空间。
2. 你不能代理包含参数的方法。
3. 如果你的接口接受参数，方法委托将无法工作，但不抛出异常(返回值始终为空)。如果你的接口需要另一种返回类型(即使是更通用的)，那么你将得到一个`IllegalArgumentException`。 

## 广播委托

`MulticastDelegate`与`MethodDelegate`有所不同，即使它的目的是提供类似功能。为了使用`MulticastDelegate`，我们需要实现一个接口的对象：

```java
public interface DelegatationProvider {
  void setValue(String value);
}

public class SimpleMulticastBean implements DelegatationProvider {
  private String value;
  public String getValue() {
    return value;
  }
  public void setValue(String value) {
    this.value = value;
  }
}
```

基于这个接口支持的bean，我们可以创建一个`MulticastDelegate`来将所有对`setValue(String)`的调用分派给实现了`DelegationProvider`接口的几个类：

```java
@Test
public void testMulticastDelegate() throws Exception {
  MulticastDelegate multicastDelegate = MulticastDelegate.create(
      DelegatationProvider.class);
  SimpleMulticastBean first = new SimpleMulticastBean();
  SimpleMulticastBean second = new SimpleMulticastBean();
  multicastDelegate = multicastDelegate.add(first);
  multicastDelegate = multicastDelegate.add(second);

  DelegatationProvider provider = (DelegatationProvider)multicastDelegate;
  provider.setValue("Hello world!");

  assertEquals("Hello world!", first.getValue());
  assertEquals("Hello world!", second.getValue());
}
```

再次，有一些缺点：

1. 对象需要实现一个单一方法的接口。这一点对第三方库来说糟透了，并且当你使用CGlib做了一些不可思议的事情并将其暴露给正常代码时会非常尴尬。此外，你也可以轻松实现你自己的代理(而不用修改字节代码，但是我怀疑采用人工委托的方式是否有更大优势)
2. 当你的代理返回一个值时，你将只接收到最后添加的代理的返回值，所有其他的返回值都丢失了(但是会在某个点被`MulticastDelegate`接收)。

## 构造器委托

`ConstructorDelegate`允许创建一个字节码编排过的(*byte-instrumented*)工厂方法。为此，我们首先需要一个带有单个方法newInstance且返回一个Object的接口，并且接收任意数量的参数以用于指定类的构造器调用。例如，为了创建`SampleBean`的一个`ConstructorDelegate` ，我们需要以下代码来调用`SampleBean`的默认(无参)构造器：

```java
public interface SampleBeanConstructorDelegate {
  Object newInstance();
}

@Test
public void testConstructorDelegate() throws Exception {
  SampleBeanConstructorDelegate constructorDelegate = (SampleBeanConstructorDelegate) ConstructorDelegate.create(
    SampleBean.class, SampleBeanConstructorDelegate.class);
  SampleBean bean = (SampleBean) constructorDelegate.newInstance();
  assertTrue(SampleBean.class.isAssignableFrom(bean.getClass()));
}
```

## 并行排序器

`ParallelSorter`声称当对数组构成的数组进行排序时，是Java标准库的数组排序器更快的替代：

```java
@Test
public void testParallelSorter() throws Exception {
  Integer[][] value = {
    {4, 3, 9, 0},
    {2, 1, 6, 0}
  };
  ParallelSorter.create(value).mergeSort(0);
  for(Integer[] row : value) {
    int former = -1;
    for(int val : row) {
      assertTrue(former < val);
      former = val;
    }
  }
}
```

`ParallelSorter`接收一组数组并且允许对数组的每一行运用归并排序或者快速排序。然而当你使用它的时候需要小心：

1. 当使用基本类型的数组，你必须使用明确的排序范围来调用归并排序(例如，在例子里是`ParallelSorter.create(value).mergeSort(0, 0, 3)`)。否则，`ParallelSorter`有一个非常明显的错误，它会尝试将基本类型的数组转换成Object[]的数组，这会导致`ClassCastException`。
2. 如果数组的行是不均匀的，那么第一个参数将决定需要考虑的行的长度。不均匀的行将会导致额外的值不会被排序考虑或者导致`ArrayIndexOutOfBoundException`。

就个人而言，我疑惑`ParallelSorter`是否真的提供了时间上的优势。诚然，我没有尝试对其进行基准测试。如果你尝试了，我很高兴可以在评论中听到。

## 快速类和快速成员

`FastClass`通过包装一个Java类承诺提供比java反射API更快的方法调用并提供类似于反射API的方法：

```java
@Test
public void testFastClass() throws Exception {
  FastClass fastClass = FastClass.create(SampleBean.class);
  FastMethod fastMethod = fastClass.getMethod(SampleBean.class.getMethod("getValue"));
  MyBean myBean = new MyBean();
  myBean.setValue("Hello cglib!");
  assertTrue("Hello cglib!", fastMethod.invoke(myBean, new Object[0]));
}
```

除了演示的`FastMethod`，`FastClass`还可以创建`FastConstructor`，不过不能创建快速字段(*fast field*)。但是`FastClass`是如何比正常反射更快的？Java反射是由**JNI**通过调用一些C代码来执行方法调用。另一方面，`FastClass`创建了一些可以直接在JVM内调用方法的字节码。然而，较新版本的HotSpot JVM(以及可能许多其他现代JVM)知道了一种膨胀(*inflation*)的概念，当一个反射方法被执行足够频繁，JVM会将本地方法调用转换成本机版本的`FastClass`。你甚至可以通过将`sun.reflect.inflationThreshold`属性(默认值是15)设置为一个较小的值来控制这个行为(至少是在HotSpot JVM)。该属性决定了经过多少次反射调用一个JNI调用可以被一个额外添加字节码(*byte code instrumented*)的版本所代替。因此，我建议不要在现代JVM上使用`FastClass`，然而在较旧版本的Java虚拟机可以微调性能。