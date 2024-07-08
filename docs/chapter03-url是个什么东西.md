# URL 是个什么东西？

URL（Uniform Resource Locator）是统一资源定位符，是用来定位互联网上的资源的。URL 是互联网上的一个资源的地址，可以是一个网页、图片、视频、音频等等。URL 是互联网上的一个资源的唯一标识，通过 URL 可以找到这个资源。

我们解析 URL 的时候，可以分为五个部分：协议、目标、路径、参数和锚点。

如下图：
![url](../images/url.png)

- 协议：`http` 是协议，表示资源的传输协议，如 https、ftp、file 等。
- 目标：`www.example.com` 是目标，表示资源的主机地址，它有细分为几个：

  - username：用户名
  - password：密码
  - host：主机地址，包括域名或 ip 地址。
  - port：端口号

- 路径：`/path/to/myfile` 是路径，表示资源的路径，可以是文件路径或目录路径。
- 参数：`?key1=value1&key2=value2` 是参数，表示资源的参数，可以是多个参数，参数之间用 `&` 分隔。
- 锚点：`#SomeWhereInTheDocument` 是锚点，表示资源的锚点，用于页面内的跳转。

接下来我们分别解析这五个部分。

## 协议

协议通俗的讲：它规定了服务通讯的格式和规则，也就是话术。对于一个 URL 地址来说，协议由协议名称 + 冒号组成，如：http、https、ftp、file 等。

当我们在浏览器输入一个 URL 地址时，浏览器会根据协议来选择合适的处理方式，如我们的样例： <http://www.example.com。> 浏览器会在 TCP 连接成功之后发送如下内容：

```shell
GET / HTTP/1.1
Host: www.example.com
Accept: */*
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.3
Cookie: ...
```

## 目标

目标是我们访问主机的地址，可以是 ip 地址，也可以是域名。同时，也可以附带一些认证信息，如用户名和密码。

### 域名

域名是 ip 地址的别名，是为了方便记忆而设置的。域名是由一串用点分隔的名字组成，如：`www.example.com`。域名是有层级的，从右到左依次是顶级域名、二级域名、三级域名等。 域名是一个树形结构，它从根域名开始，结构如下：

![domain](../images/dns-arc.png)

- 根域名：所有域名的起点都是根域名，它写作一个点.，放在域名的结尾。因为这部分对于所有域名都是相同的，所以就省略不写了，比如 example.com 等同于 example.com.（结尾多一个点）。

你可以试试，任何一个域名结尾加一个点，浏览器都可以正常解读。

- 顶级域名：根域名的下一级是顶级域名。它分成两种：通用顶级域名（gTLD，比如 .com 和 .net）和国别顶级域名（ccTLD，比如 .cn 和 .us）。

顶级域名由国际域名管理机构 ICANN 控制，它委托商业公司管理 gTLD，委托各国管理自己的国别域名。

- 一级域名： 这是域名的最高层级，通常由一个顶级域名（Top-level Domain，TLD）和一个域名注册商的名称组成。比如，example.com 就是在顶级域名.com + 注册的名称。

- 二级域名： 这是在一级域名之下的一个更具体的子域名，一般体现具体的职责和用途，如 mail.163.com、news.163.com 等。

- 三级域名： 这是在二级域名之下的一个更具体的子域名，一般体现具体的职责和用途，如 www.mail.163.com、news.mail.163.com 等。

说到域名，就不得不提域名解析系统（Domain Name System，DNS），它是互联网的一项核心服务，它可以将域名和 IP 地址相互映射的分布式数据库，能够使人更方便地访问互联网。

既然说到 DNS，我们不妨将它的工作原理也介绍一下。它的工作流程如下：

1. 浏览器缓存：浏览器会先在自己的缓存中查找域名对应的 IP 地址，如果有，直接返回。

对于 chrome 浏览器，你可以通过在地址栏输入 `chrome://net-internals/#dns` 查看浏览器的 DNS 缓存。

2. 系统`hosts`文件和系统缓存，首先检查域名存在`hosts`文件配置，如果存在则直接返回，否则查找系统缓存中是否有域名对应的 IP 地址，如果有，直接返回。

- `hosts`文件是一个计算机文件，它用于将 IP 地址映射到主机名。在 Windows 系统中，`hosts`文件位于`C:\Windows\System32\drivers\etc\hosts`；在 Linux 系统中，`hosts`文件位于`/etc/hosts`。

- 对于 windows 系统而言，可以通过命令`ipconfig /flushdns`清理系统 DNS 缓存。

- 对于 Mac 系统而言，可以通过命令`sudo killall -HUP mDNSResponder`清理系统 DNS 缓存。

- 对 Linux 系统而言，可以通过命令`sudo /etc/init.d/nscd restart`清理系统 DNS 缓存。
