# [gocrawl](https://github.com/PuerkitoBio/gocrawl) [![GoDoc](https://godoc.org/github.com/PuerkitoBio/gocrawl?status.png)](http://godoc.org/github.com/PuerkitoBio/gocrawl) [![build status](https://secure.travis-ci.org/PuerkitoBio/gocrawl.png)](http://travis-ci.org/PuerkitoBio/gocrawl)


gocrawl是一个用Go开发的友好的，微型和并发的网络爬虫。


对于一个更简单但更灵活的以更为成熟的Go风格编写的网络爬虫，您可能需要看看[fetchbot](https://github.com/PuerkitoBio/fetchbot)，这是一个基于gocrawl经验的软件包。

## Features 特点


*    完全控制要访问，检查和查询的网址（使用预先初始化的[goquery][]文档）
*    每个主机应用抓取延迟
*    服从robots.txt规则（使用[robotstxt.go][robots]库）
*    使用goroutines并发执行
*    可配置日志记录
*    开放，可定制的设计提供钩子到执行逻辑



## API

Gocrawl可以被描述为一个最小的网络爬虫（因此是“slim”标签，在〜1000 sloc），提供基本的引擎，在其上构建一个具有缓存，持久性和陈旧性检测逻辑的完整的索引机器，或使用是快速和容易爬行。 Gocrawl本身不会尝试检测页面的陈旧性，也不会实现缓存机制。 如果一个网址被排入待处理队列，它会发出一个请求来获取它（假如它被robots.txt允许 - 因此是“友好”标签）。 并且在要处理的URL之间没有优先级，它假定所有排队的URL必须在某一点被访问，并且它们的顺序是不重要的。


然而，它提供了大量的[hooks and customizations](#hc)和定制。 它不是试图做一切，并强加一种方法来做它，它提供了操纵和适应任何人的需要的方式。

As usual, the complete godoc reference can be found [here][godoc].

### Design rationale 设计原理

gocrawl的主要用例是抓取一些网页，同时遵守“robots.txt”策略的限制，同时在给每个主机的每个请求之间应用一个“好的Web公民*抓取延迟”。 因此以下设计决定：

* **每个host产生它自己的worker（goroutine）**：这是有意义的，因为它必须首先读取它的robots.txt数据，然后才按顺序，一次一个请求，在每次提取之间指定的延迟。 在主机之间没有约束，因此每个单独的worker可以独立进行爬网。


* **访问函数在worker goroutine上调用**：再次，这是确定，因为爬行延迟可能大于解析文档所需的时间，因此这种处理通常不会消耗性能。


* **支持没有爬网延迟的边缘情况，但是没有优化**：在罕见但可能的事件中，当需要没有延迟的爬行（例如：在自己的服务器上，或在繁忙时间之外获得许可等），gocrawl接受空（零）延迟，但不提供优化。也就是说，在代码中没有“特殊路径”，其中访问者函数与工作程序分离，或者可以在同一host上同时启动多个工作程序。（事实上​​，如果这种情况是你唯一的用例，我建议不要使用这个库 - 因为它没有什么价值 - ，只需使用Go的标准库并随意使用尽可能多的goroutine。）


* **Extender接口提供了写入一个完全封装的行为的方法** ：`Extender`的实现可以从根本上增强核心库，包括缓存，持久性，不同的获取策略等。这就是为什么`Extender.Start（）`方法与`Crawler.Run（）`方法有些多余，`Run`允许将crawler调用为库，而`Start`使得封装选择种子URL到`Extender`所需的逻辑成为可能。`Extender.End( )`和`Run`的返回值也是如此。



虽然它可能用于抓取大量的网页（毕竟，这是抓取，访问，入队，重复广告令人反感！），最现实的（和um ...测试！）用例应该基于一个知名的，定义明确的有限桶种子。分布式爬行是你的朋友，如果你需要移动过这种合理的使用。

### Crawler


Crawler类型控制整个执行。它生成工作程序goroutine并管理URL队列。有两个辅助构造函数：

* **NewCrawler(Extender)**：创建具有指定的`Extender`实例的搜寻器。

* **NewCrawlerWithOptions(*Options)**：创建具有预初始化的`*Options`实例的搜寻器。


唯一的公共函数是`Run (seeds interface {})error`，它接受一个种子参数（用于开始爬行的基本URL），可以用多种不同的方式表示。当没有更多的URL等待被访问，或当达到`Options.MaxVisit`时，它结束。它返回一个错误，即“ErrMaxVisits”，如果此设置是导致抓取停止的原因。

T 可用于传递种子的各种类型如下（相同的类型适用于`Extender.Start(interface {})interface {}`，`Extender.Visit(*URLContext，*http.Response ，*goquery.Document)(interface {}，bool)`和`Extender.Visited(*URLContext，interface {})`，以及`EnqueueChan`字段的类型）

*    `string` : a single URL expressed as a string  **表示为字符串的单个URL**
*    `[]string` : a slice of URLs expressed as strings  **一个以字符串表示的URL片段**
*    `*url.URL` : a pointer to a parsed URL object  **指向解析的URL对象的指针**
*    `[]*url.URL` : a slice of pointers to parsed URL objects    **一个指向解析的URL对象的指针slice**
*    `map[string]interface{}` : a map of URLs expressed as strings (for the key) and their associated state data  **表示为字符串（对于键）及其相关联的状态数据的URL的映射**
*    `map[*url.URL]interface{}` : a map of URLs expressed as parsed pointers to URL objects (for the key) and their associated state data  **表示为URL对象（对于键）的解析指针的URL的映射及其相关联的状态数据**

为了方便起见，类型`gocrawl.S`和`gocrawl.U`分别提供为字符串和URL地图的映射（例如，代码可以看起来像`gocrawl.S {"http ：//site.com"："some state data"}`）

### Options  选项

下一节详细介绍了选项类型，它提供了一个构造函数`NewOptions(Extender)`，它返回一个带有默认值和指定的`Extender`实现的初始化选项对象。

### Hooks and customizations 钩子和定制


`Options`类型提供了gocrawl提供的钩子和自定义。除了`Extender'都是可选的，并且有默认值，但是`UserAgent`和`RobotUserAgent`选项应该设置为适合你的项目的自定义值。

* **UserAgent**：用于获取页面的用户代理字符串。默认为Windows用户代理字符串上的Firefox 15。应该更改为包含对您的robot的名称和联系链接的引用（参见示例）。

*  **RobotUserAgent**：用于在robots.txt文件中查找匹配策略的robot的用户代理字符串。默认为`Googlebot（gocrawl vM.m）'，其中`M.m`是gocrawl的主要版本和次要版本。此**应始终更改为自定义值**，例如项目的名称（请参阅示例）。有关基于机器人用户代理的规则匹配的详细信息，请参阅机器人排除协议（由Google在此解释的完整规范）。如果网站所有者需要与您联系，最好在用户代理中包含联系信息。

*    **MaxVisits** : The maximum number of pages *visited* before stopping the crawl. Probably more useful for development purposes. Note that the Crawler will send its stop signal once this number of visits is reached, but workers may be in the process of visiting other pages, so when the crawling stops, the number of pages visited will be *at least* MaxVisits, possibly more (worst case is `MaxVisits + number of active workers`). Defaults to zero, no maximum.
> **MaxVisits**：限制访问的最大页数，超出停止访问。可能更有用的发展目的。请注意，一旦达到此次访问次数，抓取工具就会发送停止信号，但工作人员可能正在访问其他网页，因此当抓取停止时，访问的网页数量将至少为* MaxVisits，可能更多 （最坏情况是“MaxVisits +活跃worker数量”）。默认为0时，没有最大值。

* **EnqueueChanBuffer** : Enqueue通道的缓冲区大小（允许扩展程序随意在搜寻器中排列新URL的通道）。默认值为100。

* **HostBufferFactor**：当`SameHostOnly`设置为`false`时，工作器map和通信通道大小的因子（乘数）。 当`SameHostOnly`是`true`时，Crawler完全知道所需的大小（基于种子URL的不同主机的数量），但是当它为“false”时，大小可能呈指数增长。 默认情况下，使用10的系数（大小设置为基于种子URL的不同主机数量的10倍）。

*  **CrawlDelay**：在每个请求到同一主机之间等待的时间。 一旦从主机接收到响应，延迟就开始。 这是一个`time.Duration`类型，例如它可以用`5 * time.Second`指定（这是默认值，5秒）。 **如果robots.txt文件中指定了与robot的用户代理匹配的组中的抓取延迟，则默认使用此延迟**。 通过实现`ComputeDelay`扩展函数，可以进一步定制爬行延迟。

*  **WorkerIdleTTL** ：一个worker在清除之前允许空闲的生存时间（它的goroutine终止）。 默认为10秒。 爬网延迟不是空闲时间的一部分，这是worker可用时的具体时间，但没有要处理的网址。

* **SameHostOnly**：将网址限制为仅排入定位到同一主机的链接，默认情况下为“true”。

*  **HeadBeforeGet**：要求爬虫在发出最终GET请求之前发出HEAD请求（以及后续的“RequestGet（）`extender方法调用）。默认情况下设置为“false”。另请参见下面解释的“URLContext”结构。

* **URLNormalizationFlags**：使用purell库规范化URL时应用的标志。 URL在入队之前被标准化，并传递到`URLContext`结构中的`Extender`方法。 默认为purell允许的最积极的规范化，`purell.FlagsAllGreedy`。

*  **LogFlags**：日志记录的详细程度。默认为仅错误（`LogError`）。可以是一组标志（即“LogError | LogTrace”）。

*  **Extender**：实现Extender接口的实例。 这实现了gocrawl提供的各种回调。 必须在创建抓取工具时指定（或在创建要传递给NewCrawlerWithOptions构造函数的选项时）。 默认扩展器作为有效的默认实现DefaultExtender提供。 它可以通过嵌入它作为一个匿名字段来实现自定义扩展，当并非所有的方法都需要自定义（见上面的例子）。

### The Extender interface 扩展器接口

最后一个选项字段`Extender`是使用gocrawl的关键，所以下面是`Extender`接口所需要的每个回调函数的细节。

* **Start**：`Start(seeds interface{}) interface{}`. 当爬行器调用`Run`时调用，种子传递给`Run`作为参数。它返回将用作实际种子的数据，以便此回调可以控制爬网程序处理哪些种子。有关详细信息，请参阅[各种支持的类型](#types)。默认情况下，这是一个传递，它返回作为参数接收的数据。

*  **End** : `End(err error)`.当爬网结束时调用，带有错误或空。同样的错误也从`Crawler.Run（）`函数返回。默认情况下，此方法是无操作。

*  **Error** : `Error(err *CrawlError)`.发生抓取错误时调用。发生错误**不**停止抓取执行。一个 [`CrawlError`] [ce]实例作为参数传递。这个专门的错误实现包括 - 其他有趣的字段 - 一个“Kind”字段，指示错误发生的步骤，以及一个`*URLContext`字段，用于标识导致错误的处理的URL。 默认情况下，此方法是无操作。

* **Log** : `Log(logFlags LogFlags, msgLevel LogFlags, msg string)`. 日志功能。默认情况下，打印到标准错误（Stderr），并且只输出具有包含在`LogFlags`选项中的级别的消息。

*  **ComputeDelay** : `ComputeDelay(host string, di *DelayInfo, lastFetch *FetchInfo) time.Duration`。在请求URL之前由工作线程调用。 参数是主机名（`*url.URL.Host`的标准化形式），抓取延迟信息（包括来自选项结构，来自robots.txt的延迟和最后使用的延迟）和最后一次抓取 信息，使得可以适应当前运行的主机的响应。 它返回使用的延迟。


剩余的扩展函数都在给定URL的上下文中调用，因此它们的第一个参数始终是指向`URLContext`结构的指针。 所以在记录这些方法之前，这里是所有`URLContext`的字段和方法的解释：

* `HeadBeforeGet bool` : 此字段使用搜索器的 `Options`结构中的全局设置初始化。它可以在任何时候被覆盖，虽然有用的是它应该在调用之前完成的`Fetch`，在那里决定做出HE​​AD请求或不做。
* `State interface{}` : 此字段保存与URL关联的任意状态数据。它可以是`nil`或任何类型的值。
* `URL() *url.URL` : 以非标准化形式返回解析的网址的getter方法。
* `NormalizedURL() *url.URL` : getter方法以标准化形式返回已解析的网址。
* `SourceURL() *url.URL` : 以非标准化形式返回源网址的getter方法。可以是`nil`用于通过`EnqueueChan`入队的种子或URL。
* `NormalizedSourceURL() *url.URL` : 以规范化形式返回源网址的getter方法。可以是`nil`用于通过`EnqueueChan`入队的种子或URL。
* `IsRobotsURL() bool` : 指明当前网址是否为robots.txt网址。

有了这个方法，这里是其他`Extender`函数：

*  **Fetch** : `Fetch(ctx *URLContext, userAgent string, headRequest bool) (*http.Response, error)`. 由worker调用以请求URL。`DefaultExtender.Fetch()`实现使用公共`HttpClient`变量（一个自定义的`http.Client`）来获取没有下面重定向的页面，而是返回一个特殊的错误(`ErrEnqueueRedirect`) 将重定向到的网址入列。 这通过抓取进程获取的每个URL的`Filter()`强制实施白名单。

    在内部，gocrawl将http.Client的`CheckRedirect()`函数字段设置为自定义实现，仅跟踪robots.txt网址的重定向（因为robots.txt上的重定向仍然意味着网站所有者希望我们为此使用这些规则 主办）。 worker注意到`ErrEnqueueRedirect`错误，因此如果一个非robots.txt网址要求重定向，`CheckRedirect()`会返回这个错误，并且worker识别这个错误并将重定向的URL加入队列，停止处理 的当前URL。 可以提供基于相同逻辑的定制`Fetch()`实现。 任何返回ErrEnqueueRedirect错误的`CheckRedirect()`实现都会以这种方式工作 - 也就是说，工人将检测到此错误，并将重定向到URL入队。 有关详细信息，请参阅源文件ext.go和worker.go。

    `HttpClient`变量是public的，可以自定义它使用另一个`CheckRedirect（）`函数，或一个不同的`Transport`对象等。这个定制应该在启动爬虫之前完成。 它将被默认的`Fetch（）`实现使用，或者如果需要也可以由一个自定义的`Fetch（）`使用。 请注意，此客户端由应用程序中的所有抓取工具共享。 如果在同一个应用程序中每个搜寻器需要不同的http客户端，则应该提供使用私有http.Client实例的自定义`Fetch（）`。

*  **RequestGet** : `RequestGet(ctx *URLContext, headRes *http.Response) bool`.指示搜寻器是否应根据HEAD请求的响应继续执行GET请求。 只有在请求HEAD（基于`* URLContext.HeadBeforeGet`字段）时才调用此方法。 如果HEAD响应状态代码为2xx，则默认实现返回`true`。

* **RequestRobots** : `RequestRobots(ctx *URLContext, robotAgent string) (data []byte, request bool)`. 询问是否应抓取robots.txt网址。 如果第二个值为`false`，`data`值被认为是robots.txt缓存的内容，并且被这样使用（如果它是空的，它的行为就好像没有robots.txt一样）。 `DefaultExtender.RequestRobots`实现返回`nil，true`。

*  **FetchedRobots** : `FetchedRobots(ctx *URLContext, res *http.Response)`. 在从主机提取robots.txt网址时调用，因此可以缓存其内容并将其反馈给未来的`RequestRobots()`调用。默认情况下，不需要操作。

*  **Filter** : `Filter(ctx *URLContext, isVisited bool) bool`. 在确定是否应将某个网址排入访问时调用。 它接收`*URLContext`和`bool`“被访问”标志，指示此URL是否已在此抓取执行中访问过。 它返回一个`bool`标志命令gocrawl访问（`true`）或忽略（`false`）的URL。 即使函数返回`true`以使访问的URL入列，该URL的规范化形式仍必须遵守以下规则：
  1. 它必须是绝对URL
  2. 它必须有一个`http / https'方案
  3. 如果设置了“SameHostOnly”标志，它必须具有相同的host

  如果URL尚未被访问，则“DefaultExtender.Filter”实现返回“true”（* visited *标志基于URL的规范化版本），否则返回false。

    

*   **Enqueued** : `Enqueued(ctx *URLContext)`. 当一个URL已经被爬虫排队调用。一个排队的网址仍然可以通过一个robots.txt的政策是不允许的，所以它最终可能会*不*被取出。默认情况下，此方法是无操作。

*  **Visit** : `Visit(ctx *URLContext, res *http.Response, doc *goquery.Document) (harvested interface{}, findLinks bool)`.在访问网址时调用。 它接收URL上下文，一个`*http.Response`响应对象，以及一个现成的`*goquery.Document`对象（如果响应主体不能被解析，则为`nil`）。 它返回进程的链接（参见[上面](#types)的可能类型），和一个`bool`标志指示gocrawl是否应该自己找到链接。 当这个标志为`true`时，`harvested`返回值被忽略，gocrawl在goquery文件中搜索入列的链接。 当`false`时，`harvested`的数据被排入队列，如果有的话。 `DefaultExtender.Visit`实现返回`nil，true`，以便自动找到并处理来自访问页面的链接。

*  **Visited** : `Visited(ctx *URLContext, harvested interface{})`.访问页面后调用。 URL上下文和在访问期间（通过`Visit`函数或gocrawl）找到的URL作为参数传递。默认情况下，此方法是无操作。

*  **Disallowed** : `Disallowed(ctx *URLContext)`. 在robots.txt策略拒绝入列的网址时调用。默认情况下，此方法是无操作。

最后，按照惯例，如果一个名为`EnqueueChan`的字段具有非常特定类型的`chan < - interface {}`存在，并且可以在`Extender`实例上访问，该字段将被设置为入队通道， [预期类型](#types)作为要入列的URL的数据。 此数据随后将由爬虫处理，就好像是从访问中收获的。 它将触发对`Filter()`的调用，如果允许，将被获取和访问。

`DefaultExtender`结构有一个有效的`EnqueueChan`字段，因此如果它作为一个匿名字段嵌入到自定义Extender结构中，这个结构自动获得`EnqueueChan`功能。

这个channel对于任意排队的URL是有用的，否则不会被抓取进程处理。 例如，如果URL引发服务器错误（状态代码5xx），它可以在`Error()`extender函数中重新排队，从而尝试另一次提取。

## Thanks

* Richard Penman
* Dmitry Bondarenko
* Markus Sonderegger

## License

The [BSD 3-Clause license][bsd].

[bsd]: http://opensource.org/licenses/BSD-3-Clause
[goquery]: https://github.com/PuerkitoBio/goquery
[robots]: https://github.com/temoto/robotstxt.go
[purell]: https://github.com/PuerkitoBio/purell
[robprot]: http://www.robotstxt.org/robotstxt.html
[robspec]: https://developers.google.com/webmasters/control-crawl-index/docs/robots_txt
[godoc]: http://godoc.org/github.com/PuerkitoBio/gocrawl
[er]: http://godoc.org/github.com/PuerkitoBio/gocrawl#EndReason
[ce]: http://godoc.org/github.com/PuerkitoBio/gocrawl#CrawlError
[gotalk]: http://talks.golang.org/2012/chat.slide#33
[i10]: https://github.com/PuerkitoBio/gocrawl/issues/10
[i9]: https://github.com/PuerkitoBio/gocrawl/issues/9
[i14]: https://github.com/PuerkitoBio/gocrawl/issues/14
[i55]: https://github.com/PuerkitoBio/gocrawl/issues/55
[tmatsuo]: https://github.com/tmatsuo
