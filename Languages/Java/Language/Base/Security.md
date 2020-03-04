##  安全

Java 技术提供了以下三种确保安全的机制

* 语言设计特性（对数组的边界进行检查，无不受检查的类型转换，无指针算法）
* 访问控制机制，用于控制代码能够执行的操作（比如文件访问，网络访问等）
* 代码签名，利用该特性，代码的作者就能够用标准的加密算法来认证 Java 代码。这样，该代码的使用者就能够准确地知道谁创建了该代码，以及代码被标识后是否被修改过

### 类加载器

编译器会为虚拟机转换源指令。虚拟机代码存储在以 .class 为扩展名的类文件中，每个类文件都包含某个类或者接口的定义和实现代码。这些类文件必须由一个程序进行解释，该程序能够将虚拟机的指令集翻译成目标机器的机器语言

#### 类加载过程

虚拟机只加载程序执行时所需要的类文件。假定程序从 MyProgram.class 开始运行，虚拟机执行的步骤如下：

1）虚拟机有一个用于加载类文件的机制，如，从磁盘上读取文件或者请求 Web 上的文件；它使用该机制来加载 MyProgram 类文件中的内容

2）如果 MyProgram 类拥有类型为另一个类的域，或者是拥有超类，那么这些类文件也会被加载（加载某个类所依赖的所有类的过程称为类的解析）

3）接着，虚拟机执行 MyProgram 中的 main 方法（它是静态的，无需创建类的实例）

4）如果 main 方法或者 main 调用的方法要用到更多的类，那么接下来就会加载这些类

类加载机制并非只使用单个的类加载器。每个 Java 程序至少拥有三个类加载器：

* 引导类加载器
* 扩展类加载器
* 系统类加载器（即应用类加载器）

引导类加载器负责加载系统类（通常从 JAR 文件 rt.jar 中进行加载）。它是虚拟机不可分割的一部分，而且通常是用 C 语言来实现的。引导类加载器没有对应的 ClassLoader 对象。

扩展类加载器用于从 jre/lib/ext 目录加载『标准的扩展』。可以将 JAR 文件放入该目录，这样即使没有任何类路径，扩展类加载器也可以找到其中的各个类

系统类加载器用于加载应用类。它在由 CLASSPATH 环境变量或者 -classpath 命令行选项设置的类路径中的目录里或者是 JAR/ZIP 文件里查找这些类

在 Oracle 的 Java 语言实现中，扩展类加载器和系统类加载器都是用 Java 来实现的。都是 URLClassLoader 类的实例。

如果将 JAR 文件放入 jre/lib/ext 目录中，并且在它的类中有一个类需要调用系统类或者扩展类，那么就会遇到麻烦，因为扩展类加载器并不使用类路径。

#### 类加载器的层次结构

类加载器有一种父、子关系。除了引导类加载器外，每个类加载器都有一个父类加载器。根据规定，类加载会为它的父类加载器提供一个机会，以便加载任何给定的类，并且只有在其父类加载器加载失败时，它才会加载该给定类。（当要求系统类加载器加载一个系统类（如，java.util.ArrayList）时，它首先要求扩展类加载器进行加载，该扩展类加载器则首先要求引导类加载器进行加载。引导类加载器会找到并加载 rt.jar 中的这个类，而无须其他类加载器做更多的搜索）

某些程序具有插件架构，其中代码的某些部分是作为可选的插件打包的。如果插件被打包为 JAR 文件，可以直接用 URLClassLoader 类的实例去加载这些类

每个线程都有一个对类加载器的引用， 称为上下文类加载器。主线程的上下文类加载器是系统类加载器。当新线程创建时，它的上下文类加载器会被设置成为创建该线程的上下文类加载器。如果不做任何特殊的操作，那么所有线程就都会将它们的上下文类加载器设置为系统类加载器。也可以通过下面的调用将其设置为任何类加载器

```java
Thread t = Thread.currentThread();
t.setContextClassLoader(loader);
// 然后助手方法可以获取这个上下文类加载器
Thread t = Thread.currentThread();
ClassLoader loader = t.getContextClassLoader();
Class cl = loarder.loadClass(classNmae);
```

当上下文类加载器设置为插件类加载器时，问题依旧存在。应用设计者必须作出决策：通常，当调用由不同的类加载器加载的插件类的方法时，进行上下文类加载器的设置是一种好的思路；或者，让助手方法的调用者设置上下文类加载器。

#### 将类加载器作为命名空间

在一个正在执行的程序中，所有的类名都包含它们的包名。然而，在同一个虚拟机中，可以有两个类，它们的类名和包名都是相同的。类是由它的全名和类加载器来确定的。这项技术在加载来自多处的代码时很有用。

#### 编写自己的类加载器

可以编写自己的用于特殊目的的类加载器，使得可以在向虚拟机传递字节码之前执行定制的检查。如果要编写自己的类加载器，只需要继承 ClassLoader 类，然后覆盖下面这个方法

```java
findClass(String className);
```

ClassLoader 超类的 loadClass 方法用于将类的加载操作委托给其父类加载器去进行，只有当该类尚未加载并且父类加载器也无法加载该类时，才调用 findClass 方法。

如果要实现该方法，必须做到以下几点：

1）为来自本地文件系统或者其他来源的类加载其字节码

2）调用 ClassLoader 超类的 defineClass 方法，向虚拟机提供字节码

#### 字节码校验

当类加载器将新加载的 JAVA 平台类的字节码传递给虚拟机时，这些字节码首先要接受校验器的校验。校验器负责检查哪些指令无法执行的明显有破坏性的操作。除了系统类外，所有的类都有被校验

常用的校验器执行的一些检查

* 变量要在使用之前进行初始化
* 方法调用与对象引用类型之间要匹配
* 访问私有数据和方法的规则没有被违反
* 对本地变量的访问都落在运行时堆栈内
* 运行时堆栈没有溢出

如果以上这些检查中任何一条没有通过，那么该类就被认为遭到了破坏，并且不予加载

### 安全管理器与访问权限

一旦某个类被加载到虚拟机中，并由检验器检查过之后，JAVA 平台的第二种安全机制就启动，这个机制就是安全管理器

#### 权限检查

安全管理器是一个负责控制具体操作是否允许执行的类。安全管理器负责检查的操作包括以下内容：

* 创建一个新的类加载器
* 退出虚拟机
* 使用反射访问另一个类的成员
* 访问本地文件
* 打开 socket 连接
* 启动打印作业
* 访问系统剪贴板
* 访问 AWT 事件队列
* 打开一个顶层窗口

### 数字签名

#### 消息摘要

消息摘要是数据块的数字指纹。（SHA1可将任何数据块，无论其数据有多长，都压缩为 160 位（20字节）的序列，因为只存在 2^160 个 SHA1指纹，所以肯定会有某些消息具有相同的指纹）

消息摘要具有两个基本属性

1）如果数据的 1 位或者几位改变了，那么消息摘要也将改变

2）拥有给定消息的伪造者不能创建与原消息具有相同摘要的假消息

Java 编程语言已经实现了 `MD5`、`SHA-1`、`SHA-256`、`SHA-384`、`SHA-512`。`MessageDigest` 类是用于创建封装了指纹算法的对象的「工厂」，它的静态方法 `getInstance` 返回继承了 `MessageDigest` 类的某个类的对象。即，`MessageDigest` 类能够承担下面的双重职责

* 作为一个工厂类
* 作为所有消息摘要算法的超类

```java
 MessageDigest messageDigest = MessageDigest.getInstance(algorithm);
byte[] hashString = messageDigest.digest(origin);
StringBuilder hBuilder = new StringBuilder();
int n;
for (byte b : hashString) {
    n = b;
    if (n < 0) {
        n += 256;
    }
    if (n < 16) {
        hBuilder.append("0");
    }
    hBuilder.append(Integer.toHexString(n));
}
return hBuilder.toString();
```

#### 消息签名

如果消息改变了，那么改变后的消息的指纹与原消息的指纹将不匹配。如果消息和它的指纹是分开传送的，那么接收者就可以检查消息是否被篡改过。但是，如果消息和指纹同时被截获了，对消息进行修改，再重新计算指纹，就是一件很容易的事情。比较，消息摘要算法是公开的，不需要使用任何密钥。在这种情况下，假消息和新指纹的接收者永远不会知道消息已经被篡改。

公共密钥加密技术是基于公共密钥和私有密钥这两个基本概念的。它的设计思想是可以将公共密钥告诉给任何人，但是只有自己才持有私有密钥。这些密钥之间存在一定的数学关系，但几乎不可能用一个密钥去推算出另一个密钥。

#### 校验签名

`JDK` 配有一个 `keytool` 程序，该程序是一个命令行工具，用于生成和管理一组证书 。`keytool` 程序负责管理密钥库、证书数据库和私有/公有密钥对。密钥库中的每一项都有一个别名。

```shell
// 创建一个密钥库
keytool -genkeypair -keystore alice.certs -alias alice
```

当新建或者打开一个密钥库时，系统将提示输入密钥库口令。`keytool` 工具使用 `X.500` 格式的名字，它包含常用名，机构单位，机构，地点，州，和国家，以确定密钥持有者和证书发行者的身份。

导出证书文件：

```shell
keytool -exportcert -keystore alice.certs -alias alice -file alice.cer
```

打印证书：

```shell
keytool -printcert -file alice.cer
```

导入证书到密钥库中

```shell
keytool -importcert -keystore bob.certs -alias alice -file alice.cer
```

绝对不要将并不完全信任的证书导入到密钥库中。一旦证书添加到密钥库中，使用密钥库的任何程序都会认为这些证书可以用来对签名进行校验

`jarsigner` 工具负责对 `JAR` 文件进行签名和校验

使用 `jarsigner` 工具将签名添加到文件中，必须指定要使用的密钥库、JAR 文件和密钥的别名

```shell
jarsigner -keystore alice.certs document.jar alice
```

使用 `jarsigner` 对文件进行校验

```shell
jarsigner -verify -keystore bob.certs document.jar
```

#### 认证问题

任何人都可以生成一对公共密钥和私有密钥，再用私有密钥对消息进行签名，然后把签名好的消息和公共密钥发送给你。这种确定发送者身份的问题称为认证问题。解决这个认证问题的通常做法是，中间人证明，将受信任中间人的私有签名应用于陌生人的公共密钥文件之上即可。当你拿到公共密钥文件之后，就可以检验中间人签名是否真实。

### 加密

当信息通过认证之后，该信息本身是直白可见的。数字签名只不过负责检验信息有没有被篡改过。相比之下，信息被加密后，是不可见的，只能用匹配的密钥进行解密

#### 对称密码

「Java密码扩展」包含了一个 `Cipher` 类，该类是所有加密算法的超类。通过调用 `getInstance` 方法可以获得一个密码对象

`JDK` 中是由名为 `SunJCE` 的提供商提供密码的，如果没有指定其他提供商，则会默认为该提供商。如果要使用特定的算法，而对该算法 `Oracle` 公司没有提供支持，那么也可以指定其他的提供商。算法名称是一个字符串，比如 `AES` 或 `DES/CBC/PKCS5Padding`

DES，即数据加密标准，是一个密钥长度为 56 位的古老分组密码。DES 加密算法在现在可以用穷举法将它破译。更好的选择是采用它的后续版本，即高级加密标准（AES）。

```java
Cipher cipher = Cipher.getInstance(algorithName);
// 或者
Cipher cipher = Cipher.getInstance(algorithName, providerName);
// 一旦获得了一个密码对象，就可以通过设置模式和密钥来对它初始化
// 模式有: Cipher.ENCRYPT_MODE 、Cipher.DECRYPT_MODE、Cipher.WRAP_MODE、Cipher.UNWRAP_MODE。wrap 和 unwrap  模式会用一个密钥对另一个密钥进行加密。
cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(Crypt.key.getBytes(StandardCharsets.UTF_8), "AES"));
// 加密
byte[] result = cipher.doFinal(content.getBytes(StandardCharsets.UTF_8));
            // return byte2hex(result);
return (java.util.Base64.getEncoder()).encodeToString(result);
```

反复调用 `update` 方法来对数据块进行加密

```java
int blockSize = cipher.getBlockSize();
byte[] inBytes = new byte[blockSize];
int outputSize = cipher.getOutputSize(blockSize);
byte[] outBytes = new byte[outputSize];
int outLength = cipher.update(intBytes, 0, outputSize, outBytes);
```

完成上述操作后，还必须调用一个 `doFinal` 方法。如果还有最后一个输入数据块（其字节数小于 `blockSize`)，那么就要调用：

```java
outBytes = cipher.doFinal(inBytes, 0, inLength);
```

如果所有的输入数据都已经加密，则用下面的方法调用来代替:

```java
outBytes = cipher.doFinal();
```

对 `doFinal` 的调用是必须的，因为它会对最后的块进行“填充“。对于 `DES` 密码来说，它的数据块的大小是 8 字节。假设输入数据的最后一个数据库少于 8 字节，可以将其余的字节全部用 0 填充，从而得到一个 8 字节的最终数据块，然后对它进行加密。但是，当对数据块进行解密时，数据块的结尾会附加若干个 0 字节，因此它与原始输入文件之间会略有不同。需要一个填充方案来避免这个问题。常用的填充方案是 `PKCS5`。在该方案中，最后一个数据块不是全部用填充值 0 进行填充，而是用等于填充字节数量的值作为填充值进行填充。

在该方案中，最后一个数据块不是全部用填充值 0 进行填充，而是用等于填充字节数量的值作为填充值进行填充。

*pkcs5填充规则*

![](../Images/pkcs5填充.png)

#### 密钥生成

为了加密，需要生成密钥。每个密码都有不同的用于密钥的格式，需要确保密钥的生成是随机的。这需要遵循下面的步骤

1）为加密算法获取 `keyGenerator`

2）用随机源来初始化密钥发生器。如果密码块的长度是可变的，还需要指定期望的密码块长度

3）调用 `generateKey` 方法

```java
KeyGenerator keygen = KeyGenerator.getInstance("AES");
SecureRandom random = new SecureRandom();
keygen.init(random);
Key key = keygen.generateKey();
```

或者，可以从一组固定的原生数据中生成一个密钥，这是可以使用如下的 `SecretKeyFactory`

```java
byte[] keyData = ...; // 16 bytes for aes
SecretKey key = new SecretKeySpec(keyData, "AES");
```

如果要生成密钥，必须使用 “真正的随机” 数。如在 `Random` 类中的常规的随机数发生器。是根据当前的日期和时间来产生随机数的，因此它不够随机。假设计算机时钟可以精确到 1 / 10 秒，那么，每天最多存在 864000 个种子。如果攻击者知道发布密钥的日期（通常可以由消息日期或证书有效日期推算出来），那么就可以很容易地产生那一天所有可能的种子。

`SecureRandom` 类产生的随机数，远比由 `Random` 类产生的那些数字安全得多。仍然需要提供一个种子，以便在一个随机点上开始生成数字序列。

```java
SecureRandom secrand = new SecureRandom();
byte[] b = new byte[20];
secrand.setSeed(b);
```

如果没有为随机数发生器提供种子，那么它将通过启动线程，使它们睡眠，然后测量它们被唤醒的准备时间，以此来计算自己的 20 个字节的种子

#### 密码流

`JCE` 库提供了一组使用便捷的流类，用于对流数据进行自动加密或解密。

```java
// 对文件数据进行加密的方法
Cipher cipher = ...;
cipher.init(Cipher.ENCRYPT_MODE, key);
CipherOutputStream out = new CipherOutputStream(new FileOutputStream(outputFileName), cipher);
byte[] bytes = new byte[BLOCKSIZE];
int inLength = getData(bytes);
while (inLength != -1) {
    out.write(bytes, 0, inLength);
    inLength = getData(bytes);
}
out.flush();
```

使用 `CipherInputStream` ，对文件的数据进行读取和解密

```java
Cipher cipher = ...;
cipher.init(Cipher.DECRYPT_MODE, key);
CipherInputStream in = new CipherInputStream(new FileInputStream(inputFileName), cipher);
byte[] bytes = new byte[BLOCKSIZE];
int inLength = in.read(bytes);
while (inLength != -1) {
    putData(bytes, inLength);
    inLength = in.read(bytes);
}
```

密码流类能够透明地调用 `update` 和 `doFinal` 方法

#### 公共密钥密码

`AES` 密码是一种对称密码，加密和解密都使用相同的密钥。对称密码的致命缺点在于密码的分发。公共密钥密码技术解决了这个问题。在公共密钥密码中，一个密钥对，包括一个公共密钥和一个相匹配的私有密钥。所有已知的公共密钥算法的操作速度都比对称密钥算法慢得多。最常见的公共密钥算法是 `RSA` 算法.

```java
// 使用 keyPairGenerator 生成公共私有密钥
KeyPairGenerator pairgen = KeyPairGenerator.getInstance("RAS");
SecureRandom random = new SecureRandom();
pairgen.initialize(KEYSIZE, random);
KeyPair keyPair = pairgen.generateKeyPair();
Key publicKey = keyPair.getPublic();
Key privateKey = keyPair.getPrivate();
```
