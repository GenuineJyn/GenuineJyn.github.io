# Go Http Redirect详解及微服务网关对Redirect的支持

## 1.前言

在网关实践过程遇到了Redirect的需求，网关实现的http反向代理跟标准库net/http/httputil/reverseproxy.go类似, 都没有支持Redirect，从后端返回http status 3xxx意味着调用者需要采取进一步的操作，反向代理作为调用者应该具备Redirect的逻辑，Go的标准库中的http.Client也做了支持，Go1.8区别于之前的版本实现，到现在的go1.12的实现是一致的，本文的主要内容：。


> * 1. **本文对Go1.12 http.Client/Server Redirect实现的源码进行简单的梳理说明；**
> * 2. **介绍网关的http反向代理对Redirect支持考虑；**


3xx状态码具体含义参考这里：【[HTTP状态码#3xx重定向](https://zh.wikipedia.org/wiki/HTTP%E7%8A%B6%E6%80%81%E7%A0%81#3xx)】


## 2.Http Redirect


### 2.1 http.Client执行流程简单梳理
本节先大致梳理一下http.Client执行流程进行简单的梳理，在标准库net/http/client.go中可以看到RoundTripper Interface定义，所有的实现都是基于RoundTripper的实现，RoundTripper的实现可以看看Transport代码实现，之前已经较为详细的介绍过，可以参考这里【[Golang标准库源码阅读 - net/http.Transport](http://genuinejyn.top/2019/05/28/Golang%E6%A0%87%E5%87%86%E5%BA%93%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB-net-http.Transport/)】

Client的提供Get/Post/Put/Head等等方法最后都会归结到Do上，而Do的实现最终也会走到如下代码：

```
// file: net/http/client.go
640         var didTimeout func() bool
641         if resp, didTimeout, err = c.send(req, deadline); err != nil {
```

```
167 // didTimeout is non-nil only if err != nil.
168 func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err     error) {
169     if c.Jar != nil {
170         for _, cookie := range c.Jar.Cookies(req.URL) {
171             req.AddCookie(cookie)
172         }
173     }
174     resp, didTimeout, err = send(req, c.transport(), deadline)
175     if err != nil {
176         return nil, didTimeout, err
177     }
178     if c.Jar != nil {
179         if rc := resp.Cookies(); len(rc) > 0 {
180             c.Jar.SetCookies(req.URL, rc)
181         }
182     }
183     return resp, nil, nil
184 }
```
从`c.transport()`可以看到，Client.Transport是可以自定义的，不然就是DefaultTransport。

```
193 func (c *Client) transport() RoundTripper {
194     if c.Transport != nil {
195         return c.Transport
196     }
197     return DefaultTransport
198 }
```

## 2.2 Redirect细节

### 2.2.1 http.Server如何返回的
Server我们可能会采用不同的http框架，例如常用的gin，最后一般都会回归到标准库，来看一下server端是如何返回Redirect的, 重要逻辑代码如下：

```
// file: net/http/server.go

2051 func Redirect(w ResponseWriter, r *Request, url string, code int) {
		......
2087     h := w.Header()
2088
2089     // RFC 7231 notes that a short HTML body is usually included in
2090     // the response because older user agents may not understand 301/307.
2091     // Do it only if the request didn't already have a Content-Type header.
2092     _, hadCT := h["Content-Type"]
2093
2094     h.Set("Location", hexEscapeNonASCII(url))
2095     if !hadCT && (r.Method == "GET" || r.Method == "HEAD") {
2096         h.Set("Content-Type", "text/html; charset=utf-8")
2097     }
2098     w.WriteHeader(code)
2099
2100     // Shouldn't send the body for POST or HEAD; that leaves GET.
2101     if !hadCT && r.Method == "GET" {
2102         body := "<a href=\"" + htmlEscape(url) + "\">" + statusText[code] + "</a>.\n"
2103         fmt.Fprintln(w, body)
2104     }
2105 }
```
Server在返回结果中会把redirect内容添加到Http的`Location`头部字段，在采取Redirect操作之前能看到大概下面这个样子：

```
Content-Type: text/html; charset=utf-8
Location: http://www.google.com
Date: Tue, 16 Jul 2019 04:28:09 GMT
Content-Length: 56

<a href="http://www.google.com">Moved Permanently</a>
```

在Server端代码这里看到一个：`RedirectHandler` 的简单封装，可以将指定的请求转发到指定的url中，可以用来设置response的Location头部字段，使用比较方便，设定url和status code即可.

```
2125 // Redirect to a fixed URL
2126 type redirectHandler struct {
2127     url  string
2128     code int
2129 }
2130
2131 func (rh *redirectHandler) ServeHTTP(w ResponseWriter, r *Request) {
2132     Redirect(w, r, rh.url, rh.code)
2133 }
2134
2135 // RedirectHandler returns a request handler that redirects
2136 // each request it receives to the given url using the given
2137 // status code.
2138 //
2139 // The provided code should be in the 3xx range and is usually
2140 // StatusMovedPermanently, StatusFound or StatusSeeOther.
2141 func RedirectHandler(url string, code int) Handler {
2142     return &redirectHandler{url, code}
2143 }
```

### 2.2.2 http.Client是如何Redirect的

回归到Do()看一下http.Client是如何接收到Response后进行Redirect的，主要逻辑代码如下：

```
// file: net/http/client.go
514 func (c *Client) do(req *Request) (retres *Response, reterr error) {

526     var (
527         deadline      = c.deadline()
528         reqs          []*Request
529         resp          *Response
530         copyHeaders   = c.makeHeadersCopier(req) // 【标注点 7】
531         reqBodyClosed = false // have we closed the current req.Body?
532
533         // Redirect behavior:
534         redirectMethod string
535         includeBody    bool
536     )
		......
554     for {
555         // For all but the first request, create the next
556         // request hop and replace req.
557         if len(reqs) > 0 {	// 【标注点 2】
558             loc := resp.Header.Get("Location")
		......
577             ireq := reqs[0]
578             req = &Request{
579                 Method:   redirectMethod,
580                 Response: resp,
581                 URL:      u,
582                 Header:   make(Header),
583                 Host:     host,
584                 Cancel:   ireq.Cancel,
585                 ctx:      ireq.ctx,
586             }
587             if includeBody && ireq.GetBody != nil {
588                 req.Body, err = ireq.GetBody() // 【标注点 5】
589                 if err != nil {
590                     resp.closeBody()
591                     return nil, uerr(err)
592                 }
593                 req.ContentLength = ireq.ContentLength
594             }
595
596             // Copy original headers before setting the Referer,
597             // in case the user set Referer on their first request.
598             // If they really want to override, they can do it in
599             // their CheckRedirect func.
600             copyHeaders(req)	// 【标注点 8】
601
602             // Add the Referer header from the most recent
603             // request URL to the new one, if it's not https->http:
604             if ref := refererForURL(reqs[len(reqs)-1].URL, req.URL); ref != "" {
605                 req.Header.Set("Referer", ref)
606             }
607             err = c.checkRedirect(req, reqs)  // 【标注点 6】
608
609             // Sentinel error to let users select the
610             // previous response, without closing its
611             // body. See Issue 10069.
612             if err == ErrUseLastResponse {
613                 return resp, nil
614             }
		......
636         }	 // end of if len(reqs) > 0【标注点 3】
637
638         reqs = append(reqs, req)
639         var err error
640         var didTimeout func() bool
641         if resp, didTimeout, err = c.send(req, deadline); err != nil {【标注点 1】
642             // c.send() always closes req.Body
643             reqBodyClosed = true
644             if !deadline.IsZero() && didTimeout() {
645                 err = &httpError{
646                     // TODO: early in cycle: s/Client.Timeout exceeded/timeout or context cancelation    /
647                     err:     err.Error() + " (Client.Timeout exceeded while awaiting headers)",
648                     timeout: true,
649                 }
650             }
651             return nil, uerr(err)
652         }
653
654         var shouldRedirect bool
655         redirectMethod, shouldRedirect, includeBody = redirectBehavior(req.Method, resp, reqs[0]) // 【标注点 4】
656         if !shouldRedirect {
657             return resp, nil
658         }
659
660         req.closeBody()
661     }
662 }
```

在for的死循环中，请求进来的时候【标注点2】【标注点3】之间代码没有执行，先执行了【标注点1】代码向后端发送请求，拿到结果,【标注点4】代码判断`shouldRedirect`,正常情况通信请求就结束了。`shouldRedirect`置位，则需要回去重新构造请求，【标注点5】`GetBody`在【[Golang标准库源码阅读 - net/http.Transport](http://genuinejyn.top/2019/05/28/Golang%E6%A0%87%E5%87%86%E5%BA%93%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB-net-http.Transport/)】文档最后有细节描述，这里不再介绍。循环的正常退出是通过checkRedirect或者不需要继续Redirect实现的，后续会详细介绍。下面分别看一下Redirect的一些细节实现。

### 2.2.3 转发行为判断 redirectBehavior
当拿到Response后【标注点4】redirectBehavor先进行是否需要Redirect的行为判断，redirectBehavor的代码如下：

```
419 // redirectBehavior describes what should happen when the
420 // client encounters a 3xx status code from the server
421 func redirectBehavior(reqMethod string, resp *Response, ireq *Request) (redirectMethod string, should    Redirect, includeBody bool) {
422     switch resp.StatusCode {
423     case 301, 302, 303:
424         redirectMethod = reqMethod
425         shouldRedirect = true
426         includeBody = false
427
428         // RFC 2616 allowed automatic redirection only with GET and
429         // HEAD requests. RFC 7231 lifts this restriction, but we still
430         // restrict other methods to GET to maintain compatibility.
431         // See Issue 18570.
432         if reqMethod != "GET" && reqMethod != "HEAD" {
433             redirectMethod = "GET"
434         }
435     case 307, 308:
436         redirectMethod = reqMethod
437         shouldRedirect = true
438         includeBody = true
439
440         // Treat 307 and 308 specially, since they're new in
441         // Go 1.8, and they also require re-sending the request body.
442         if resp.Header.Get("Location") == "" {
443             // 308s have been observed in the wild being served
444             // without Location headers. Since Go 1.7 and earlier
445             // didn't follow these codes, just stop here instead
446             // of returning an error.
447             // See Issue 17773.
448             shouldRedirect = false
449             break
450         }
451         if ireq.GetBody == nil && ireq.outgoingLength() != 0 {
452             // We had a request body, and 307/308 require
453             // re-sending it, but GetBody is not defined. So just
454             // return this response to the user instead of an
455             // error, like we did in Go 1.7 and earlier.
456             shouldRedirect = false
457         }
458     }
459     return redirectMethod, shouldRedirect, includeBody
460 }
```
从redirectBehavor可以看到仅对状态码301，302，303，307，308进行处理。

状态码 | http.Status
------|-------------
301   |StatusMovedPermanently
302   |StatusFound
303   |StatusSeeOther
307   |StatusTemporaryRedirect
308   |StatusPermanentRedirect


* 对于301、302、303的状态码， 接下来Redirect请求会将Request Method转换成GET, 而且Body为空；后端服务返回的时候无论是gin框架还是别的http框架，都是不需要携带body，标准库会自动生成,参见：`2.2.1节的描述`，不同版本的标准在这里处理稍微有区别，也不应该设定Context-Type。 
* 对于307、308状态码，接下来Redirect Request的Method没有变化，与原始保持一致，也继续使用原来的body内容来发送转发请求。

### 2.2.4 转发策略  checkRedirect

如果需要转发，在Do的for循环中则开始重新构造请求，并且要有转发策略的考虑，参考【标注点6】, checkRedirect的代码如下：

```
409 // checkRedirect calls either the user's configured CheckRedirect
410 // function, or the default.
411 func (c *Client) checkRedirect(req *Request, via []*Request) error {
412     fn := c.CheckRedirect
413     if fn == nil {
414         fn = defaultCheckRedirect
415     }
416     return fn(req, via)
417 }
```
http.Client提供了CheckRedirect用于设定转发策略，如果没有指定则使用默认策略：`defaultCheckRedirect `。

```
728 func defaultCheckRedirect(req *Request, via []*Request) error {
729     if len(via) >= 10 {
730         return errors.New("stopped after 10 redirects")
731     }
732     return nil
733 }
```
defaultCheckRedirect最多10次转发， 避免出现过多转发或者不小心的死循环，如果这个默认策略难以满足需求，那就自己实现`CheckRedirect`。

Client如何禁止Redirect呢？其实实现自己的CheckRedirect即可实现。

### 2.2.5 Redirect Header处理
构造请求的时候Header基于可能的安全考虑，Header也会做特定的处理，参考【标注点7】【标注点8】，下面来看一下makeHeadersCopier的代码。

```
664 // makeHeadersCopier makes a function that copies headers from the
665 // initial Request, ireq. For every redirect, this function must be called
666 // so that it can copy headers into the upcoming Request.
667 func (c *Client) makeHeadersCopier(ireq *Request) func(*Request) {
668     // The headers to copy are from the very initial request.
669     // We use a closured callback to keep a reference to these original headers.
670     var (
671         ireqhdr  = ireq.Header.clone()
672         icookies map[string][]*Cookie
673     )
674     if c.Jar != nil && ireq.Header.Get("Cookie") != "" {
675         icookies = make(map[string][]*Cookie)
676         for _, c := range ireq.Cookies() {
677             icookies[c.Name] = append(icookies[c.Name], c)
678         }
679     }
680
681     preq := ireq // The previous request
682     return func(req *Request) {
683         // If Jar is present and there was some initial cookies provided
684         // via the request header, then we may need to alter the initial
685         // cookies as we follow redirects since each redirect may end up
686         // modifying a pre-existing cookie.
687         //
688         // Since cookies already set in the request header do not contain
689         // information about the original domain and path, the logic below
690         // assumes any new set cookies override the original cookie
691         // regardless of domain or path.
692         //
693         // See https://golang.org/issue/17494
694         if c.Jar != nil && icookies != nil {
695             var changed bool
696             resp := req.Response // The response that caused the upcoming redirect
697             for _, c := range resp.Cookies() {
698                 if _, ok := icookies[c.Name]; ok {
699                     delete(icookies, c.Name)
700                     changed = true
701                 }
702             }
703             if changed {
704                 ireqhdr.Del("Cookie")
705                 var ss []string
706                 for _, cs := range icookies {
707                     for _, c := range cs {
708                         ss = append(ss, c.Name+"="+c.Value)
709                     }
710                 }
711                 sort.Strings(ss) // Ensure deterministic headers
712                 ireqhdr.Set("Cookie", strings.Join(ss, "; "))
713             }
714         }		// 【标注点 9】
715
716         // Copy the initial request's Header values
717         // (at least the safe ones).
718         for k, vv := range ireqhdr {
719             if shouldCopyHeaderOnRedirect(k, preq.URL, req.URL) { 【标注点10】
720                 req.Header[k] = vv
721             }
722         }
723
724         preq = req // Update previous Request with the current request
725     }
726 }
```
【标注点9】之前的代码在恢复原来的Cookie，每次redirect会删除上次的Redirect造成的变动，再恢复原始的请求的Cookie, 但是也有例外情况，继续看【标注点10】代码。

```
887 func shouldCopyHeaderOnRedirect(headerKey string, initial, dest *url.URL) bool {
888     switch CanonicalHeaderKey(headerKey) {
889     case "Authorization", "Www-Authenticate", "Cookie", "Cookie2":
890         // Permit sending auth/cookie headers from "foo.com"
891         // to "sub.foo.com".
892
893         // Note that we don't send all cookies to subdomains
894         // automatically. This function is only used for
895         // Cookies set explicitly on the initial outgoing
896         // client request. Cookies automatically added via the
897         // CookieJar mechanism continue to follow each
898         // cookie's scope as set by Set-Cookie. But for
899         // outgoing requests with the Cookie header set
900         // directly, we don't know their scope, so we assume
901         // it's for *.domain.com.
902
903         ihost := canonicalAddr(initial)
904         dhost := canonicalAddr(dest)
905         return isDomainOrSubdomain(dhost, ihost)
906     }
907     // All other headers are copied:
908     return true
909 }
```

下次Redirect之前其他字段是要恢复到上次请求的，对于"Authorization", "Www-Authenticate", "Cookie", "Cookie2"字段只有下次Redirect的域名是子域的时候才会被copy，注意这里要好好理解一下指针和闭包。

到这里就把server端和client端中涉及到http Redirect相关的内容梳理了一遍。

## 3.网关http反向代理对Redirect的支持
微服务网关以反向代理的角色存在于调用者和后端服务之间，隔断了Redirect，因此就需要对Redirect进行特殊的考虑，下面简单介绍一下网关http反向代理对Redirect支持的实现内容。
### 3.1 场景描述
由于客户端调用和后端服务之间多了一层反向代理，因此，对于Redirect的需求的场景也与原始的Redirect存在的区别，使用到的场景大致分为两种场景：迭代方式和旁路方式【本文定义的专有名词】。

#### 迭代方式

后端服务发送回Redirect给网关的Http ReverseProxy，希望反向代理继续向后端服务发送请求，例如：后端服务是follower-leader的情况，某些情况到达follower的请求需要redirect到leader；
![RedirectIterator](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/redirect_iterator.png)
需要特别注意：需要支持多次Redirect迭代遍历，借鉴上面讲述的http.Client的checkRedirect；

#### 旁路方式
后端服务发送回Redirect，希望绕过http ReverseProxy，由调用方完成Redirect，例如：静态资源的访问，后端服务返回了静态资源的位置，由调用方再去拉取资源；
![RedirectBypass](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/redirect_bypass.png)

### 3.2 特殊实现点简介
网关http ReverseProxy支持了上述两种方式，后端服务返回通过`c.Writer.Header().Set("X-ReverseProxy-Redirect", "True")`来指定使用网关进行迭代Redirect，默认是旁路方式，旁落方式比较简单，这里不过多介绍，迭代方式实现借鉴了http.Client实现内容，简单介绍一下迭代方式实现的2个特殊考虑点。

1. 遍历方式需要满足一定的规范，例如，如果携带Body或者Post请求，后端服务要使用307/308，考虑到golang http.Request.Body只能读取一次，如果需要重复读取就需要缓存以便再次构造，作为网关性能是非常重要，绝大部分情况下是不需要Redirect的，更大概率上Redirect不携带Body，因此没必要所有的请求都缓存Body，增加一次内存拷贝，基于此，如果选用迭代方式Redirect的时候需要携带Body，那就需要后端服务返回的时候Body携带用户Redirect的Request的Body内容，以便网关的反向代理在Redirect的时候使用这部分信息进行重新构造Request的Body。307与302重定向有所区别的地方在于，收到307响应码后，客户端应保持请求方法不变向新的地址发出请求，301-303如果请求的方法不是GET或HEAD，Redirect的时候会统一转换为GET请求。
2. 迭代方式Location http ReverseProxy提供多种形式以便进行服务发现，不同互联网大厂可能使用不同的服务发现方式；

```
XxxScheme://{XxxNamingService}{path}?{queries}
http://{host}{path}?{queries}
```
对应具体的服务发现方式，网关的http ReverseProxy会进行服务发现和负载均衡然后redirect。
