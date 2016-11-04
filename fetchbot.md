# [fetchbot](https://github.com/PuerkitoBio/fetchbot) [![build status](https://secure.travis-ci.org/PuerkitoBio/fetchbot.png)](http://travis-ci.org/PuerkitoBio/fetchbot) [![GoDoc](https://godoc.org/github.com/PuerkitoBio/fetchbot?status.png)](http://godoc.org/github.com/PuerkitoBio/fetchbot)

fetchbot包提供了一个简单、灵活的网络爬虫，遵循robots.txt策略和爬行延迟。

这是一个非常简单的 gocrawl API 改写的，更少的内置，但同时更灵活。至于go，有时少是更多！


### Fetcher

总的来说，一个 **Fetcher** 就是一个爬虫实例，和其他的 Fetcher 实例不相关。它通过 **Queue** 来接收命令并执行请求，并调用一个 **Handler** 来处理响应。一个 **Command** 是一个接口，告诉爬虫实例（**Fetcher**）哪个链接需要爬取，以及要使用的HTTP方法（“GET”，“HEAD”，...）



Fetcher.Start() 返回和一个和此Fetcher相关的队列。这是一个线程安全的对象，可以用来发送命令或者停止爬取。

**Command** 和 **Handler** 都是接口，并可以以不同的方式实现

They are defined like so:


```go
type Command interface {
	URL() *url.URL
	Method() string
}
type Handler interface {
	Handle(*Context, *http.Response, error)
}
```

**Context** 是一个struct（结构体），保存 **Command** 和 **Queue**，所以**Handler**总是知道是哪个**Command**启动了这个调用，并包含一个**Queue**的句柄

**Handler** 类似于 net/http Handler，并且可以在其上构建中间件样式组合，因为提供了 **HandlerFunc** ，所以拥有正确标志的简单函数可以用作处理程序（类似于net/http.HandlerFunc），并且还有一个多路复用器Mux，可分派给基于某些标准的不同的处理程序。

### Command 相关接口

**Fetcher** 可以识别一些由 **Command** 实现的接口，以便满足更高的需求

*  `BasicAuthProvider` ：实现此接口以指定对请求的基本认证凭证。

* `CookiesProvider`：实现此接口可以设置请求中的Cookie。

* `HeaderProvider` ：实现此接口可以指定请求中的header

* `ReaderProvider`：实现这个接口通过一个`io.Reader`设置请求的正文。

* `ValuesProvider`：实现此接口以将请求的主体设置为表单的值，如果Content-Type没有通过`HeaderProvider`特别设置，它将被设置为“application / x-www-form-urlencoded”。`ReaderProvider`和`ValuesProvider`应该是互斥的，因为它们都设置了请求的主体。 如果两者都实现，则`ReaderProvider`接口优先被使用。

*  `Handler`：如果Command的响应应该由特定的回调函数处理，请实现此接口。默认情况下，响应由Fetcher的处理程序处理，但如果Command实现此操作，则此处理程序函数优先，并且Fetcher的处理程序被忽略。

由于Command是一个接口，它可以是一个自定义结构，保存附加信息，例如URL的ID（例如来自数据库）或深度计数器，以便爬行停止在某一深度等。对于基本命令不需要额外的信息，包提供了实现Command接口的Cmd结构。 这是在使用各种Queue.SendString *方法时使用的Command实现。

对于应该由特定的回调函数处理的命令，还有一个方便的`HandlerCmd`结构。 它是一个具有Handler接口实现的命令。

### Fetcher Options

Fetcher有多个字段可提供进一步的定制

* `HttpClient` ：默认情况下，Fetcher使用net / http默认客户端进行请求。可以在Fetcher.HttpClient字段上设置不同的客户端。

* `CrawlDelay`：该值仅在指定主机的robots.txt没有指定延迟时使用。

* `UserAgent`：设置要用于请求并针对robots.txt条目进行验证的用户代理字符串。

* `WorkerIdleTTL`：设置工作程序goroutine可以等待而不接收要获取的新命令的持续时间。 如果达到空闲生存时间，则停止工作程序goroutine并释放其资源。 这对于长时间运行的抓取工具特别有用。

* `AutoClose` : 如果为true，则在活动主机数量达到0时自动关闭队列。

* `DisablePoliteness`：如果为true，则忽略主机的robots.txt策略。

What fetchbot doesn't do - especially compared to gocrawl - is that it doesn't keep track of already visited URLs, and it doesn't normalize the URLs. This is outside the scope of this package - all commands sent on the Queue will be fetched.
Normalization can easily be done (e.g. using [purell](https://github.com/PuerkitoBio/purell)) before sending the Command to the Fetcher. How to keep track of visited URLs depends on the use-case of the specific crawler, but for an example, see /example/full/main.go.

> fetchbot不能做什么 - 尤其是与gocrawl相比，它不跟踪已访问的网址，它不会规范化网址。 这不在此包的范围内，队列上发送的所有命令将被抓取。

## License

The [BSD 3-Clause license](http://opensource.org/licenses/BSD-3-Clause), the same as
the Go language. The iq package source code is under the CDDL-1.0 license (details in
the source file).

[oli]: https://github.com/oli-g
[buro9]: https://github.com/buro9
[mmcdole]: https://github.com/mmcdole
