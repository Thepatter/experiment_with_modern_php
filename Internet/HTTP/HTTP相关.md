### HTTP 相关

#### 域名

域名是一个有层次的结构，是一串用 . 分隔的多个单词，最右边的被称为『顶级域名』，然后是『二级域名』，层级关系向左依次降低。最左边的是主机名，通常用来表明主机的用途，如 www 表示提供万维网服务，mail 表示提供邮件服务。在 Apache、Nginx 这样的 Web 服务器里，域名可以用来标识虚拟主机，决定由那个虚拟主机来对外提供服务。

##### 域名解析

DNS 核心系统是一个三层的树状、分布式服务，基本对应域名的结构：

* 根域名服务器 Root DNS Server

    管理顶级域名服务器，返回 com 、net、cn 等顶级域名服务器的 IP 地址

* 顶级域名服务器 Top-level DNS Server

    管理各自域名下的权威域名服务器，如 com 顶级域名服务器可以返回 apple.com 域名服务器的 IP 地址

* 权威域名服务器 Authoritative DNS Server

    管理自己域名下主机的 IP 地址，如 apple.com 权威域名服务器可以返回 www.apple.com 的 IP 地址

<u>nginx 中的 resolver 指令，就是用来配置 DNS 服务器的，如果没有它，那么 nginx 就无法查询域名对应的 IP，也就无法反向代理到外部的网站</u>

```nginx
# 指定 Google 的 DNS，缓存 30s
resolver 8.8.8.8 valid = 30s
```

#### MIME

##### 定义

Multipurpose Internet Mail Extensions 是一个很大的标准规范，但 HTTP 只取了其中的一部分，用来标记 body 的数据类型，即 MIME type，MIME 把数据分成了八大类，每个大类下再细分多个子类，形式是 type/subtype 的字符串。HTTP 里经常遇到的类别

###### 数据类型

在 TCP/IP 协议栈里，传输数据基本上都是 header + body 的格式。但 TCP、UDP 是传输层协议，不会关心 body 数据是什么，只要把数据发送到对方就算完成任务。而 HTTP 是应用层协议，数据到达之后还需要告诉上层应用这是什么数据才行，即数据类型

*MIMIE 分类*

|    类型     |              描述              |                             示例                             |
| :---------: | :----------------------------: | :----------------------------------------------------------: |
|    text     | 文件是普通文本，理论上人类可读 |       text/plain, text/html, text/css, text/javascript       |
|    image    |      某种图像，不包括视频      |              image/gif、image/png、image/x-icon              |
|    audio    |          某种音频文件          |   audio/midi、audio/mpeg、audio/webm、audio/ogg、audio/wav   |
|    video    |          某种视频文件          |                    video/webm、video/ogg                     |
| application |         某种二进制数据         | application/octet-stream、application/pkcs12、application/pdf |

对于 text 文件类型若没有特定的 subtype，就使用 `text/plain`；二进制文件没有特定或已知的 subtype，即使用 `application/octet-stream`（不透明的二进制数据）

###### MIME 嗅探

在缺失 MIME 类型或客户端认为文件设置了错误的 MIME 类型时，浏览器可能会通过查看资源来进行 MIME 嗅探。每一个浏览器在不同的情况下会执行不同的操作。因为这个操作会有一些安全问题，有的 MIME 类型表示可执行内容而有些是不可执行内容。浏览器可以通过请求头 `Content-Type` 来设置 `X-Content-Type-Options`以阻止 MIME 嗅探。

`X-Content-Type-Options` HTTP 消息头相当于一个提示标志，被服务器用来提示客户端一定要遵循在 `Content-Type` 首部中对 MIME 类型的设定，而不能对其进行修改。这就禁用了客户端的 MIME 类型嗅探行为

```
# 请求类型是"style" 但是 MIME 类型不是 "text/css",请求类型是"script" 但是 MIME 类型不是  JavaScript MIME 类型时，请求将被阻止
X-Content-Type-Options: nosniff
```

##### 头字段

###### mulitpart/form-data

multipart 标识细分领域的文件类型的种类，对应不同的 MIME 类型。

可用于 HTML 表单从浏览器发送信息给服务器。作为多部分文档格式，它由边界线（`--`）划分出不同的部分组成。每一部分有自己的实体，以及自己的 HTTP 请求头。Content-Disposition 和 Content-Type 用于文件上传领域，最常用的 Content-Length 因为边界线作为分隔符而被忽略

###### Content-Disposition 

响应头指示回复的内容该以何种形式展示，是以**内联**的形式（即网页或者页面的一部分），还是以**附件**的形式下载并保存到本地。

消息头最初是在 MIME 标准中定义的，HTTP 表单及[`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST) 请求只用到了其所有参数的一个子集。只有`form-data`以及可选的`name`和`filename`三个参数可以应用在 HTTP 场景中

在 multipart/form-data 类型的应答消息体中， **`Content-Disposition`** 消息头可以被用在 multipart 消息体的子部分中，用来给出其对应字段的相关信息。各个子部分由在[`Content-Type`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type) 中定义的**分隔符**分隔。用在消息体自身则无实际意义

*   作为消息主体中的消息头

    在 HTTP 场景中，第一个参数是 inline（默认），标识回复中的消息体会以页面的一部分或整个页面的形式展示。attachment（意味着消息体应该被下载到本地；大多数浏览器会呈现一个保存为的对话框，将 filename 的值预填为下载后的文件名）

    ```
    Content-Disposition: inline
    Content-Disposition: attachment
    Content-Disposition: attachment; filename="filename.jpg"
    ```

*   作为 multipart body 中的消息头

    在 HTTP 场景中，第一个参数总是固定不变的 form-data；附加的参数不区分大小写，并且拥有参数值，参数名与参数值用 `=` 连接，参数值用双引号括起来，参数之间用分号`;`分割

    ```
    Content-Disposition: form-data
    Content-Disposition: form-data; name="fieldName"
    Content-Disposition: form-data; name="fieldName"; filename="filename.jpg"
    ```

    name 是表单字段名，每个字段名会对应一个子部分。在同一个字段名对应多个文件的情况下，则多个子部分共用同一个字段名，如果 name 参数值为 `_charset_`，意味着这个子部分表示的不是一个 HTML 字段，而是在未明确指定字符集信息的情况下各部分使用的默认字符集

    filename 是要传送的文件的初始名称字符串，该参数总是可选的，而且不能盲目使用：路径信息必须舍去。当 `filename` 和 `filename*` 同时出现的时候，优先采用 `filename*`，假如二者都支持

    *form-data 请求*

    ```
    POST /test.html HTTP/1.1
    Host: example.org
    Content-Type: multipart/form-data;boundary="boundary"
    
    --boundary
    Content-Disposition: form-data; name="field1"
    
    value1
    --boundary
    Content-Disposition: form-data; name="field2"; filename="example.txt"
    ```

###### multipart/byteranges

用于把部分的响应报文发送回浏览器。当发送状态码 206 Partial Content 时，这个 MIME 类型用于指出这个文件由若干部分组成。每个都有其请求范围，每个不同的部分都有 Content-Type 头说明文件类型和 Content-Range 说明其范围

###### application/x-www-form-urlencoded

数据被编码成以 `&` 分割的键值对，同时以 `=` 分隔键和值，非字母或数字的字符会被百分比编码，该类型不支持二进制数据

#### 连接管理

##### Nagle 算法

试图在发送一个分组之前，将大量 TCP 数据绑定在一起，以提高网络效率，鼓励发送全尺寸的段，只有当所有其他分组都确认之后，Nagle 算法才允许发送非全尺寸的分组。如果其他分组仍然在传输过程中，将那部分数据缓存起来，只有当挂起分组都确认，或缓存中积累了足够发送一个全尺寸分组的数据时，才会将缓存的数据发送出去

###### Nagle 算法性能问题

*   小的 HTTP 报文可能无法填满一个分组，可能会因为等待那些永远不会到来的额外数据而产生时延。
*   Nagle 算法与确认延迟之间交互存在问题，Nagle 算法会阻止数据的发送，直到有确认分组抵达为止，但确认分组自身会被延迟确认算法延迟 100 ～ 200 毫秒

###### 关闭 Nagle 算法

通过对套接字的修改关闭 Nagle 算法。为套接字设置 TCP_NODELAY 选项即可关闭 Nagle 算法

##### TIME_WAIT 累计

###### TIME_WAIT 端口耗尽

当某个 TCP 端点关闭 TCP 连接时，会在内存中维护一个小的控制块，用来记录最近所关闭连接的 IP 地址和端口号。通常会维持 2MSL（2 分钟），以确保这段时间内不会创建具有相同地址和端口的新连接（主要防止在两分钟内创建、关闭并重新创建两个具有相同 IP 地址和端口号的连接）

客户端每次连接到服务器时，都会使用一个新的源端口，以实现连接的唯一性。由于可用源端口的数量有限，而且在 2MSL 内连接是无法重用的，连接率就被限制在（可用端口数）/ 120

##### HTTP 连接优化

###### 1.0 Keep-Alive

实现 HTTP/1.0 keep-alive 连接的客户端可以通过包含 Connection:Keep-Alive 首部请求将一条连接保持在打开状态。如果服务器愿意为下一条请求将连接保持在打开状态，就在响应中包含相同的首部（Connection:Keep-Alive），如果不包含则客户端认为服务器不支持 keep-alive，会接收报文后关闭连接。

可以用 Keep-Alive 通用首部（使用该首部，必须包含 Connection:Keep-Alive 首部）显示 keep-Alive 属性

###### 1.1 Keep-Alive

1.1 使用持久连接取代了 keep-Alive。默认激活，要在事务处理结束后关闭连接，必须显式添加 Connection:close 首部。允许在持久连接上可选的使用请求管道。在响应到达之前，可将多条请求放入队列。当第一条请求通过网络流向服务器时，后续请求也可以开始发送，在高时延网络条件下，可以降低网络的环回时间，提高性能。

#### WebSocket

WebSocket 是『全双工』的通信协议，与 TCP 一样，客户端和服务器都可以随时向对方发送数据，采用二进制帧结构，算法，语义与 HTTP 完全不兼容。服务器发现方面，WebSocket 采用了 URI 格式：`ws` 和 `wss`，对应端口 80 和 443

##### 帧格式

###### 帧结构

*WebSocket帧结构定义*

![](../Images/WebSocket帧结构.png)

*长度不固定，最少2个字节，最多14字节*

* 开头两个字节是必须的

  *帧组成*

  |        位        |                             描述                             |        bit        |
  | :--------------: | :----------------------------------------------------------: | :---------------: |
  |       FIN        |                        消息结束标志位                        |         1         |
  |  RSV1/RSV2/RSV3  |                       保留位，必须为 0                       |         1         |
  |      opcode      | 帧类型（x0：后续帧，x1：文本帧，x2：二进制帧，x3-7 保留非控制帧，x8：关闭连接，x9：PING，xA：PONG） |         4         |
  |       MASK       |        表示载荷是否有掩码，只适用客户端发送服务器消息        |         1         |
  |   Payload len    | 载荷长度，变长编码。载荷长度由可变长字段表示，如果是 0 ~ 125，就是载荷长度；如果是 126，则后续 2 字节的 16 位无符号整数才是载荷长度；如果是 127，则后续 8 字节的 64 位无符号整数才是载荷长度。最大载荷 2^64 | 7，7 + 16，7 + 64 |
  |   Masking-key    |    载荷掩码，如果 MASK = 1 则为 4 字节随机数，否则不存在     |      0 ~ 32       |
  |  Extension data  |             x Byte，默认 0 byte，除非协商了扩展              |                   |
  | Application data |                      y Byte，剩余帧部分                      |                   |
  |   Payload data   |        （x + y）即 Extension data + Application data         |                   |

###### 分帧

是否把消息分帧由客户端和服务器实现决定，应用并不需要关注 WebSocket 帧如何分帧。RFC 6455 规定的分帧规则如下：

1.  一个未分帧的消息包含单个帧，FIN 设为 1，opcode 不为 0

2.  一个分帧的消息包含：

    *   开始帧（单个帧，FIN 设为 0，opcode 非 0）
    *   后续帧（0 个或多个帧，FIN 设为 0，opcode 设为 0）
    *   尾部帧（单个帧，FIN 设为 1，opcode 设为 0）

    一个分帧消息的有效载荷长度等于所有帧的有效载荷长度的累加。

##### WebSocket 的握手

和 TCP、TLS 一样，WebSocket 也要有一个握手过程，然后才能正式收发数据。

1.  标准的 HTTP GET 请求，带上协议升级的专用头字段：

    *请求头字段*

    |        头字段         |         值          |         描述          |
    | :-------------------: | :-----------------: | :-------------------: |
    |      Connection       |       Upgrade       |     要去协议升级      |
    |        Upgrade        |      websocket      | 升级成 WebSocket 协议 |
    |   Sec-WebSocket-key   | base64 编码的随机数 |       认证密钥        |
    | Sec-WebSocket-Version |         13          |      协议版本号       |

2.  服务器响应 `101 Switching Protocols` 报文，通知客户端

    *响应头字段*

    |        头字段        |                              值                              | 描述 |
    | :------------------: | :----------------------------------------------------------: | :--: |
    | Sec-WebSocket-Accept | 请求 Sec-WebSocket-Key 值加专用 UUID（'258EAFA5-E914-47DA-95CA-C5AB0DC85B11'）进行 sha1 摘要后转 base64 | 验证 |

3.  客户端收到响应报文，就可以用同样的算法，比较值是否相等，相等认证成功，握手完成，后续传输数据即为 WebSocket 格式二进制帧

#### data 协议

即前缀为 `data:` 协议的 URL，其允许内容创建者向文档中嵌入小文件

```
data:[<mediatype>][;base64],<data>
```

Data URLs 由四个部分组成：

*   前缀（`data:`）

*   指示数据类型的 MIME 类型

    为 MIME 类型的字符串，如果省略，则默认值为 `text/plain;charset=US-ASCII`

*   如果非文本则为可选的 base64 标记

*   数据本身

    如果数据是文本类型，可以直接将文本嵌入（根据文档类型，使用合适的实体字符或转义字符），如果是二进制数据，可以将数据进行 base64 编码后再进行嵌入

    ```
    # ascii 码 Hello world
    data:,Hello%2C%20World!
    # base64 编码的 text/plain 类型数据
    data:text/plain;base64,SGVsbG8sIFdvcmxkIQ%3D%3D
    # html 文档源码
    data:text/html,%3Ch1%3EHello%2C%20World!%3C%2Fh1%3E
    # 带 js 的 html 文档
    data:text/html,<script>alert('hi');</script>
    ```

#### CDN

##### Content Delivery Network

外部加速 HTTP 协议的服务。CDN 的核心原则是『就近访问』。主要适用缓存代理技术，使用『推』或『拉』的手段，把源站的内容逐级缓存到网络的每一个节点上。用户在上网的时候就不直接访问源站，而是访问离他最近的一个 CDN 节点（边缘节点）即缓存了源站内容的代理服务器。

在 CDN 领域里，『内容』即 HTTP 协议的『资源』，比如超文本、图片、视频、应用程序安装包。资源按照是否可缓存分为：

* 静态资源

  数据内容静态不变，任何时候来访问都是一样的，如图片、音频

* 动态资源 

  由服务器实时计算生成的。每次访问不一样。

只有静态资源才能被缓存加速、就近访问，而动态资源只能由源站实时生成，即使缓存了也没有意义。不过，如果动态资源指定了 Cache-Control，允许缓存短暂的时候，那它在这段时间里就变成了『静态资源』，可以被 CDN 缓存加速

##### CDN 负载均衡

CDN 有两个关键组成部分：**全局负载均衡**和**缓存系统**，对应的是 DNS 和缓存代理技术。

###### 全局负载均衡

全局负载均衡（Global Server Load Balance）GSLB，用于用户接入网络的时候在 CDN 专网中挑选出一个最佳节点提供服务，解决的是用户如何找到最近的边缘节点，对整个 CDN 网络进行负载均衡。

GSLB 最常见的实现方式是DNS负载均衡，未使用 CDN 时，权威 DNS 返回的是网站实际 IP 地址，浏览器收到 DNS 解析结果后直连网站。加入 CDN 后，权威 DNS 返回的不是 IP 地址，而是一个 CNAME（Canonical Name）别名记录，指向的就是 CDN 的 GSLB。因为没能获取 IP 地址，于是本地 DNS 就会向 GSLB 再发起请求，这样就进入了 CDN 的全局负载均衡系统，开始调度，依据：

* 看用户的 IP 地址，查表得到地理位置，找相对最近的边缘节点
* 看用户所在的运营商网络，找相同网络的边缘节点
* 检查边缘节点的负载情况，找负载较轻的节点
* 检查节点的服务能力，带宽，响应时间等

GSLB 会根据这些因素，用算法，找出一个最合适的边缘节点。把这个节点的 IP 地址返回给用户，用户就可以就近访问 CDN 的缓存代理

###### CDN 缓存代理

缓存系统是 CDN 的另一个关键组成部分，衡量 CDN 服务质量的指标：命中（指用户访问的资源恰好在缓存系统里，可以直接返回给用户）率和回源（缓存里没有，必须使用代理的方式回源站取）率

#### 会话

##### Cookie

是服务器发送到用户浏览器并保存在本地的一小块数据，它会在浏览器下次向同一服务器再发起请求时被携带并发送到服务器上。

```h
Set-Cookie: id=a3fWa; Expires=Wed, 21 Oct 2015 07:28:00 GMT; Secure; HttpOnly
```

###### 头字段

服务器使用 `Set-Cookie` 响应头来设置 Cookie，格式是 key=value，使用 `;` 分隔，客户代理收到响应报文，看到里面有 Set-Cookie，保存起来，下次请求时自动把这个值放进 Cookie 字段里发送给服务器。

###### Cookie 的属性

|   属性   |                             描述                             |
| :------: | :----------------------------------------------------------: |
| Expires  | 过期时间，绝对时间，即截止时间（如果不指定 Expires 或 Max-Age 属性，那么 Cookie 为会话 Cookie（session cookie，关闭浏览器失效）或内存 Cookie 在 Chrome 里过期时间会显示为 Session 或 N/A） |
| Max-Age  | 相对时间，单位为秒，浏览器收到报文时间点再加上 Max-Age 得到失效绝对时间（优先 Expires），当 Cookie 的过期时间被设定时，设定的日期和时间只与客户端相关 |
|  Domain  | 设置那些主机可以接受 Cookie，如果不指定，默认为 origin，即不包含子域名（当前大多数浏览器遵循 [RFC 6265](http://tools.ietf.org/html/rfc6265)，设置 Domain 时 不需要加前导点。否则需要加前导点，例如：`Domain=.mozilla.org`） |
|   Path   | 标识主机下的那些路径可以接受 Cookie，以 `%x2F` 作为路径分隔符，子路径也会被匹配 |
| HttpOnly | 此 Cookie 只能通过浏览器 HTTP 协议传输，浏览器 JS 引擎会禁用 document.cookie 等 API。可防止 XSS 跨站攻击 |
| SameStie | 允许服务器要求某个 cookie 在跨站请求时不会被发送，从而阻止 XSS 攻击。`None`：浏览器会在同站、跨站请求下继续发送 cookies，不区分大小写；`Strict`：浏览器将只在访问相同站点时发送 cookie；`Lax`：与 Strict 类似，但用户从外部站点导航至 URL 时除外（默认）。未设置或浏览器不支持时为 None |
|  Secure  | 表示这个 Cookie 仅能用 HTTPS 协议加密传输，明文的 HTTP 协议会禁止发送。但 Cookie 本身不是加密的，浏览器还是以明文的形式存储 |

###### 第三方 Cookie

Cookie 与域关联，如果此域与所在页面的域相同，则该 cookie 称为第一方 Cookie（first-party cookie）。如果域不同，则它是第三方 cookie（third-party cookie）

当服务器设置第一方 Cookie 时，该页面可能包含存储在其他域中的服务器上的图像或其他组件，这些图像或组件可能会设置第三方 cookie。第三方服务器可以基于同一浏览器在访问多个站点时发送给它的 cookie 建立用户浏览历史和习惯的配置文件。

#### HTTP Security

##### XSS 跨站脚本攻击

###### 原理

攻击者可以利用这种漏洞在网站上注入恶意的客户端代码，利用浏览器对文档源的信任，在展示文档时执行恶意代或重写 HTML 内容。问题在于没对用户数据进行过滤、转义

在以下情况,容易发生 XSS 攻击:

1.  数据从一个不可靠的链接进入到一个 Web 应用程序
2.  没有过滤掉恶意代码的动态内容被发送给 web 用户

恶意内容一般包括 JavaScript，有时也会包括 HTML，FLASH 或是其他浏览器可执行的代码。XSS 攻击的形式千差万别，但他们通常都会：将 cookies 或其他隐私信息发送给攻击者，将受害者重定向到由攻击者控制的网页，或是经由恶意网站在受害者的机器上进行其他恶意操作。

###### 方式

XSS 攻击可以分为3类：

*   存储型（持久型）

    注入型脚本永久存储在目标服务器上，当浏览器请求数据时，脚本从服务器上传回并执行

*   反射型（非持久型）

    当用户点击一个恶意链接，或者提交一个表单，或者进入一个恶意网站时，注入脚本进入被攻击者的网站。Web服务器将注入脚本，比如一个错误信息，搜索结果等返回到用户的浏览器上。由于浏览器认为这个响应来自可信任的服务器，所以会执行这段脚本

*   DOM 型

    通过修改原始的客户端代码，受害者浏览器的 DOM 环境改变，导致有效载荷的执行。即页面本身并没有变化，但由于DOM环境被恶意修改，有客户端代码被包含进了页面，并且意外执行
    
*   Self-XSS（社会工程学）

    黑客诱导用户复制恶意代码到浏览器 Console 执行

###### 防范

*   通过对用户输入数据进行过滤或转义（Java 使用 Jsoup 框架对字符串进行 XSS 过滤，或 Spring 下的 HtmlUtils 进行过滤）

*   前端在浏览器展示数据时，也需要使用安全的 API 进行展示（如 innerText 非 innerHTML）
*   内容安全策略

##### CSRF 跨站请求伪造

###### 原理

Corss-Site Request Forgery 也被称为 One-click-Attack，即在用户不知情的情况下，冒充用户发起请求，在用户当前已登录的 Web 应用程序上执行恶意操作。

主要是盗用用户浏览器中的登录信息，冒充用户执行操作。问题在于接口或表单没有防范不受信任的调用

###### 防范

*   CSRF Token 验证

    在表单渲染中增加 csrf-token，只有验证通过才允许提交

*   人机交互

    关键操作两步验证

##### CSP 内容安全策略

###### 原理

CSP 是一个额外的安全层，用于检测并削弱某些特定类型的攻击，包括跨站脚本 XSS 和数据注入攻击。CSP 被设计成向后兼容（除 CSP2 在向后兼容有明确提及的不一致）不支持 CSP 的浏览器也能与实现了 CSP 的服务器正常合作，不支持 CSP 的服务器也能与支持 CSP 的浏览器正常合作（浏览器会忽略，默认为网页内容使用标准的同源策略）

###### 用途

*   减少或消除 XSS 攻击

    CSP 通过指定有效域（即浏览器认可的可执行脚本的有效来源）使服务器管理者有能力减少或消除XSS攻击所依赖的载体。即限制加载内容的域

    一个 CSP 兼容的浏览器将仅执行从白名单域获取到的脚本文件，忽略所有的其他脚本 (包括内联脚本和HTML的事件处理属性)。

    作为一种终极防护形式，始终不允许执行脚本的站点可以选择全面禁止脚本执行

*   数据包嗅探攻击

    指明那种协议允许使用。

###### 实现

服务器返回 `Content-Security-Policy` 头或使用 meta 元素配置

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; img-src https://*; child-src 'none';">
# 头字段指定，禁用不安全内联/动态执行,只允许通过 https 加载资源
Content-Security-Policy: default-src https:
```

###### 头字段

*   Content-Security-Policy

    响应头，允许站点管理者控制用户代理能够为指定的页面加载哪些资源

    ```
    Content-Security-Policy: <policy-directive>; <policy-directive>
    ```

    策略由一系列策略指令所组成，每个策略指令都描述了一个针对某个特定类型资源以及生效范围的策略

    fetch 指令用来控制某些具体类型的资源可以从那些来源被加载，如果某个 fetch 指令在 CSP 头部中未定义，那么用户代理会寻找 default-src 指令的值来替代

    *fetch 指令来控制某些可能被加载的确切的资源类型的位置*

    |     指令     |                             描述                             |
    | :----------: | :----------------------------------------------------------: |
    |  child-src   |                                                              |
    | connect-src  |                 限制能通过脚本接口加载的 URL                 |
    | default-src  |                   为其他取指令提供备用服务                   |
    |   font-src   |                   设置允许勇敢 @font-face                    |
    |  frame-src   | 设置允许通过类似 <frame> 和 <iframe> 标签加载的内嵌内容的源地址 |
    |   img-src    |                    限制图片和图标的源地址                    |
    | manifest-src |                   限制应用声明文件的源地址                   |
    |  media-src   |  限制通过 <audio>,<video>,<track> 标签加载的媒体文件源地址   |
    |  object-src  | 限制 <object>,<embed>,<applet> 标签的源地址,建议设为 'none'  |
    | prefetch-src |                指定预加载或预渲染的允许源地址                |
    |  script-src  |                   限制 JavaScript 的源地址                   |
    |  style-src   |                       限制 css 文件源                        |
    |  webrtc-src  |                 指定 WebRTC 连接的合法源地址                 |
    |  worker-src  |         限制 worker,ShardWorker,ServiceWorker 脚本源         |

    *文档指令管理文档属性或 worker 环境应用的策略*

    |     指令      |                            描述                            |
    | :-----------: | :--------------------------------------------------------: |
    |   base-uri    |          限制在 DOM 中 <base> 元素可以使用的 URL           |
    | plugin-types  | 通过限制可以加载的资源类型来限制那些插件可以被嵌入到文档中 |
    |    sandbox    |       类似<iframe>,sandbox 属性,为请求的资源启用沙盒       |
    | disown-opener |             确保资源在导航的时候能够脱离父页面             |

    *导航指令管理用户能打开的链接或表单可提交的链接*

    |      指令       |                             描述                             |
    | :-------------: | :----------------------------------------------------------: |
    |   form-action   | 限制被用来作为给定上下文的表单提交的目标 URL 即 from 的 action 属性 |
    | frame-ancestors | 指定可能嵌入页面的有效父项 <frmae>,<iframe>,<object>,<embed>,<applet> |
    |  navigation-to  | 限制文档可以通过(a,from,window.location,window.open,etc.)方法的 URL |

    *报告指令控制 CSP 违规的报告过程*

    | 指令       | 描述                                       |
    | ---------- | ------------------------------------------ |
    | report-uri | 当出现可能违反 CSP 操作时,让客户端提交报告 |
    | report-to  |                                            |

    *其他指令*

    |           指令            |                             描述                             |
    | :-----------------------: | :----------------------------------------------------------: |
    |  block-all-mixed-content  |      当使用 HTTPS 加载页面时阻止使用 HTTP 加载任何资源       |
    |         referrer          | 指定会离开当前页面的跳转链接的 referer header 信息(应使用 Referrer-Policy 替代) |
    |      require-sri-for      |            需要使用 SRI 作用于页面上的脚本或样式             |
    | upgrade-insecure-requests | 让浏览器把一个网站所有的不安全 URL 当做已经被安全的 URL 链接替代 |

*   Content-Security-Policy-Report-Only

    响应头允许通过监视(但不强制执行)效果来实验策略，这些违规报告包含通过 HTTP 请求发送到指定 URI 的 JSON 文档 POST 请求

    ```
    Content-Security-Policy-Report-Only: <policy-directive>; <policy-directive>
    Content-Security-Policy-Report-Only: default-src https:; report-uri /csp-violation-report-endpoint/
    ```

    Content-Security-Policy 头指令也可应用于该指令，Content-Security-Policy 的 report-uri 指令需要和该 header 一起使用，否则不会起作用

    违规报告

    ```json
    {
      "csp-report": {
          // 发生违规的文档 URI
        "document-uri": "http://example.com/signup.html",  
          // 发生违规的文档 referrer
        "referrer": "",
          // 被内容安全政策阻塞加载的资源的 URI,如果被阻塞的 URI 于文档 URI 不同源,则被阻塞的 URI 被截断为只包含 schema,host,port,其他源则只包含来源
        "blocked-uri": "http://example.com/css/style.css",
          // 被违反的策略名
        "violated-directive": "style-src cdn.example.com",
          // CSP HTTP 头原始策略
        "original-policy": "default-src 'none'; style-src cdn.example.com; report-uri /_/csp-reports",
          // 执行或报告即是因为 CSP 头报告还是因为 CSPRO 头报告
        "disposition": "report"
      }
    }
    ```

##### HPKP 公钥锁定

###### 原理

HTTP 公钥锁定（HPKP）是一种安全功能，它告诉 Web 客户端将特定加密公钥与某个Web 服务器相关联，以降低使用伪造证书进行 MITM（中间人攻击）攻击的风险

为确保 TLS 会话中使用的服务器公钥的真实性，此公钥将包装到 X.509 证书中，该证书通常由证书颁发机构（CA）签名。诸如浏览器之类的 Web 客户端信任这些 CA，它们都可以为任意域名创建证书。如果攻击者能够攻击单个CA，则他们可以对各种TLS连接执行 MITM 攻击。 HPKP 可以通过告知客户端哪个公钥属于某个 Web 服务器来规避 HTTPS 协议的这种威胁

HPKP是首次使用信任（TOFU）技术。 Web服务器第一次通过特殊的 HTTP 头告诉客户端哪些公钥属于它，客户端会在给定的时间段内存储此信息。当客户端再次访问服务器时，它希望证书链中至少有一个证书包含一个公钥，其指纹已通过 HPKP 已知。如果服务器提供未知的公钥，则客户端应向用户发出警告

###### 启用

需要在通过 HTTPS 访问站点时返回 Public-Key-Pins 头：

```
# pin-sha256 引用的字符串是 Base64 编码的主体公钥信息指纹,当前规范要去包含第二个备用密钥
# max-age 浏览器应记住仅使用其中一个已定义的密钥访问此站点的时间(以秒为单位)
# includeSubDomains 可选,如果指定了此可选参数,则此规则也适用于所有站点的子域
# report-uri 可选,如果指定了此可选参数,则会将指纹验证失败报告给给定的 URL

Public-Key-Pins: pin-sha256="base64=="; max-age=expireTime [; includeSubDomains][; report-uri="reportURI"]

Public-Key-Pins: 
  pin-sha256="cUPcTAZWKaASuYWhhneDttWpY3oBAkE3h2+soZS7sWs="; 
  pin-sha256="M8HztCzM3elUxkcjR2S5P4hhyBNf6lHkmjAHKhpGPWE="; 
  max-age=5184000; includeSubDomains; 
  report-uri="https://www.example.org/hpkp-report"
```

###### 服务器配置

*   nginx

    ```nginx
    add_header Public-Key-Pins 'pin-sha256="base64+primary=="; pin-sha256="base64+backup=="; max-age=5184000; includeSubDomains' always;
    ```

*   apache

    ```apache
    Header always set Public-Key-Pins "pin-sha256=\"base64+primary==\"; pin-sha256=\"base64+backup==\"; max-age=5184000; includeSubDomains"
    ```

##### 浏览器同源策略

用于限制一个 origin 的文档或者它加载的脚本如何能与另一个源的资源进行交互，它能帮助阻隔恶意文档，减少可能被攻击的媒介

###### 同源定义

如果两个 URL 的协议、端口、主机都相同的话，则这两个 URL 是同源

###### 源更改

满足某些限制条件的情况下，页面可以修改它的源：

1.  脚本可以将 `document.domain` 的值设置为其当前域或其当前域的父域，如果将其设置为其当前域的父域。则这个较短的父域将用于后续源检查。端口号由浏览器另行检查，任何对 `document.domain` 的赋值操作，包括 `document.domain = document.domain` 都会导致端口号被重写为 null。

2.  使用 `document.domain` 来允许子域安全访问其父域时，需要在父域和子域中设置 `document.domain` 为相同的值。这是必要的，即使这样做只是将父域设置回其原始值。不这样做可能会导致权限错误

###### 跨源网络访问

同源策略控制不同源之间的交互，在使用 XMLHttpRequest 或 img 标签时则会受到同源策略的约束

*   跨域写操作（Cross-origin writes）

    一般是被允许，例如 links 重定向及表单提交

*   跨域资源嵌入（Cross-origin embeddin）

    一般是被允许（`<script src=""></script>`,`<link rel="stylesheet" href="">`（由于 CSS 的松散的语法规则，CSS 跨域需要设置一个正确的 HTTP 头部 Content-Type)，`<img>`、`<video>`、`<object>`、`<embed>`、`<applet>`、`@font-face` 引入的字体、`<iframe>` 载入的任何资源)

##### CORS HTTP 访问控制

跨域资源共享是一种机制，它使用额外的 HTTP 头告诉浏览器，让运行在一个 `origin domain` 上的 Web 应用被准许访问来自不同源服务器上的指定的资源。

当一个资源从与该资源本身所在的服务器不同的域、协议或端口请求一个资源时，资源会发起一个跨域 HTTP 请求。出于安全原因，浏览器限制从脚本内发起的跨源 HTTP 请求（这些 API 的 Web 应用只能从加载应用程序的同一个域请求 HTTP 资源，除非响应报文包含了正确的 CORS 头；也可能跨站请求可以正常发起，但返回结果被浏览器拦截了）

###### cors 标准

跨域资源共享标准允许在下列场景中使用跨域 HTTP 请求（XMLHttpRequest 或 Fetch 发起的跨域 HTTP 请求；Web 字体，CSS 中通过 @font-face；WebGL 贴图；使用 drawImage 将 Image/video 绘制到 canvas）

跨域资源共享标准新增了一组 HTTP 首部字段，允许服务器声明哪些源站通过浏览器有权限访问哪些资源。规范要求：<strong>对那些可能对服务器数据产生副作用，特别是 GET 以外的 HTTP 请求，或者搭配某些 MIME 类型的 POST 请求，浏览器必须首先使用 OPTIONS 方法发起一个预检请求，从而获知服务端是否允许该跨域请求，服务器确认允许后，才发起实际的 HTTP 请求，在预检请求的返回中，服务器端也可以通知客户端，是否需要携带身份凭证（包括 Cookies 和 HTTP 认证相关数据）</strong>

###### 相关头字段

|                字段                |                             用途                             | 类型      |
| :--------------------------------: | :----------------------------------------------------------: | --------- |
|               Origin               | 标识请求来自于那个站点，仅指示服务器名称，不包含路径信息。该首部用于 CORS 请求或 POST 请求。除了不包含路径信息，该字段与 Referer 首部字段相似 | 请求头    |
|   `Access-Control-Allow-Origin`    | 指示响应的资源是否被允许与给定的 origin 共享，对于不需要携带身份凭证的请求，服务器可以指定该字段的值为通配符 `*`，即允许来自所有域的请求，当指定一个域时，为了向客户端表面服务器会根据 Origin 请求头而有所不同，必须在 Vary 响应头包含 `Origin` | 响应头    |
|  `Access-Control-Request-Headers`  | 在预检请求中，用于通知服务器在真正的请求中采用那些请求头，用逗号分割请求头 | 请求/预检 |
|   `Access-Control-Allow-Headers`   |         表明服务器允许请求携带那些请求头，用逗号分割         | 响应头    |
|  `Access-Control-Request-Method`   | 用于通知服务器在真正的请求种会采用的请求方法，预检请求时，该头是必须的 | 请求/预检 |
|   `Access-Control-Allow-Methods`   |             表明服务器允许的请求方法，用逗号分割             | 响应头    |
|      `Access-Control-Max-Age`      | 表明该预检查请求响应的有效时间，在有效时间内，浏览器无须为同一请求再次发起预检请求，浏览器自身维护了一个最大有效时间，如果该首部字段的值超过了最大有效时间，将不会生效 | 响应头    |
|  `Access-Control-Expose-Headers`   | 在跨域访问时，`XMLHttpRequest.getResponseHeader()` 方法只能获取基本的响应头（Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma），如果要访问其他头，则需要服务器设置本响应头，让服务器把允许浏览器访问的头放入白名单 | 响应头    |
| `Acsess-Control-Allow-Credentials` | 指定了当浏览器的 credentials 设置为 true 时是否允许浏览器读取 reponse 的内容。当用在对预检请求的响应中时，它指定了实际的请求是否可以使用 credentials。简单 GET 请求不会被预检；如果对此类请求的响应中不包含该字段，这个响应将被忽略掉，并且浏览器也不会将相应内容返回给网页 | 响应头    |

###### 附带身份凭证的请求

XMLHttpRequest 或 Fetch 可以基于 HTTP cookies 和 HTTP 认证信息发送身份凭证。一般而言，对于跨域 XMLHttpRequest 或 Fetch 请求，浏览器不会发送身份凭证信息。如果要发送凭证信息，需要设置 XMLHttpRequest 的某个特殊标志位

```js
# foo.example 脚本向 bar 域发起一个 GET 请求，并设置 Cookie
varinvocation = new XMLHttpRequest();
var url = 'http://bar.com/resources/credentialed-content';
function callBarDomain() {
    if (invocation) {
        invocation.open('GET', url, true);
        invocation.withCredentials = true;
        invocation.onreadystatechange = handler;
        invocation.send();
    }
}
```

XMLHttpRequest 的 withCredentials 标志设置为 true，从而向服务器发送 Cookies。如果服务端响应头未包含 `Access-Control-Allow-Credentials: true`，浏览器将不会把响应内容返回给请求的发送者

对于附带身份凭证的请求，服务器不得设置 `Access-Control-Allow-Origin` 的值为 `*`，因为请求的首部中携带了 Cookie 信息，如果 `Access-Control-Allow-Origin: '*'`，请求将会失败，响应头如果携带了 Set-Cookie 字段，尝试对 Cookie 进行修改，如果操作失败，将会抛出异常。

在 CORS 响应中设置的 cookies 使用一般性第三方 cookie 策略，如果用户设置其浏览器拒绝所有第三方 cookies，将不会保存 cookie

#### Authentication

##### 通用 HTTP 认证框架

RFC 7235 定义了 HTTP 身份验证框架，服务器可以使用它来质询客户端请求，客户端可以使用它来提供身份验证信息，浏览器使用 UTF-8 编码认证信息，认证流程为：

1.  客户端请求需要身份验证的资源，服务端返回 401 响应状态，并在 WWW-Authenticate 头中说明认证方式（代理服务器返回 407 和 Proxy-Authenticate 响应头）
2.  客户端通过在 Authorization 请求头（代理使用 Proxy-Authorization 请求头将凭证提供给代理服务器）中添加凭证来进行身份验证

###### 认证头

*   响应头：WWW-Authenticate

    定义了使用何种验证方式去获取对资源的连接，通常会和一个 401 Unauthorized 的响应一同被发送

    ```
    WWW-Authenticate: <type> realm=<realm>
    Proxy-Authenticate: <type> realm=<realm>
    # type 验证类型，realm 一个保护区域的描述，如果未指定 realm, 客户端通常显示一个格式化的主机名来替代。
    WWW-Authenticate: Basic realm="Access to the staging site"
    ```

*   请求头：Authorization

    请求头含有服务器用于验证用户代理身份的凭证，通常会在服务器返回 401 状态及 WWW-Authenticate 消息头之后在后续请求中发送此消息头

    ```
    # type 验证类型，credentials 如果使用 basic 会用冒号分割用户名密码并进行 base64 编码
    Authorization: <type> <credentials>
    Proxy-Authorization: <type> <credentials>
    Authorization: Basic YWxhZGRpbjpvcGVuc2VzYW1l
    ```

###### 认证方案

通用 HTTP 身份验证框架可以被多个验证方案使用。不同的验证方案会在安全强度以及在客户端或服务器端软件中可获得的难易程度上有所不同

###### Basic 认证

使用用户的 ID/密码作为凭证信息，并且使用 base64 算法进行编码。基本验证方案并不安全。基本验证方案应与 HTTPS / TLS 协议搭配使用。假如没有这些安全方面的增强，那么基本验证方案不应该被来用保护敏感或者极具价值的信息。

*   apache 配置 Basic 验证

    *.htaccess*

    ```shell
    AuthType Basic
    AuthName "Access to the staging site"
    AuthUserFile /path/to/.htpasswd # .htpasswd 为 ：分割的加密用户名密码
    Require valid-user
    ```

    使用 htpasswd 工具生成密文

    ```shell
    # bcrypt
    htpasswd -nbB myName myPassword
    # md5
    htpasswd -nbm myName myPassword
    # sha1
    htpasswd -nbs myName myPassword
    # crypt
    htpasswd -nbd myName myPassword
    ```

    使用 openssl-cli

    ```shell
    # md5
    openssl passwd -apr1 myPassword
    # crypt
    openssl passwd -crypt myPassword
    ```

*   nginx

    ```nginx
    location /status {
    	  auth_basic "Access to the staging site";
    	  auth_basic_user_file /etc/pass/.htpasswd;
    }
    ```



