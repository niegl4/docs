[TOC]

# net.Dial

## Timeout字段

```go
type Dialer struct {
	// Timeout is the maximum amount of time a dial will wait for
	// a connect to complete. If Deadline is also set, it may fail
	// earlier.
	//
	// The default is no timeout.
	//
	// When using TCP and dialing a host name with multiple IP
	// addresses, the timeout may be divided between them.
	//
	// With or without a timeout, the operating system may impose
	// its own earlier timeout. For instance, TCP timeouts are
	// often around 3 minutes.
	Timeout time.Duration
  ...
}
```

 它代表为网络连接建立完成而等待的最长时间。

开始的时间点几乎就是调用net.DialTimeout函数的那一刻。之后，时间会主要花费在解析参数network和address的值，和创建socket实例并建立网络连接。

只要在超时时间到来的那一刻，网络连接还没有建立完成，函数就会返回一个代表了I/O操作超时的错误。

在解析address的值的时候，函数会确定网络服务的IP地址，端口号等必要信息，并在需要时访问DNS服务。如果解析出的IP地址有多个，那么函数会串行或并发地尝试建立连接。无论用什么方式尝试，以最先建立成功的那个连接为准。它还会根据超时前的剩余时间，去设定针对每次连接尝试的超时时间，以便让它们都有适当的时间执行。

## KeepAlive字段

```go
type Dialer struct {
  ...
  // KeepAlive specifies the interval between keep-alive
	// probes for an active network connection.
	// If zero, keep-alive probes are sent with a default value
	// (currently 15 seconds), if supported by the protocol and operating
	// system. Network protocols or operating systems that do
	// not support keep-alives ignore this field.
	// If negative, keep-alive probes are disabled.
	KeepAlive time.Duration
  ...
}
```

它是针对TCP连接的存活探测机制，用来表示每隔多长时间发送一次探测包。它与http协议的长连接不是一个概念。



# http.Client

## Timeout字段

```go
type Client struct {
  ...
  // Timeout specifies a time limit for requests made by this
	// Client. The timeout includes connection time, any
	// redirects, and reading the response body. The timer remains
	// running after Get, Head, Post, or Do return and will
	// interrupt reading of the Response.Body.
	//
	// A Timeout of zero means no timeout.
	//
	// The Client cancels requests to the underlying Transport
	// as if the Request's Context ended.
	//
	// For compatibility, the Client will also use the deprecated
	// CancelRequest method on Transport if found. New
	// RoundTripper implementations should use the Request's Context
	// for cancellation instead of implementing CancelRequest.
	Timeout time.Duration
}
```

它表示单次http事务的超时时间。零值表示不设置超时时间。

## Transport字段

```go
type Client struct {
	// Transport specifies the mechanism by which individual
	// HTTP requests are made.
	// If nil, DefaultTransport is used.
	Transport RoundTripper
  ...
}

```

它代表了：向网络服务发送http请求，并从网络服务接收http响应的操作过程。该字段是http.RoundTripper接口类型的，该字段的RoundTrip应该实现单次http事务（或者说基于http协议的单次交互）需要的所有步骤。

初始化http.Client时，如果没有显式地为该字段赋值，那么该字段会直接使用DefaultTransport。

```go
// DefaultTransport is the default implementation of Transport and is
// used by DefaultClient. It establishes network connections as needed
// and caches them for reuse by subsequent calls. It uses HTTP proxies
// as directed by the $HTTP_PROXY and $NO_PROXY (or $http_proxy and
// $no_proxy) environment variables.
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
		DualStack: true,
	}).DialContext,
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}

type Transport struct {
  ...
	// TLSHandshakeTimeout specifies the maximum amount of time waiting to
	// wait for a TLS handshake. Zero means no timeout.
	TLSHandshakeTimeout time.Duration

  ...
	// MaxIdleConns controls the maximum number of idle (keep-alive)
	// connections across all hosts. Zero means no limit.
	MaxIdleConns int

	// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
	// (keep-alive) connections to keep per-host. If zero,
	// DefaultMaxIdleConnsPerHost is used.
	MaxIdleConnsPerHost int

	// MaxConnsPerHost optionally limits the total number of
	// connections per host, including connections in the dialing,
	// active, and idle states. On limit violation, dials will block.
	//
	// Zero means no limit.
	MaxConnsPerHost int

	// IdleConnTimeout is the maximum amount of time an idle
	// (keep-alive) connection will remain idle before closing
	// itself.
	// Zero means no limit.
	IdleConnTimeout time.Duration

	// ResponseHeaderTimeout, if non-zero, specifies the amount of
	// time to wait for a server's response headers after fully
	// writing the request (including its body, if any). This
	// time does not include the time to read the response body.
	ResponseHeaderTimeout time.Duration

	// ExpectContinueTimeout, if non-zero, specifies the amount of
	// time to wait for a server's first response headers after fully
	// writing the request headers if the request has an
	// "Expect: 100-continue" header. Zero means no timeout and
	// causes the body to be sent immediately, without
	// waiting for the server to approve.
	// This time does not include the time to send the request header.
	ExpectContinueTimeout time.Duration

  ...
}
```

- **IdleConnTimeout**：空闲的连接在多久之后应该关闭。如果是零值，表示不关闭空闲的连接。

  DefaultTransport设置为90s。

- ResponseHeaderTimeout：从客户端把请求完全递交给操作系统到从操作系统接收到响应报文头的最大时长。

  DefaultTransport没有设置该字段。

- **ExpectContinueTimeout**：在客户端想要使用http的post方法把一个很大的报文体发送给服务端的时候，它可以先发送一个包含了“Expect: 100-continue“请求报文头，来询问服务端是否愿意接收这个大报文体。这个字段就是用于设定在这种情况下的超时时间的。如果是零值，表示无论多大的请求报文体都将会被立即发送出去。

  DefaultTransport设置为1s。

- **TLSHandshakeTimeout**：TLS握手阶段的超时时间。如果是零值，表示对这个时间不设限制。

  DefaultTransport设置为10s。

- **MaxIdleConns**：对于该客户端访问的，所有的网络服务的，所有网络连接，允许的空闲连接的总数。

  DefaultTransport设置为100。

- **MaxIdleConnsPerHost**：对于该客户端访问的，每一个网络服务的，所有网络连接，允许的空闲连接数。默认为2。

- MaxConnsPerHost：对于该客户端访问的，每一个网络服务的，最大连接数，无论连接是否是空闲的。如果是零值，表示不设限。

  DefaultTransport没有设置该字段。