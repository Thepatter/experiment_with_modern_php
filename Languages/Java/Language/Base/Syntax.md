### Syntax

#### 数据类型

##### 基本数据类型

 8 种基本类型，包含 4 种整型、2 种浮点类型、1 种用于表示 Unicode 编码的字符类型、布尔类型，基本类型值传递时采用复制传值

|  类型   | 字长  |  最小值   |    最大值     | 包装类型  |                          特殊常量                           |
| :-----: | :---: | :-------: | :-----------: | :-------: | :---------------------------------------------------------: |
| boolean |       |           |               |  Boolean  |                         TRUE/FALSE                          |
|  char   | 16bit | Unicode 0 | Unicode 65535 | Character |                                                             |
|  byte   | 8bit  |   -128    |      127      |   Byte    |                                                             |
|  short  | 16bit |   -2^15   |    2^15-1     |   Short   |                                                             |
|   int   | 32bit |   -2^31   |    2^31-1     |  Integer  |                                                             |
|  long   | 64bit |   -2^63   |    2^63-1     |   Long    |                                                             |
|  float  | 32bit |  IEEE754  |    IEEE754    |   Float   |                                                             |
| double  | 64bit |  IEEE754  |    IEEE754    |  Double   | POSITIVE_INFINITY（正无穷)/NEGATIVE_INFINITY（负无穷），NaN |
|  void   |       |           |               |   Void    |                                                             |


原始数据类型操作不是线程安全的（基本类型的值直接栈中，对其进行操作并非原子操作），需要使用并发相关类（*AtomicInteger* 等）。对于比较宽的数据类型，float、double 甚至不能保证更新操作的原子性，可能出现程序读取到只更新了一半数据位的数值

###### 比较

浮点数计算结果之间的等值判断：

比较方式：

指定一个误差范围，两个浮点数的差值在此范围之内，则认为是相等的

```java
float a = 1.0f - 0.9f;
float b = 0.9f - 0.8f;
float diff = 1e-6f;
if (Math.abs(a - b) < diff) {
    System.out.print("true");
}
```

使用 *BigDecimal* 来定义值，再进行浮点数的运算操作

```java
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("0.9");
BigDecimal c = new BigDecimal("0.8");
BigDecimal x = a.subtract(b);
BigDecimal y = b.subtract(c);
if (x.compareTo(y) == 0) {
	System.out.println("true");
}
```

##### char

char 类型的字面量要用单引号括起来，可以用十六进制值（范围 \u0000 ~ \UFFFF 即 0 ～ 65535），字长为 2 byte 表示 Unicode BMP 中单个代码单元。

##### boolean

boolean 类型有两个值: false 和 true，用来判定逻辑条件。<u>整型值和布尔值之间不能进行相互转换</u>，所占空间的大小没有明确指定，仅定义为能够取字面值 true 或 false

只能赋予 true 或 false 值，并测试为真还是为假，而不能将布尔值进行任何运算

##### 数值类型转换

使用两个数进行二元操作时，较小单位会隐式转换为较大单位再计算，结果为较大单位。允许使用 () 操作符进行强制类型转换，但可能丢失信息。

* 布尔类型不允许进行任何类型的转换处理
* float 或 double 转型位整型值时，总是对数字执行截尾

##### 自动装开箱

基本数据类型值不是一个对象，可用包装类来包装成一个对象，根据上下文环境，基本数据类型值可以使用包装类自动转换成一个对象，将基本类型值转换为包装类的过程为装箱，将包装类转换为基本类型为开箱，支持自动装开箱

基本类型对应装箱类型：

* 包装类没有无参构造方法，所有包装类都是不可变的，一旦创建该对象，它们内部值就不能再改变
* 每一个数值包装类都有常量 **MAX_VALUE** 和 **MIN_VALUE** 表示对象基本数据类型的最大值和最小值
* 数值包装类中包含 compareTo 方法用于比较两个数值，如果该数值大于，等于，小于另外一个数值时，分别返回 1，0，-1

```java
Integer a = 17;
Integer b = 17;
System.out.println(a == b); // true
Integer c = 273;
Integer d = 273;
System.out.println(c == d); // false
// JVM 默认会缓存 -128 ~ 127 之间的对象，因此建议使用 equals 比较
System.out.println(c.equals(d)); // true
```

#### 语法

##### 变量

每个变量都有一个类型。在声明变量时，变量的类型位于变量名之前。大小写敏感，没有长度限制。声明一个变量之后，必须用赋值语句对变量进行赋值。不能使用未初始化的变量。

```java
// 声明变量
int size;
// 赋值
size = 32;
// 初始化
String = "jaca";
```

##### 常量

使用关键字 final 指示常量，关键字 final 表示这个变量只能被赋值一次。一旦被赋值之后，就不能再修改。习惯上常量名使用全大写蛇形。

类常量使用 static final  定义，当类常量被声明为 public 时，其他类也可以该常量​     

运算符号

元素符优先级递减

|                      运输符号                      | 运算顺序 |
| :------------------------------------------------: | :------: |
|                        [] .                        |  左到右  |
|        ！～  ++  --  ()  new  +(正)  -(负)         |  右到左  |
|                       * / %                        |  左到右  |
|                        - +                         |  左到右  |
|                    <<  >>  >>>                     |  左到右  |
|              <  <=  > >=  instanceof               |  左到右  |
|                       ==  !=                       |  左到右  |
|                         &                          |  左到右  |
|                         ^                          |  左到右  |
|                         ｜                         |  左到右  |
|                         &&                         |  左到右  |
|                        \|\|                        |  左到右  |
|                        ? :                         |  右到左  |
| =  +=  -=  *=  /=  %=  &=  \|=  ^=  <<=  >>=  >>>= |  右到左  |

##### 位运算符

###### 按位操作符

用来操作整数基本数据类型中的单个 bit，按位操作符会对两个参数中对应的位执行布尔代数运算，并最终生成一个结果。

| 操作符 |  含义  |                       描述                        |
| :----: | :----: | :-----------------------------------------------: |
|   &    | 与操作 |         两个位都是 1，输出 1，否则输出 0          |
|   \|   | 或操作 | 两个位有一个是 1：输出 1，两个位都是 0 时：输出 0 |
|   ^    |  异或  |    只有两个比较的位不同时其结果为 1，否则为 0     |
|   ~    |  取反  |     一元操作符，输入 0 输出 1，输入 1 输出 0      |

###### 移位操作符

移位操作符操作的运算对象也是二进制的 bit，只可用来处理整数类型。

| 操作符 |       含义       |                             描述                             |
| :----: | :--------------: | :----------------------------------------------------------: |
|  `<<`  |      左移位      | 按照操作符右侧指定的位数将操作符左边的操作数向左移动，低位补 0，等价翻多少倍 |
|  `>>`  | 右移位（有符号） | 按照操作符右侧指定的位数将操作符左边的操作数向右移动（若符号为证，则在高位插入0；若符号为负，则在高位插入 1）等价缩小多少被 |
| `>>>`  | 右移位（无符号） |             使用零扩展，无论正负，都在高位插入 0             |

* 如果对 char、byte、short 类型的数值进行移位处理，在移位之前，会被转换为 int 类型，结果也是 int，只有数值右端的低 5 位才有用（2^5=32，int 型只有 32 位，这样可以防止移位超过 int 型值所具有的位数）
* 若对一个 long 类型的数值进行处理，结果为 long，只会用到数值右端的低 6 位，防止移位超过 long 型数值位数

##### 操作符

##### 关键字

|    关键字    |                             含义                             |
| :----------: | :----------------------------------------------------------: |
|   abstract   |                         抽象类或方法                         |
|    assert    |                             断言                             |
|   boolean    |                           布尔类型                           |
|    break     |                    跳出一个 switch 或循环                    |
|     byte     |                           8 位整数                           |
|     case     |                         switch 分支                          |
|    catch     |                          try 块子句                          |
|     char     |                       Unicode 字符类型                       |
|    class     |                            类声明                            |
|    const     |                             保留                             |
|   continue   |                  跳出当前循环继续下一次循环                  |
|   default    |                      swich 默认条件子句                      |
|      do      |                      do/while 循环子句                       |
|    double    |                         双精度浮点数                         |
|     else     |                         if/else 子句                         |
|     enum     |                           枚举类型                           |
|   extends    |                             继承                             |
|    final     |                          不允许覆盖                          |
|   finally    |                     try 中 finally 子句                      |
|    float     |                         单精度浮点数                         |
|     for      |                             循环                             |
|     goto     |                             保留                             |
|      if      |                           条件判断                           |
|  implements  |                           实现接口                           |
|    import    |                        导入类或静态块                        |
|  instanceof  |                      测试对象是否属于类                      |
|     int      |                          32 位整形                           |
|  interface   |                           声明接口                           |
|     long     |                          64 长整形                           |
|    native    |                     由宿主系统实现的方法                     |
|     new      |                            实例化                            |
|     null     |                         null 直接量                          |
|   package    |                            声明包                            |
|   private    |                             私有                             |
|  protected   |                             保护                             |
|    public    |                             公有                             |
|    return    |                             返回                             |
|    short     |                          16 位整形                           |
|    static    |                             静态                             |
|   strictfp   | 使用精确浮点数计算模式，保证在所有的 java 虚拟机中计算结果都相同 |
|    super     |                      超类或超类对象引用                      |
|    switch    |                         代替 if/else                         |
| synchronized |                对线程而言是原子的方法或代码块                |
|     this     |                         当前对象引用                         |
|    throw     |                            抛异常                            |
|    throws    |                  声明一个方法可能抛出的异常                  |
|  transient   |                       标志非永久的数据                       |
|     try      |                       捕获异常的代码块                       |
|     void     |                         指定不返回值                         |
|   volatile   |                 确保一个域可以由多个线程访问                 |
|    while     |                             循环                             |

##### 文档注释

###### javadoc

Jdk 包含 javadoc 工具，它可以由源文件生成一个 HTML 文件。javadoc 从下面几个特性中抽取信息：

* 包
* 公有类与接口
* 公有的和受保护的构造器及方法
* 公有的和受保护的域

为上面几部分编写注释，注释放在描述特性前面，注释以 /** 开始以 */ 结束，文档注释在标记之后为自由标记文件。标记由 @ 开始，自由格式文本的第一句是一个概要性的句子，在自由格式文本中，可以使用 HTML 修饰符，与 <h1> 或 <hr> 冲突。等宽代码使用 {@code ...}

###### 注释标记

* 通用注释

  * @since

  * @deprecated

  * @see

    指定超链接

    ```java
    @see <a href="www.javadoc.com/lang/string">string page</a>
    ```

    链接类方法

    ```java
    @see com.mysql.jdbc.Result#setInt(String)
    ```

* 类注释

  类注释必须放在 import 语句之后，类定义之前

  * @author

    可以使用多个

  * @version

* 方法注释

  每一个方法注释必须放在所描述的方法之前，除了通用标记之外，可以使：

  * @param
  * @return
  * @throws

* 域注释

  只需要对公有域建立文档

* 包注释

  需要在每个包目录中添加一个单独的文件：

  * 以 package.html 命名的 HTML 文件，<body>...</body> 之间所有文本都会被抽取
  * 以 package-info.java 命名的 java 文件，在一个包语句后以 javadoc 注释，不包含其他注释和语句

###### 注释抽取

假定 HTML 文件将被存放在目录 /doc 下。执行以下步骤：

1）切换到包含想要生成文档的源文件目录。如果有嵌套的包要生成文档，如 com.horstmann.corejava，就必须切换到包含子目录 com 的目录

2）如果是一个包，执行命令：

```shell
javadoc -d doc nameOfPackage
# 或对于多个包生成文档
javadoc -d doc nameOfPackage1 nameOfPackage2
# 如果文件在默认包中，就应该运行
javadoc -d doc *.java
```

如果省略了 -d doc 选项，HTML 文件就会被提取到当前目录下。

#### 程序组成及结构