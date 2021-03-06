### 语言特性

#### 反射

反射机制是 Java 语言提供的一种基础功能，赋予程序在运行时自省（introspect）的能力，通过反射可以直接操作类或对象（获取某个对象的类定义，获取类声明的属性和方法，调用方法或构造对象，在运行时修改类定义）

##### 反射机制

*java.lang.Class* 类与 *java.lang.reflect* 类库（*Filed*、*Method*、*Constructor*）支撑反射（*Class* 类提供方法获取 *Filed*/*Method*/*Constructor* 对象），这些类型的对象是由 JVM 在运行时创建的，用以表示类对应的成员，匿名对象的类信息能在运行时被完全确定下来，而在编译时不需要知道任何细节

当通过反射与一个未知类型的对象打交道时，JVM 只是简单地检查这个对象，看它属于那个特定的类，在用它做其他事情之前必须先加载那个类的 Class 对象。那个类的 `.class` 文件对于 JVM 来说必须是可获取的，要么在本地机器上，要么可以通过网络取得。对于反射机制来说，.class 文件在编译时可以不获取，在运行时打开和检查 .class 文件

反射在 Java 中用来支持其他特性（对象序列化和 JavaBean）。反射提供的 `AccessibleObject.setAccessible(boolean flag)` 可以在运行时修改成员访问限制（即使类中使用 private 修饰）。

9 之后，新增的模块化系统，出于强封装性的考虑，对反射访问进行了限制（只有当被反射操作的模块和指定的包对反射调用者模块 Open，才能使用 `setAccessible`，否则被认为不合法操作）如果我们的实体类是定义在模块里面，需要在模块描述符中明确声明：

```java
module MyEntities {
    // Open for reflection
    opens com.mycorp to java.persistence;
}
```

Java 9 仍然保留了兼容 Java 8 的行为

##### 动态代理

*   代理

    是基本的设计模式，插入了用来代替实际的对象提供额外的或不同的操作。这些操作涉及与实际对象的通信，代理通常充当中间人角色。通过代理可以让调用者与实现者之间解耦。

*   动态代理

    可以动态的创建代理并动态地处理对所代理方法的调用。在动态代理上所做的所有调用都会被重定向到单一的调用处理器上（负责根据调用类型确定调用策略）

###### JDK 动态代理

基于反射机制实现，通过调用静态方法 `Proxy.newProxyInstance` 创建动态代理。

动态代理可以将所有调用重定向到 *InvocationHandler* 调用处理器，通常会向调用处理器的构造器传递一个实际对象的引用，从而使得调用处理器在执行其中介任务时，可以将请求转发

`InvocationHandler.invoke()` 方法中传递了代理对象，在 invoke 内部，在代理上调用方法时对接口的调用将被重定向为对代理的调用

```java
// 接口
public interface Hello {
	void sayHello();
}
public class DynamicProxyDemo {
    Hello dynamicHello = (Hello) Proxy.newProxyInstance(
            Hello.class.getClassLoader(),
            new Class[] { Hello.class },
            (proxy, method, args1) -> {
                System.out.println("lambda invocationHandler");
                // 对接口实现替换为 lambda
                return method.invoke((Hello) () -> {
                    System.out.println("lambda provider say");
                }, args1);
            }
    );
    dynamicHello.sayHello();
}
```

基于接口实现的局限在于：以接口为中心，要先定义接口，实例化的是 Proxy 对象，而不是真正的调用类型，实践中会有一定的不便和能力退化；优势在于：最小化依赖关系，开发和实现简单，JDK 本身的支持

###### 基于 cglib 字节码实现

cglib 动态代理采取的是创建目标类的子类方式，因为是子类化，可以达到近似使用被调用者本身的效果。优势在于：去除接口依赖，只操作关注的类，而不必为其他相关类增加工作量，高性能。

##### ref 库

*java.lang.ref* 库包含一组类，这些类为垃圾回收提供了暗示。当垃圾回收器正在考察的对象只能通过某个 *Reference* 对象才可获得时，会使用以下类进行可达性分析

###### 垃圾收集可达性

对象可在程序中的某处找到，如果一个对象是可达的，垃圾回收器就不能释放它，否则垃圾回收器可将其释放，对于普通对象，如果没有其他引用关系，只要超过了引用的作用域或将强引用赋值为 null，就可以被垃圾收集器收集，具体回收时机取决于垃圾收集策略）

*   强可达（Strongly Reachable）

    当一个对象可以被一个或多个线程在不通过各种引用访问到的情况（新创建一个对象，创建它的线程对它就是强可达）

*   软可达（Softly Reachable）

    只能通过软引用才能访问到对象的状态

*   弱可达（Weakly Reachable）

    只能通过弱引用访问时的状态。此时临近 finalize 状态的时机，当弱引用被清除的时候，就符合 finalize 的条件了

*   幻象可达（Phantom Reachable）

    已经 finalize 过了，只有幻象引用指向这个对象的时候

*   不可达（unreachable）

    此时对象可以被垃圾收集器回收

判断对象可达性，是垃圾收集器决定如何处理对象的一部分考虑

###### 引用对象

所有引用类型，都是抽象类 `java.lang.ref.Reference` 的子类，除了幻象引用，如果对象还没有被销毁，都可以通过 get 方法获取原有对象并重新指向强引用，因此对于软引用、弱引用、垃圾收集器可能会存在二次确认的问题，以保证处于弱引用状态的对象，没有改变为强引用。

*   强引用（Strong Reference）

    普通对象引用，只要还有强引用指向一个对象，就表明对象还存活，垃圾收集器不会收集该对象。

*   软引用（*SoftReference*）

    可以让对象豁免一些垃圾收集。只有当 JVM 认为内存不足时，才会回收软引用指向的对象，JVM 会确保在抛出 *OutOfMemoryError* 之前，清理软引用指向的对象。
    通常用来实现内存敏感的缓存，如果还有空闲内存，就可以暂时保留缓存，当内存不足时清理掉。

    软引用通常会在最后一次引用后，还能保持一段时间，默认值是根据堆剩余空间计算的（以 M bytes）从 Java 1.3.1 开始，提供了

    ```shell
    # 以毫秒为单位设置保持时间
    --XX:SoftRefLRUPolicyMSPerMB=3000
    ```

*   弱引用（*WeakReference*）

    是为了实现规范映射（canonicalizing mappings）而设计的，不妨碍垃圾回收器回收映射的键或值，仅仅提供一种访问在弱引用状态下对象的途径。
    可以用来维护一种非强制性的映射关系，如果试图获取时对象还在，就使用它，否则重新实例化。同样是很多缓存实现的选择

*   幻象引用（*PhantomReference*）

    不能通过它访问对象。通常用以调度回收前的清理工作，比 Java 终止机制更灵活。不能通过它访问对象(get 方法只返回 null)。仅仅提供了一种确保对象被 finalize 以后做某些事情的机制。通常用来做 Post-Mortem 清理机制。也可以利用幻象引用监控对象的创建和销毁

###### 引用编程

当使用各种引用进行编程时，在创建各种引用并关联到相应对象时，可以选择是否需要关联引用队列（*ReferenceQueue*），虚拟机会在特定时机将引用 enqueue 到队列里，可以从队列里获取引用进行后续逻辑（尤其是幻象引用，get 方法只返回 null，如果不指定引用队列，则没有意义）

```java
<T> void example(T obj) {
    ReferenceQueue<T> referenceQueue = new ReferenceQueue<>();
    PhantomReference<T> p = new PhantomReference<>(obj, referenceQueue);
    obj = null;
    System.gc();
    try {
    	Reference<T> reference = (Reference<T>) referenceQueue.remove();
    	if (reference != null) {
    		System.out.println(reference.get());
    	}
    } catch (InterruptedException e) {
    	e.printStackTrace();
    }
}
```

9 开始可以通过底层 API 来达到强引用的效果，有时，对象本身没有强引用，但它的部分属性还在被使用，如各种异步调用

```java
class Resource {
    private int myIndex;
    private static ExternalResource[] externalResourceArray;
    public Resource(int i) {
        myIndex = i;
        externalResourceArray[myIndex] = i;
    }
    public void action() {
        try {
            int i = myIndex;
            Resource.update(externalResourceArray[i]);
        } finally {
            // finally 中明确声明对象强可达
            Reference.reachabilityFence(this);
        }
    }
    private static void update(ExternalResource exter) {
        ext.status = ...;
    }
}
new Resource().action();
```

#### 异常

在程序运行过程中，如果虚拟机检测出一个不可执行的操作，就会出现运行时错误 (runtime error)，运行时错误会作为异常（一种对象，表示阻止正常程序正常执行的错误或情况，异常对象在堆上创建）抛出，如果异常没有被处理，那么程序就会非正常终止。异常从方法抛出，方法的调用者可以捕获以及处理该异常，使用 throw 语句抛出一个异常。

##### 异常类型

异常对象派生于 *java.lang.Throwable*，所有的异常都直接或间接继承该类。对于受检异常，Java会强制要求处理，否则无法编译。如果方法没有声明（使用 throws 声明所有异常类，用逗号分隔）所有可能发生的受检异常，编译无法通过。如果在子类中覆盖了超类的一个方法，子类方法中声明的受检异常不能比超类方法中声明的异常更通用。如果超类方法没有抛出任何受检异常，子类也不能抛出任何异常。

###### 非受检异常

派生于 *Error* 或 *RuntimeException* 的所有异常。未受检异常表示编程的逻辑错误（空指针），编程时应该检查以避免这些错误。

* *Error* 

  类层次结构描述了运行时系统的内部错误和资源耗尽错误。应用程序不应该抛出这种类型的对象。如果发生，除了通知用户以及尽量稳妥的终止程序外，几乎什么都不能做

  *场景错误*

  |          类型          |                             说明                             |
  | :--------------------: | :----------------------------------------------------------: |
  | *NoClassDefFoundError* | 运行时错误：运行时系统尝试加载类定义时引发的错误，并且该类定义不再可用，所需的类定义在编译时存在，但在运行时丢失（缺少对应 .class 文件） |
  |                        |                                                              |
  |                        |                                                              |

*   *RuntimeException*

    由程序错误导致的异常，会自动被虚拟机抛出，如果没有被捕获而直到 main()，在程序退出前将调用 printStackTrace()。该异常发生在运行时，代表编程错误，编译器无法捕获该异常及其子异常其他类型异常的处理都是由编译器强制实施的
    
    *常见异常*
    
    |               异常                |                             说明                             |
    | :-------------------------------: | :----------------------------------------------------------: |
    |      *NullPointerException*       |                          空指针异常                          |
    |      *NumberFormatException*      |                         数字格式错误                         |
    |      *IllegalStateException*      |                           非法状态                           |
    |    *IndexOutOfBoundsException*    |                           索引越界                           |
    |       *ClassCastException*        |                         类型转换错误                         |
    | *ArrayIndexOutOfBoundsException*  |                         数组索引越界                         |
    |    *IllegalArgumentException*     |                           非法参数                           |
    | *StringIndexOutOfBoundsException* |                        字符串索引越界                        |
    |     *ClassNotFoundException*      | 运行时异常：执行：class.forName()、loadClass()、findSystemclass()、反序列化时在运行时加载或验证类，并且在类路径中找不到具有指定名称的类时抛出 |

###### 受检异常

受检异常必须出现在 throws 语句中，调用者必须处理。Java 编译器会强制检查。表示程序本身没问题，但由于 I/O、网络、数据库等其他错误导致的异常，调用者应该适当处理

* *Exception*

  应用程序应该预料到并从中恢复的异常。基本上是所有派生自 Exception 的子类，但不是 *RuntimeException* 的子类（如 *IOException*、*SQLException*）。编译器可以通过拒绝编译任何包含可能引发已检查异常的程序来防止发生已检异常，一个编写良好的 Java 库应向它的用户指出方法所有可能抛出的异常

##### 异常处理

异常处理目标分为恢复和报告，恢复（通过程序自动处理），报告的目标可能是用户也可能是运维、程序员。报告的目的是为了恢复（常需要人参与）

* 用户

  输入错误，提示用户那里输入错误

* 程序员

  编程错误，提供用户系统错误

* 第三方

  I/O 错误，网络、数据库、第三方服务等，提示用户稍后重试

###### 捕获异常

如果某个异常发生的时候没有在任何地方进行捕获，程序便会终止执行，并在控制台打印异常堆栈信息。使用 try/catch 语句块捕获异常，一个 try 语句块中可以捕获多个异常类型，SE 7 开始，同一个 catch 子句中可以捕获多个异常类型（当捕获的异常类彼此之间不存在子类关系时才需要这个特性），捕获多个异常时，细化异常类型在前（抛出异常类型是 catch 中声明异常的子类也算匹配）。

###### 抛出异常

创建异常类的对象（要自定义异常，需要继承 *Exception* 或其子类，应包含默认和有描述信息的构造器）并使用 throw 关键字抛出异常，一旦方法抛出了异常，这个方法就不能返回到调用者，如果抛出的异常没有被捕获，应用程序将崩溃

*   重新抛出异常

    在 catch 子句中可以抛出异常，这样可以改变异常的类型

    ```java
    catch (SQLException e) {
      	Throwable se = new ServletException("database error");
      	se.initCause(e); // 构建异常链
      	throw se;
    }
    // 捕获异常时，获取原始异常
    Throwable e = se.getCause();
    ```

*   如果只是把当前异常对象重新抛出，printStackTrace() 方法将打印原来异常抛出点的调用栈信息，而非重新抛出点的信息

    ```java
    // 更新异常堆栈信息, 调用 fillInStackTrace() 的那一行为异常新的发生地
    throw (Exception) e.fillInStackTrace();
    ```

###### finally 子句

在任何情况下，finally 块中的代码都会执行：

* 如果没有异常发生，在 try 内的代码执行结束后执行
* 如果有异常发生，在 catch 内代码结束后执行
* 如果有异常发生但没捕获，在异常被抛给上层之前执行

有 finally 语句时的 return 情况

* 如果在 try 或 catch 语句内有 return 语句，则 return 语句在 finally 语句执行结束后才执行，但 finally 不会改变返回值

  实际执行过程中，执行到 try 内的 return 语句前，会先将返回值保存在一个临时变量中，然后执行 finally 语句，最后返回临时变量，finally 中对 ret 的修改不会被返回）

  ```java
  public static int finallyReturn() // 返回 0
  {
      int ret = 0; 
      try {
          return ret;
      } finally {
          ret = 2;
      }
  }
  ```

* 如果在 finally 中存在 return 语句，则 try 和 catch 内的 return 会丢失（会掩盖 try 和 catch 内的异常，就像异常未发生），实际会返回 finally 中的返回值。

  ```java
  public static int finallyReturn()
  {
      int ret = 0;
      try {
          int a = 5/0;  // 触发 ArithmeticException 异常，但不会向上传递
          return ret;
      } finally {
          return 2;
      }
  }
  ```

* 如果 finally 中抛出了异常，则原异常也会被掩盖

  ```java
  public static int finallyReturn()
  {
      int ret = 0;
      try {
          int a = 5/0;  // 触发 ArithmeticException 异常，但不会向上传递
          return ret;
      } finally {
          // 抛出该异常
          throw new RuntimeException("hello");
      }
  }
  ```

以下情况 finally 代码不会被执行

```java
try {
	// do something
	System.exit(1);
} finally {
	System.out.println("Print from finally");
}
```

###### 异常机制良好实践

* 异常处理不能代替测试即只在异常情况下使用异常机制，因为捕获异常时间更久
* 不过分细化异常导致代码量膨胀
* 利用异常层次结构，不要只抛出 *RuntimeException* 异常，不要只捕获 *Thowable*
* 不要压制异常
* 早抛出，晚捕获（抛出异常到上层方法及应用）

###### throws

声明一个方法可能抛出的异常，支持逗号分割多个异常。对于未受检异常，不要求使用 throws 进行声明的，对于受检异常则必须进行声明（未声明则不能抛出，可以声明抛出但实际不抛出，子类不能抛出父类方法中没有声明的受检异常）

#### 断言

##### 断言机制

断言机制允许在测试期间向代码中插入一些检查语句。当代码发布时，这些插入的检测语句将会自动地移走。关键字 assert ：

```java
assert Condition;
// 或
assert Condition : expression;
```

这两种形式都会对条件进行检测，如果结果为 false，则抛出一个 *AssertionError* 异常。在第二种形式中，表达式将被传入 *AssertionError* 的构造器，并转换成一个消息字符串。（表达式部分的唯一目的是产生一个消息字符串。*AssertionError* 对象并不存储表达式的值，因此，不可能在以后得到它）

##### 启用和禁用断言

在默认情况下，断言被禁用

* 启用

  在运行程序时用 -enableassertions 或 -ea 选项启用，在启用或禁用断言时不必要重新编译程序。

* 禁用

  运行时使用 -da 禁用，类加载器将跳过断言代码，因此，不会降低程序运行的速度

由于可以使用断言，当方法被非法调用时，将会出现难以预料的结果，有时抛出断言错误，有时产生 null 指针异常，完全取决于类加载器的配置

##### try-with-resources

可以使资源（实现了 *java.lang.AutoCloseable*  接口的对象）自动关闭。资源可以定义多个，以分号分隔。在 Java 9 之前，资源必须声明和初始化在 try 语句块内，Java 9 去除了该限制，资源可以在 try 语句外被声明和初始化（但必须是 final 或事实上 final 的（没有声明未 final 但未重新赋值））

#### 日志

##### 基本日志

要生成简单的日志记录，可以使用全局日志记录器并调用其 info 方法

```java
// 记录日志
Logger.getGlobal().info("File->Open menu item selected");
// 取消日志
Logger.getGlobal().setLevel(Level.OFF);
```

##### 日志配置

可以通过修改配置文件来修改日志系统的属性，jdk11 配置文件位于 `$JAVA_HOME/conf/logging.properties`，

```shell
# 要使用另一个配置文件，将 java.util.logging.config.file 特性设置为配置文件存储位置
java -Djava.util.logging.config.file=configFile MainClass
```

日志管理器在虚拟机启动过程中初始化，在 main 执行之前完成，如果在 main 中调用 `System.setProperty("java.util.logging.config.file", file)`，也会调用 *LogManager*.readConfiguration() 来重新初始化日志管理器

*my_log.properties*

```properties
# 日志记录级别 
my_log.properties.level = WARNING
# 设置 handler，控制台输出
handlers= java.util.logging.ConsoleHandler
```

##### 使用

应用中不可直接使用日志系统（Log4j、Logback）的 API，而应依赖使用日志框架 SLF4J 的 API，使用门面模式的日志框架，有利于维护和各个类的日志处理方式统一

在日志输出时，字符串变量之间的拼接使用占位符的方式（Spring 字符串的拼接会使用 StringBuilder 的 append() 方式，有一定的性能损耗。使用占位符是替换动作，可以有效提升性能

```java
if (logger.isDebugEnabled) {
	logger.debug("Processing trade with id: {} and symbol: {}", id, symbol);
}
```

#### 泛型

泛型实现了参数化类型的概念，使代码可以用于多种类型。泛型的主要目的之一就是用来指定容器要持有什么类型的对象，而且由编译器来保证类型的正确性。是一种编译期的技巧，Java 编译期会自动将类型转换为对应的特定类型，使用泛型，必须保证相应类型可以转换为 Object

##### 泛型机制

普通的类和方法，只能使用具体的类型；要们是基本类型，要么是自定义的类。在面向对象中，多态算是一种泛化机制。SE 5 引入泛型实现了参数化类型的概念。使代码能够应用于某种不具体的类型，而不是一个具体的接口或类。创建参数化类型的一个实例时，编译器会负责转型操作，并且保证类型的正确性。

在泛型代码内部，无法获得任何有关泛型参数类型的信息，泛型使用擦除来实现

###### 泛型类

泛型类作用：创造容器类；元祖类（在需要返回多个对象的情况下，可以创建一个对象，用它来持有想要返回的多个对象，使用泛型，在编译期就能确保类型安全，将一组对象打包存储于其中一个单一对象，这个容器对象允许读取其中元素，但是不允许向其中存放新的对象）

一个泛型类就是具有一个或多个类型变量的类。类型变量使用尖括号 <> 括起来，放在类名的后面。使用泛型类时，必须在创建对象时指定类型参数的值。

类型变量使用大写形式：

* E 表示集合的元素类型
* K 和 V 分别表示键与值的类型
* T、U、S 表示任意类型

泛型类可看作普通类的工厂，SE 7 及以后的版本中，表达式右边的类型变量可以省略。

###### 泛型接口

泛型可以应用于接口，如生成器，实际上，这是工厂方法设计模式的一种应用。

###### 泛型方法

泛型方法可以定义在普通类中，也可以定义在泛型类中，用尖括号 <> 修饰类型变量。类型变量放在修饰符的后面，返回类型的前面。调用一个泛型方法时，在方法名前的尖括号中放入具体的类型。

泛型方法使得该方法能够独立于类而产生变化，<u>无论何时，只要能做到，就应该尽量使用泛型方法，如果使用泛型方法可以取代整个类泛型化，就应该只使用泛型化方法。对于 static 方法而言，无法访问泛型类的类型参数，所以如果 static 方法需要使用泛型能力，就必须使其成为泛型方法</u>

大多数情况下，调用泛型方法可以省略尖括号中类型参数。编译器有足够的信息能够推断出所调用的方法。

当使用泛型类时，必须在创建对象的时候指定类型参数的值，而使用泛型方法时，通常不必指明参数类型，编译器会使用类型参数推断

###### 泛化的 Class 引用

Class 引用总是指向某个 Class 对象，int.class 不继承 Number.class，此时可以使用通配符 ?，或使用 `Class<? extend Number>` 来进行限定为 Number 或 Number 的任何子类型

在 SE5 中，Class＜?＞优于的 Class，Class 不会产生编译器警告信息

##### 泛型特性

###### 边界-类型限定

边界使得可以在用于泛型的参数类型上设置限制条件。因为擦除移除了类型信息，所以，可以用无界泛型参数调用的方法只是那些可以用 Object 调用的方法。如果将这个参数限制为某个类型子集，就可以用这些类型子集来调用方法。为了这些这种限制，泛型重用了 extends 关键字。extends 关键字在泛型边界上下文环境中和在普通情况下所具有的意义是完全不同的

extends

```java
// 具有从 Dimension 和 HasColor 和 Weight 继承或实现的类型
public <T extends Dimension & HasColor & Weight> void add(T t);
```

###### 通配符

在泛型参数表达式中的问号。通配符限定与类型变量限定十分类似，还提供了一个附加能力，可以指定一个超类型限定

```java
// 任何泛型 Pair 类型，它的类型参数是 Employee 的子类
Pair<? extends Employee>
// 该通配符限制为 Manager 的所有超类型，可以为方法提供参数，但不能使用返回值
Pair<? super Manager>
```

* ?

  无界通配符，声明：用的泛型来编写这段代码，并不是要用原生类型，在当前这种情况下，泛型参数可以持有任何类型

* &

  指定必须实现多个类型

泛型通配符 <? extends T> 来接收返回的数据。此写法的泛型集合不能使用 add 方法，而 <? super T>  不能使用 get 方法。频繁往外读取内容时，适合用 <? extends T>，经常往里写数据时，适合用 <? super T>

```java
Pair<? extends Employee>
```

###### 类型擦除

无论何时定义一个泛型类型，都自动提供了一个相应的原始类型。原始类型的名字就是删去类型参数后的泛型类型名。

原始类型用第一个限定的类型变量来替换，如果没有给定限定就用 Object 替换。

* 使用强制转换一个对象为泛型会得到警告，使用 instanceof 测试对象是否为泛型类时会发生编译错误
* 不能创建参数化类型的数组
* 不能在静态域和方法中引用类型变量
* 不能抛出或捕获泛型类的实例，泛型类也不能直接或间接继承自 Throwable 
* 任何基本类型都不能作为类型参数。
* 一个类不能实现同一个泛型接口的两种变体，由于擦除的原因，这两个变体会成为相同的接口
* 不能创建 new T()，因为擦除和编译器不能验证 T 具有默认（无参）构造器

<u>在泛型代码内部，无法获得任何有关泛型参数类型的信息</u>，如果打印泛型类的参数（`genericObj.getClass().getTypeParameters()`）只能得到参数占位符的标识

在基于擦除的实现中，泛型类型被当作第二类类型处理，即不能在某些重要的上下文环境中使用的类型。泛型类型只有在静态类型检查期间才出现，在此之后，程序中的所有泛型类型都将被擦除，替换为它们的非泛型上界。List<T> 这些的类型将被擦除为 List，而普通的类型变量在未指定边界的情况下将被擦除为 Object，擦除使得泛化的客户端可以用非泛化的类库来使用，反之亦然，通过允许非泛型代码和泛型代码共存，擦除使得泛型迁移成为可能，擦除使得在不破坏现有类库的情况下，将泛型融入 Java 语言。擦除使得现有的非泛型客户端代码能够在不改变的情况下继续使用。

擦除代价是泛型不能用于显式的应用运行时类型的操作之上（转型、instanceof 操作、new 表达式）使用泛型并不是强制的，当没有泛型参数时，可以使用 `@SuppressWarnings("unchecked")` 来屏蔽警告

即使擦除在方法或类内部移除了有关实际类型的信息，编译器仍可以确保在方法或类中使用的类型的内部一致性，擦除在方法体中移除了类型信息，在运行时问题是边界：即对象进入和离开方法的地点（编译器在编译期执行类型检查并插入转型代码的地点），泛型中的所有动作（对传递进来的值进行额外的编译器检查，并插入对传递出去的值的转型）都发生在边界处

###### 自限定的类型

```java
// SelfDounded 类接受泛型参数 T，而 T 由一个边界类限定，这个边界就是拥有 T 作为其参数的 SelfBounded
class SelfBounded<T extends SelfBounded<T>> {}
```

自限定将采取额外的步骤，强制泛型当作其自己的边界参数来使用。强制要求将正在定义的类当作参数传递给基类，它可以保证类型参数必须与正在被定义的类型相同

###### 参数协变

方法参数类型会随着子类而变化。还可以产生与子类类型相同的返回类型（SE 5 中引入）

```java
// 返回子类型
class Base {}
class Derived extends Base{}
interface OrdinaryGetter { Base get(); }
interface DerivedGetter extends OrdinaryGetter { @Override Derived get(); }
```

###### 数组与泛型

通常数组与泛型不能很好结合使用。不能实例化具有参数化类型的数组

```java
Peel<Banana>[] za = new Peel<Banana>[10]; // error
```

擦除会移除参数类型信息，而数组必须知道它们所持有的确切类型，以强制保证类型安全。但可以参数化本身如使用参数化方法时使用参数化数组类型。

泛型在类或方法的边界处很有效，而在类或方法的内部，擦除通常会使泛型变得不适用。

#### 注解

##### 注解机制

注解是那些插入到源代码中使用其他工具可以对其进行处理的标签。这些工具可以在源码层次上进行操作，或者可以处理编译器在其中放置了注解的类文件。

注解不会改变程序的编译方式，java 编译器对于包含注解和不包含注解的代码会生成相同的虚拟机指令                                                                                  

在 java 中，注解是当作一个修饰符来使用的，它被置于被注解项之前，中间没有分号，每一个注解的名称前都加了 @ 符号，类似于 Javadoc 的注释，Javadoc 注释出现在注释符内部，而注解是代码的一部分

除了方法外，还可以注解类、成员以及局部变量，这些注解可以存在于任何可以放置一个像 public 或者 static 这样的修饰符的地方。还可以注解包、参数变量、类型参数和类型用法。每个注解都必须通过一个注解接口进行定义，这些接口中的方法与注解中的元素相对应。

###### 元注解

定义注解时，需要一些『元注解』：

* @Taget

  定义注解将应用于什么地方，一条没有 @Taget 限制的注解可以应用于任何项上。可选值为 *java.lang.annotation.ElementType* 枚举类实例

  |    元素类型     |          注解适用场合          |
  | :-------------: | :----------------------------: |
  | ANNOTATION_TYPE |          注解类型声明          |
  |     PACKAGE     |               包               |
  |      TYPE       | 类（包含枚举）接口（包括注解） |
  |     METHOD      |              方法              |
  |   CONSTRUCTOR   |             构造器             |
  |      FIELD      |      成员域包含 enum 常量      |
  |    PARAMETER    |        方法或构造器参数        |
  | LOCAL_VARIABLE  |            局部变量            |
  | TYPE_PARAMETER  |            类型参数            |
  |    TYPE_USE     |            类型用法            |

* @Retention

  定义该注解在哪一个级别可以用，可选值为 *java.lang.annotation.RetentionPolicy* 枚举类实例

  |   值    |                             描述                             |
  | :-----: | :----------------------------------------------------------: |
  | SOURCE  |                   源码，注解将被编译器丢弃                   |
  |  CLASS  |      类文件，注解在 class 文件中可用，但会被虚拟机丢弃       |
  | RUNTIME | 运行时，虚拟机将在运行期也保留，可以通过反射记住读取注解的信息 |

* @Documented

  此注解包含在 Javadoc 中

* @Inherited

  允许子类继承父类中的注解

###### 标准注解

SE 在 java.lang、java.lang.annotation、javax.annotation 包中定义了大量的注解接口

|       注解接口       |      应用场合      |                             目的                             |
| :------------------: | :----------------: | :----------------------------------------------------------: |
|     @Deprecated      |        全部        |                          标记为过时                          |
|  @SuppressWarnings   |  除了包和注解之外  |                  阻止某个给定类型的警告信息                  |
|     @SafeVarargs     |    方法和构造器    |                断言 varargs 参数可以安全使用                 |
|      @Override       |        方法        |                      覆写超类或接口方法                      |
| @FunctionalInterface |        接口        |           将接口标记为只有一个抽象方法的函数式接口           |
|    @PostConstruct    |        方法        |             被标记的方法应该在构造之后立即被调用             |
|     @PreDestroy      | 类，接口，方法，域 |             被标记的方法应该在移除之前立即被调用             |
|      @Resource       | 类、接口、方法、域 | 在类或接口，标记为其他地方要用到的资源，在方法或域上为『注入』而标记 |
|      @Resources      |      类，接口      |                         一个资源数组                         |
|      @Generated      |        全部        | 提供代码生成工具来使用，任何生成的源代码都可以被注解，从而与程序员提供的代码区分开。 |
|        Target        |        注解        |                 指明可以应用这个注解的那些项                 |
|      Retention       |        注解        |                   指明这个注解可以保留多久                   |
|      Documented      |        注解        |             指明这个注解应该包含在注解项的文件中             |
|      Inherited       |        注解        |     指明当这个注解应用于一个类的时候，能自动被它子类继承     |
|      Repeatable      |        注解        |              指明这个注解可以在同一项上应用多次              |

* 用于管理资源的注解

  * @PostConstruct 和 @PreDestroy 注解用于控制对象生命周期的环境中，如 web 容器和应用服务器。标记这些注解的方法应该在对象被构建之后，或者在对象被移除之前，紧接着调用

  * @Resource 注解用于资源注入。如，访问数据库的 web 应用。当然，数据库访问信息不应该被硬编码到 Web 应用中。而是应该让 Web 容器提供某种用户接口，以便设置连接参数和数据库资源的 JNDI 名字，当包含这个域的对象被构造时，容器会注入一个对该数据源的引用

##### 自定义注解

###### 定义注解接口

@interface 声明创建了一个真正的 java 接口，与其他任何 java 接口一样，注解也将会编译成 class 文件。

没有元素的注解为『标记注解』

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UseCase {
    int id();
  	// 在注解某个方法时没有给出，则注解处理器使用默认值
    String description() default "no description";
}
```

所有的注解接口都隐式地扩展 java.lang.annotation.Annotation 接口。这个接口是一个常规接口，不是一个注解接口。无法扩展注解接口，即所有的注解接口都直接扩展自 java.lang.annotation.Annotation

标签 @UseCase 由 UseCase.java 定义，其中包含 int 元素 id，以及一个 String 元素 description，注解元素可用的类型：

* 基本类型
* String
* Class
* enum
* Annotation
* 以上类型的数组

如果使用了其他类型，编译器会报错，也不允许使用任何包装类型，元素不能有不确定的值（元素必须要么具有默认值，要么在使用注解时提供元素的值）

对于非基本类型的元素，无论是在源代码中声明时，或是在注解接口中定义默认值时，都不能以 null 作为其值。

###### 使用注解

```java
public class PasswordUtils {
    @UseCase(id = 47, description = "Password must contain at least one numeric")
    public boolean validatePassword(String password) {
        return password.matches("\\w*\\d\\w*");
    }
}
```

注解的元素在使用时表现为 key = value 形式

###### 注解处理器

如果没有用来读取注解的工具，那么注解也不会比注释更有用。使用注解的过程中，很主要的一个部分就是创建与使用『注解处理器』

```java
public static void trackUseCases(List<Integer> useCases, Class<?> cl) {
        for (Method method : cl.getDeclaredMethods()) {
            UseCase useCase = method.getAnnotation(UseCase.class);
            if (useCase != null) {
                System.out.println("Found Use Case:" + useCase.id() + " " + useCase.description());
            }
        }
        for (int i: useCases) {
            System.out.println("Warning: Missing use case-" + i);
        }
    }
```

注解处理已经被集成到 java 编译器中，在编译过程中调用注解处理器

```shell
javac -processor ProcessorClassName1,ProcessorClassName2... sourceFiles
```

编译器会定位源文件中的注解，每个注解处理器会依次执行，并得到它表示感兴趣的注解。如果某个注解处理器创建了一个新的源文件，那么将重复执行这个处理过程。如果某次处理循环没有再产生任何新的源文件，那么就编译所有的源文件，注解处理器只能产生新的源文件，它无法修改已有的源文件

注解处理器通常通过扩展 *AbstractProcessor* 类而实现 *Processor* 接口。

###### 常用注解

|         注解         |                             含义                             |           备注            |
| :------------------: | :----------------------------------------------------------: | :-----------------------: |
|     @Deprecated      |                声明不建议使用，方法/类上声明                 |                           |
|      @Override       |                覆写基类或接口方法，方法上声明                |                           |
|  @SuppressWarnings   |                         抑制警告信息                         |                           |
|        @Named        |    为 Bean 设置 ID，可以替代 Spring 中的声明 Bean 的注解     | Java Dependency Injection |
|       @Inject        | 自动装配 Bean，在方法或属性上声明 Bean 之间依赖关系，可以替代 Spring 的 @Autowired | Java Dependency Injection |
| @FunctionalInterface |                        标注函数式接口                        |                           |
|                      |                                                              |                           |
|                      |                                                              |                           |
|                      |                                                              |                           |

#### lambda

Lambda 即闭包

##### lambda 表达式语法

Lambda 即带参数变量的表达式，由参数，箭头操作符及代码块组成。无需指定 lambda 表达式的返回类型，返回类型总是会由上下文推导出。

如果一个 lambda 表达式只在某些分支返回一个值，而在另外一些分支不返回值，这是不合法的

```java
(String first, String second)
    ->first.length() - seconde.length()
```

如果代码要完成的计算无法放在一个表达式中，就可以像写方法一样，把这些代码放在 ｛｝  中，并包含显式的 return 语句。

```java
(String first, String second)->
	{
        if (first.length() < second.length()) {
            return 1;
        } else {
            return 0;
        }
	}
```

即使 lambda 表达式没有参数，仍然要提供空括号，就像无参数方法一样，如果方法只有一个参数，而且这个参数的类型可以推导得出，还可以省略小括号。

```Java
()->{
    for (int i = 100; i )
}
```

如果可以推导出一个 lambda 表达式的参数类型，则可以忽略其类型。如果方法只有一个参数，而且这个参数的类型可以推导得出，那么可以省略小括号

```java
(first, second)->first.length() - second.length()
```

###### 方法引用

可以为接受函数式对象的参数传递一个方法引用来代替 Lambda 表达式。方法引用是一个类名和对象引用，后跟双冒号运算符和方法名，可以在方法引用中使用 this 和 super 引用，使用 ::new 进行构造器引用

* Object::instanceMethod

  方法引用等价于提供方法参数的 lambda 表达式

* Class::staticMethodName

  如果静态方法具有与函数式接口的抽象方法兼容的返回类型和兼容的参数类型，可以将对静态方法的引用作为参数传递给函数式接口方法

* Class::instanceMethod

  第一个参数会成为方法的目标

###### 变量作用域

Lambda 类似闭包作用域

##### 函数式接口

只有一个抽象方法的接口，该抽象方法不能覆盖 *java.lang.Object* 类的方法。函数式接口也称为单抽象方法接口。Lambda 表达式等价于实现函数式接口的实例。

*java.util.function* 包中定义了很多非常通用的函数式接口，可以使用 @FunctionalInterface 标注函数式接口

###### Function/BiFunction

* Function

  ```java
  public interface Function<T, R>
  ```

  创建一个返回结果的单参数函数，是一个参数化类型，调用函数是通过调用 apply 方法完成

* BiFunction

  是 Function 变体，接受两个参数

* IntFunction/LongFunction/DoubleFunction

  Function 变体，只返回参数化的返回类型

* IntToDouble/DoubleToInt

  接受 Int 返回 double，直接声明参数，前面为参数，后面为返回参数，没有参数化。

###### Predicate

参数化接口，接受一个参数并根据该参数值返回布尔值，抽象方法为 test

###### Supplier

参数化接口，不接受任何参数并返回一个值，抽象方法为 get

###### Consumer

参数化接口，不返回接口，抽象方法为 accept



#### 国际化

##### *Locale*

标识或获得地区，语言，是国际化的基础

##### *NumberFormat*

`java.text.NumberFormat` 是一个抽象类，用于数字格式化，依赖于 *Locale*，getNumberInstance() 、getCurrencyInstance() 、getPercentInstance()，这些方法返回的对象可以分别对数字、货币量和百分比进行格式化和解析。

##### Currency

格式化货币值，可以使用 *NumberFormat*.getCurrencyInstance() 方法。但是这个方法灵活性不好，它返回的是一个只针对一种货币的格式器。

使用 *Currency* 类来控制被格式器所处理的货币。可以通过将一个货币标识符传给静态的 *Currency*.getInstance() 方法来得到一个 *Currency* 对象，然后对每一个格式器都调用 setCurrency() 方法

货币标识符由 ISO 4217 定义，美元：USD，人民币：CNY

#### 排序和范化

##### 字符校对与范化

###### Collator

compareTo() 方法使用的是字符串的 UTF-16 编码值，这会导致很荒唐的结果，即使在英文比较中也是如此。为了获得 *Locale* 敏感的比较符，可以调用静态的 *Collator*.getInstances() 方法，获得本地排序器

排序器有几个高级设置项。可以设置排序器的强度以此来选择不同的排序行为。字符间的差别可以被分为首要的（primary）、其次的（secondary）和再次的（tertiary）。

如果将排序器的强度设置成 Collator.PRIMARY，那么排序器将只关注 primary 级的差别。如果设置成 Collator.SECONDARY，排序器将把 secondary 级的差别也考虑进去。

如果强度被设置为 Collator.IDENTICAL 则不允许有任何差异。这种设置与排序器的第二种具有相当技术性的设置，即分解模式，联合使用时，就会显得非常有用

###### Unicode 范化机制

字符或字符串被表述为Unicode 时有多种形式（ffi 可以用代码 U+FB03 描述成单个拉丁字符，也可以被描述成三个小写字母）。Unicode 标准字符串定义了四种范化形式：

* D

  重音符号被分解为基字符和组合重音字符

* KD

  完全分解字符

* C

  重音符号总是组合的，如带音标字母被组合成一个字母

* KC

  分解字符

指定排序器使用的范化程度：*Collator*.NO_DECOMPOSITION 表示不对字符串做任何范化，默认值 *Collator*.CANONICAL_DECOMPOSITION 使用 D 形式。

###### Normalizer

在将字符串存储到数据库中，或与其他程序进行通信时。*java.text.Normalizer* 类实现了对范化的处理

##### 消息格式化

###### *MessageFormat*

java 类库中 *MessageFormat*，与用 printf() 方法进行格式化很类似，但是它支持 *Locale* ，并且会对数字和日期进行格式化。

支持在字符串中使用大括号里加数字索引占位，占位符索引后面可以跟一个类型（type）和一个风格（style）, 它们之间用逗号隔开。

类型是：number , time、date 、choice。类型是 number 那么风格可以是 integer 、currency、percent，如果类型是 time 或 date ，风格可以是 short，medium、long、full 或者是一个日期格式模式，就像 yyyy-MM-dd。choice 类型指定了消息能够随占位符的值而变化。

静态的 *MessageFormat*.format() 方法使用当前的 *Locale* 对值进行格式化，可以实例化带 *Locale* 的 *MessageFormat*

#### 正则表达式

##### 正则表达式的匹配

###### Pattern 和 Matcher

1.通过调用静态方法 `Pattern.compile()` 来创建一个模式

2.为每个String（或其他字符序列）调用 `pattern.matcher(CharSequence)`，以从模式中请求一个 `Matcher`

3.在结果 `Matcher` 中调用（一次或多次）方法

正则表达式用于指定字符串的模式，可以在任何需要定位匹配某种特定模式的字符串的情况下使用正则表达式。

正则表达式的常用就是测试某个特定的字符串是否与它匹配。在 Java 中，首先用表示正则表达式的字符串构建一个 `Pattern` 对象。然后从这个模式获得一个 `Matcher`，并调用它的 `matches` 方法

```java
Pattern pattern = Pattern.compile(patternSing);
Matcher matcher = pattern.matcher(input);
if (matcher.matches()) {
    m.group();
}
```

这个匹配器的输入可以实任何实现了 `CharSequence` 接口的类的对象

在编译这个模式时，可以设置一个或多个标志

```java
Pattern pattern = Pattern.compile(expression, Pattern.CASE_INSENSIVE + Pattern.UNICODE_CASE);
```

标志：

* `Pattern.CASE_INSENSITIVE` 或 `r`：匹配字符时忽略字母的大小写，默认情况下，这个标志只考虑 `US ASCII` 字符
* `Pattern.UNICODE_CASE` 或 `u`：当与 `CASE_INSENSITIVE` 组合使用时，用 `Unicode` 字母的大小写来匹配
* `Pattern.UNICODE_CHARACTER_CLASS` 或 `U`：选择 `Unicode` 字符流代替 `POSIX` ，其中蕴含了 `UNICODE_CASE`
* `Patern.MULITLINE` 或 `m`：`^` 和 `$` 匹配行的开头和结尾，而不是整个输入的开头和结尾
* `Pattern.UNIX_LINES` 或 `d`：在多行模式中匹配 `^` 和 `$` 时，只有 `\n` 被识别成行终止符
* `Pattern.DOTALL` 或 `s`：当使用这个标志时，`.` 符号匹配所有字符，包括行终止符
* `Pattern.COMMENTS` 或 `x`：空白字符和注释（从 # 到行末尾）将被忽略
* `Pattern.LITERAL`：该模式将被逐字地采纳，必须精确匹配，因字母大小写而造成的差异除外
* `Pattern.CANON_EQ`：考虑 `Unicode` 字符规范的等价

最后两个标志不能在正则表达式内部指定

如果想要在集合或流中匹配元素，可以将模式转换为谓词

```java
Stream<String> strings = ...;
Stream<String> result = strings.filter(pattern.asPredicate());
```

其结果中包含了正则表达式的所有字符串

如果正则表达式包含群组，那么 `Matcher` 对象可以揭示群组的边界。

`Matcher` 有多个 `finder` 方法，能比 `String` 的 `match` 操作提供更大的灵活性，这些方法返回布尔值，返回 `true` 意味着匹配成功，`false` 意味着匹配不成功。

* `matches()` 

  将整个字符串和模式比较，这和 `java.lang.String` 中一样。因为它匹配整个字符串

* `lookingAt()`

  只在字符串的开始匹配

* `find()`

  在字符串中匹配模式（不一定非从字符串的第一个字符开始），从字符串的首字符开始，或者如果之前已成功调用该方法，则从未能符合前面匹配的第一个字符开始匹配

匹配成功后，可以使用以下方法获取匹配信息：

* `start()，end()`

  分别返回匹配结果在字符串中的开始或结束位置

* `groupCount()`

  返回用括号括起来的捕捉组的数量，如果没有分组则返回 0

* `group(int i)`

  如果 `0 <= i <= groupCount()`，则返回当前匹配中与分组 i 匹配的字符。如果组号是 0，则表示完全匹配。`group(0)` 或 `group()` 将返回匹配的整个字符串，group 即匹配正则表达式括号中内容

##### 替换

Java 正则表达式提供了一些相应的替换方法，对于各种形式的替换方法而言，都需要传递替换文本或替换“右手边”这样的参数（在命令行文本编辑器的替换命令中，左手边是模式，右手边是替换文本）

* `replaceAll(newString)`

  用 `newString` 替换所有匹配的地方

* `appendReplacement(StringBuffer, newString)`

  将匹配结果之前的字符串填加到 `StringBuffer`，再将匹配结果替换为 `newString`，并追加到 `StringBuffer`。即获取替换后的 `StringBuffer`

* `appendTail(StringBuffer)`

  将上次替换过的内容连接后面未替换过的内容，并放入 `StringBuffer`（通常在调用 `appendReplacement()` 之后调用 `appendTail()`，即获取原始`StringBuffer`

#### 时间日期

##### SE 8 之前

###### Date

1.0 中，对日期和时间的支持智能依赖 *java.util.Date* 类，只能以毫秒的精度表示时间，年份从 1900 年开始，月份从 0 开始，且对象可变。代表一个时刻，绝对时间，与年月日无关。内部由 long 类型表示纪元的毫秒数（默认 `System.currentTimeMillis()` 未当前时刻距离纪元时的毫秒数）

###### Time

1.0 时间类

###### Calendar

1.1 中，*Date* 类的很多方法被废弃了（getDate、getMonth、getYear），使用 *Calendar* 类代替表示年历是抽象类，表示公历子类是 *GrgorianCalendar*，修改了时间，但月份还是从 0 开始，对象可变。

*Calendar* 类是日期和时间操作中的主要类，表示与 *TimeZone* 和 *Locale* 相关的日历信息。内部有一个表示时刻的毫秒数。和日历各个字段值的数组。由静态变量引用这些字段（*Calendar.YEAR*、*Calendar.MONTH*：月，1 月是 0、*C*）

###### DateFormat

*java.text.DateFormat* 格式化和解析日期或时间，非线程安全。支持样式和模式，将日期和时间与字符串相互转换。抽象类，常用子类 *SimpleDateFormat*

|       样式        |           模式           |
| :---------------: | :----------------------: |
| DateFormat.SHORT  |         12/2/18          |
| DateFormat.MEDIUM |       Dec 2, 2018        |
|  DateFormat.LONG  |     December 2, 2018     |
|  DateFormat.FULL  | Friday, December 2, 2018 |

默认 DateFormat 是宽松的（Lenient），可以接收不存在的日期，自由地将日期改为存在的日期。否则不会接收不存在的日期

###### *TimeZone*

时区（抽象类），默认获取当前时区 `user.timezone` 属性指定

```java
java -Duser.timezone=Asia/Shanghai
// 获取
TimeZone tz = TimeZone.getTimeZone("US/Eastern");
TimeZone tz = TimeZone.getTimeZone("GMT + 08:00");
```

###### *Locale*

国家地区和语言。由国家（地区）和语言代码组成。国家地区不是必须。

##### SE 8

SE 8 中引入了 *java.time* 包以提供日期和时间支持，还包含 *java.time.chrono*，*java.time.format*，*java.time.temporal*，*java.time.zone*

###### LocalDate

用于为日期建模，不带时间，不包含时区。`LocalDate.until` 方法支持以指定时间单位到另一个日期的时间

```java
public int intvalDays(LocalDate start, LocalDate end) {
  	return start.until(end, ChronoUnit.DAYS);
}
```

###### LocalDate 与 LocalTime

该类实例是一个不可变对象，它只提供了简单的日期，并不包含当天的时间信息和任何与时区相关的信息。可以通过 getYear() 等方法或直接传递一个 *TemporalField* 参数给 get() 方法拿到相关信息（*TemporalField* 是一个接口，它定义了如何访问 *Temporal* 对象某个字段的值，枚举 *ChronoField* 实现了该接口）

###### LocalDateTime

是 *LocalDate* 和 *LocalTime* 复合类。同时表示了日期和时间，但不带有时区信息，可以直接创建，也可以合并日期和时间对象创建，支持从中提取 *LocalDate* 和 *LocalTime*

###### Instant

*java.time.Instant* 类似 UNIX_TIMESTAMP，包含秒和纳秒（0～999999999），表示时间线上的一个点，通常用于计算操作的时间。

```java
// instant 构建local time
Instant instant = Instant.now();
LocalDateTime localDateTime = LocalDateTime.ofEpochSecond(instant.getEpochSecond(),
                instant.getNano(),
                ZoneOffset.ofTotalSeconds(28800));
```

###### Duration

*Duration* 类主要用于秒和纳秒对持续时间建模，类似 Period，定义了两个 *Temporal* 对象时间段。但不能创建 *LocalDateTime* 和 *Instant* 两类对象的 *Duration*。会触发 *DateTimeException* 异常。

*用于时间的 Instant 和 Duration 算术运算*

|                             方法                             |                             描述                             |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|                         plus，minus                          |      在当前的 Instant 或 Duration 上加/减一个 Duration       |
| plusNanos，plusMillis，plusSeconds，minusNanos，minusMillis，minusSeconds |    在当前的 Instant 或 Duration 上加/减给定时间单位的数值    |
| plusMinutes，plusHours，plusDays，minusMinutes，mi nusHours，minusDays |          在当前 Duration 上加/减给定时间单位的数值           |
|               multipliedBy，dividedBy，negated               | 返回由当前 Duration 乘/除给定 long 或 -1 而得到 Duration，可以缩放 Duration 但不能缩放 Instant |
|                      isZero，isNegative                      |             检查当前的 Duration 是否是 0 或负值              |

*Instant* 和 *Duration* 类都是不可修改的类

###### Period

如果需要以年，月，日等周期性的方式对多个时间单位建模，可以使用 *Period* 类。

###### ZonedDateTime

对带有时区的日期时间建模，互联网编码分配管理机构（Internet Assigned Numbers Authority，IANA）保存着一个数据库，里面存储着世界上所有已知的时区，它每年会更新数次，而批量更新会处理夏令时的变更规则，java 使用了 IANA数据库

每个时区都有一个 ID，*ZonedDateTime* 与 *LocalDateTime* 有大量相同的方法

###### ZoneId

表示时区标识符

###### ZonedDateTime

ZoneDateTime 类用于对带时区的日期-时间建模，是不可变，且时间分量的存储精度为纳秒

###### DateTimeFormatter

格式化本地或带时区的日期-时间，用于在 *LocalDate*、*LocalDateTime*、*LocalTime*、*ZoneDateTime* 的 format 方法参数

*预定义格式器*

|        格式器        |                             描述                             |                    示例                     |
| :------------------: | :----------------------------------------------------------: | :-----------------------------------------: |
|    BASIC_ISO_DATE    |            年、月、日、时区偏移量，中间没有分隔符            |                19690716-0500                |
|    ISO_LOCAL_DATE    |                      分隔符为：-、：、T                      |                 1969-07-16                  |
|    ISO_LOCAL_TIME    |                        以 ：为分隔符                         |                  09:32:00                   |
| ISO_LOCAL_DATE_TIME  |                          以 T 分隔                           |             1969-07-16T09:32:00             |
|   ISO_OFFSET_DATE    |             类似 ISO_LOCAL_XXX 但是有时区偏移量              |              1969-07-16-05:00               |
|   ISO_OFFSET_TIME    |                                                              |               09:32:00-05:00                |
| ISO_OFFSET_DATE_TIME |                                                              |          1969-07-16T09:32:00-05:00          |
| ISO_ZONED_DATE_TIME  |                    有时区偏移量和时区 ID                     | 1969-07-16T09:32:00-05:00[America/New_York] |
|     ISO_INSTANT      |                在 UTC 中，用 Z 时区 ID 来表示                |            1969-07-16T14:32:00Z             |
|       ISO_DATE       |            类似 ISO_OFFSET_DATE，但时区信息是可选            |              1969-07-16-05:00               |
|       ISO_TIME       |                     类似 ISO_OFFSET_TIME                     |               09:32:00-05:00                |
|    ISO_DATE_TIME     |                   类似 ISO_ZONED_DATE_TIME                   | 1969-07-16T09:32:00-05:00[America/New_York] |
|   ISO_ORDINAL_DATE   |                    LocalDate 的年和年日期                    |                  1969-197                   |
|    ISO_WEEK_DATE     |                LocalDate 的年，星期和星期日期                |                 1969-W29-3                  |
|  RFC_1123_DATE_TIME  | 用于邮件时间戳的标准，RFC822，并在 RFC1123 中将年份更新到 4 位 |       Web, 16 Jul 1969 09:32:00 -0500       |

标准格式器主要是为了机器刻度的时间戳设计的。对人类可以使用与 Locale 相关的格式化风格：SHORT、MEDIUM、LONG、FULL

可以通过指定模式来定制自己的日期格式

*常用的日期/时间格式的格式化符号*

|    时间域或目的     |                             示例                             |
| :-----------------: | :----------------------------------------------------------: |
|         ERA         |                            G: AD                             |
|     YEAR_OF_ERA     |                      yy:69，yyyy: 1969                       |
|    MONTH_OF_YEAR    |         M: 7，MM: 07，MMM： Jul，MMM：July，MMMM：J          |
|    DAY_OF_MONTH     |                          d:6，dd:06                          |
|     DAY_OF_WEEK     |            e:3，E：Wed，EEEE：Wednesday，EEEEE：W            |
|     HOUR_OF_DAY     |                         H：9，HH：09                         |
| CLOCK_HOUR_OF_AM_PM |                         K：9，KK：09                         |
|     AMPM_OF_DAY     |                            a：AM                             |
|   MINUTE_OF_HOUR    |                            mm：02                            |
|  SECOND_OF_MINUTE   |                            ss: 00                            |
|   NANO_OF_SECOND    |                        nnnnnn：000000                        |
|       时区 ID       |                     VV：America/New_York                     |
|       时区名        |             z：EDT，zzzz：Eastern Daylight time              |
|     时区偏移量      | x：-04，xx：-0400，xxx：-4:00，XXX：与 xxx 相同，但 z 表示为 0 |
| 本地化的时区偏移量  |                   0:GMT-4，0000:GMT-04:00                    |

