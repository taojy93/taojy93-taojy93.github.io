---
title: "Golang 实现 http 服务的分析"
date: 2022-04-28T21:30:20+08:00
tags: ["GO", "GO 标准库"]
draft: false

---

## 启动一个 HTTP 服务

```go

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello, Golang")
}

// 写法一
func main() {

    http.HandleFunc("/hello", handler)

    http.ListenAndServe(":8888", nil)

}

// 写法二
func main() {

    http.HandleFunc("/", handler)

    srv := http.Server{
        Addr:    ":8888",
        Handler: nil,
    }

    srv.ListenAndServe()

}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上面第一种写法是一种简化的写法，其实底层也是调用的第二种写法的 srv.ListenAndServe()

## ListenAndServe() 处理请求的底层

```go
func (srv *Server) ListenAndServe() error {
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}


---------------------------

func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
	...
    // 不断循环取出TCP连接
	for {
        // 看我看我！！！
		rw, e := l.Accept()
        ...
        // 再看我再看我！！！
		go c.serve(ctx)
	}
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以看到每个来自客户端的请求都会起一个独立的 goroutine 去处理。

## 多路复用器

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在创建服务的时候，ListenAndServe 方法的第二个参数其实指的就是一个多路复用器，因为 Golang 实现了一个默认的 ` 多路复用器 DefaultServeMux ` ，所以我们传递 nil 的时候也可以正常使用。<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不过Golang实现的这个默认的多路复用器不是那么好，例如不支持更高级的路由功能（路由组，中间件），也需要对路由算法做更好的优化；所以很多 web框架 其实都是重新实现了更细致的多路复用器的。

## Gin对多路复用器的实现

### 实现（伪代码）

```go
package gin

import (
	"fmt"
	"net/http"
)

type HandlerFunc func(http.ResponseWriter, *http.Request)

// 多路复用器
type Engine struct {
	router map[string]HandlerFunc
}

// http.Handler interface 的实现
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	key := req.Method + "-" + req.URL.Path
	if handler, ok := engine.router[key]; ok {
		handler(w, req)
	} else {
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}

func New() *Engine {
	return &Engine{router: make(map[string]HandlerFunc)}
}

func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
	key := method + "-" + pattern
	engine.router[key] = handler
}

func (engine *Engine) GET(pattern string, handler HandlerFunc) {
	engine.addRoute("GET", pattern, handler)
}

func (engine *Engine) POST(pattern string, handler HandlerFunc) {
	engine.addRoute("POST", pattern, handler)
}

func (engine *Engine) Run(addr string) (err error) {
	return http.ListenAndServe(addr, engine)
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们可以看到上面需要实现一个 `多路复用器`，必须实现 `ServeHTTP()` 方法。因为必须实现 Handler 接口，Handler 接口如下：
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

### 使用

```go
func main() {

    engine := gin.New()
    engine.GET("/", func(c *gin.Context) {
        c.Writer.Write([]byte("hello gin framework!"))
    })

    server := http.Server{
        Addr:    ":8888",
        Handler: engine,
    }

    server.ListenAndServe()

}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上面的 engine 就是对 `多路复用器` 的实现，替代了之前的 `DefaultServeMux`。