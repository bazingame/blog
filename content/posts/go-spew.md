---
title: "调试利器 go-spew"
summary: 调试利器 go-spew
date: 2022-07-24
weight: 1
tags: ["golang"]
---

对于应用的调试，我们经常会使用 `fmt.Println`来输出关键变量的数据。或者使用 log 库，将数据以 log 的形式输出。对于基础数据类型，上面两种方法都可以比较方便的满足需求。对于一些结构体类型数据通常我们可以先将其序列化后再输出。

如果结构体中包含不可序列化的字段，比如 func 类型，那么序列化就会抛出错误，阻碍调试。

## go-spew

上面的需求，`go-spew` 可以完美的帮我们实现。`go-spew` 可以以一种非常友好的方式输出完整的数据结构信息。如：

```go
s := "GoCN"
i := 123

spew.Dump(s, i)	
//-----
(string) (len=4) "GoCN"
(int) 123
```

对于复杂的数据类型：

```go
type S struct {
	S2 *S2
	I  *int
}

type S2 struct {
	Str string
}

func main() {
	s := "GoCN"
	i := 2
	f := []float64{1.1, 2.2}
	m := map[string]int{"a": 1, "b": 2}
	e := errors.New("new error")
	ss := S{S2: &S2{Str: "xxx"}, I: &i}
	spew.Dump(s, i, f, m, e, ss)
}
//-----

(string) (len=4) "GoCN"
(int) 2
([]float64) (len=2 cap=2) {
 (float64) 1.1,
 (float64) 2.2
}
(map[string]int) (len=2) {
 (string) (len=1) "a": (int) 1,
 (string) (len=1) "b": (int) 2
}
(*errors.errorString)(0xc000010320)(new error)
(main.S) {
 S2: (*main.S2)(0xc000010330)({
  Str: (string) (len=3) "xxx"
 }),
 I: (*int)(0xc00001e1f0)(2)
}


```

可以看到，对于 map、slice、嵌套 struct 等类型的数据都可以友好的展示出来。包括长度、字段类型、字段值、指针值以及指针指向的数据等。

```go
u := &url.URL{Scheme: "https", Host: "gocn.vip"}
	req, err := http.NewRequestWithContext(context.Background(), "GET", u.String(), nil)
	if err != nil {
		panic(err)
	}

spew.Dump(req)
//-----
(*http.Request)(0xc000162000)({
 Method: (string) (len=3) "GET",
 URL: (*url.URL)(0xc000136090)(https://gocn.vip),
 Proto: (string) (len=8) "HTTP/1.1",
 ProtoMajor: (int) 1,
 ProtoMinor: (int) 1,
 Header: (http.Header) {
 },
 Body: (io.ReadCloser) <nil>,
 GetBody: (func() (io.ReadCloser, error)) <nil>,
 ContentLength: (int64) 0,
 TransferEncoding: ([]string) <nil>,
 Close: (bool) false,
 Host: (string) (len=8) "gocn.vip",
 Form: (url.Values) <nil>,
 PostForm: (url.Values) <nil>,
 MultipartForm: (*multipart.Form)(<nil>),
 Trailer: (http.Header) <nil>,
 RemoteAddr: (string) "",
 RequestURI: (string) "",
 TLS: (*tls.ConnectionState)(<nil>),
 Cancel: (<-chan struct {}) <nil>,
 Response: (*http.Response)(<nil>),
 ctx: (*context.emptyCtx)(0xc000020090)(context.Background)
})
```

上面是一个 `http.Request`类型的变量，其中包含多种复杂的数据类型，但 `go-spew`都可以友好的输出出来，非常方便我们调试。

## Dump系列函数 

`go-spew`有三个 Dump 系列函数：

- Dump()  标准输出到`os.Stdout`
- Fdump() 允许输出自定义`io.Writer`
- Sdump() 输出的结果作为字符串返回

此外，还有 Printf、 Fprintf、Sprintf 几个函数来支持定制输出风格。用法与`fmt`的函数相似。

## 自定义配置

`go-spew` 支持一些自定义配置，可以通过`spew.Config` 修改。

一些常用配置如下：

- Indent  修改分隔符
- MaxDepth 控制遍历最大深度
- DisablePointerAddresses 控制是否打印指针类型数据地址

此外还有其他很多配置，大家可以自己测试一下，这里不再举例。

## 小结

`go-spew`是一个非常完美的输出Go数据结构的调试工具，并且经过了全面的测试，测试覆盖率为100%。该工具支持各种自定义配置，非常方便，可以较大提升我们日常开发的效率。

## References

[davecgh/go-spew: Implements a deep pretty printer for Go data structures to aid in debugging (github.com)](https://github.com/davecgh/go-spew)

[ 飞雪无情的博客 (flysnow.org)](https://www.flysnow.org/2019/02/03/golang-classic-libs-go-spew.html)

