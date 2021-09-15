---
title: go发送http请求和http服务器
date: 2021-08-31
categories: go
tags: [go, http]
description: go语言的net/http包含http客户端和http服务端，客户端主要用来发送http请求，服务端主要是监听端口处理http请求
---

## HttpClient
### GET请求
```golang
func HttpGet(url string) {
	res, err := http.Get(url)
	if err != nil {
		fmt.Printf("err:%s", err.Error())
		return
	}
	defer res.Body.Close()

	body, _ := ioutil.ReadAll(res.Body)
	fmt.Print(string(body))
}
```
### POST请求
```golang
func HttpPost(url string) {
	param := "name=tom&age=18"
	res, err := http.Post(url, "application/x-www-form-urlencoded", strings.NewReader(param))
	if err != nil {
		fmt.Printf("err:%s", err.Error())
		return
	}
	defer res.Body.Close()
	body, _ := ioutil.ReadAll(res.Body)
	fmt.Print(string(body))
}
```
### PostForm 请求 
> Content-Type 设置成了  application/x-www-form-urlencoded.

```golang 
func HttpPostForm(urlStr string) {
	form := url.Values{"foo": {"bar"}}
    // or
	// form := make(url.Values)
	// form.Set("foo", "bar")
	// form.Add("foo", "bar2")
	// form.Set("bar", "baz")
	resp, err := http.PostForm(urlStr, form)
	if err != nil {
		// handle error
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		// handle error
	}
	fmt.Println(string(body))
}
```

### 自定义Http Client

```golang 
func httpDo(urlStr string) {
	client := &http.Client{
		Timeout: 5 * time.Second,
	}

	req, err := http.NewRequest("POST", urlStr, strings.NewReader("name=cjb"))
	if err != nil {
		// handle error
	}

	req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
	req.Header.Set("Cookie", "name=anny")

	resp, err := client.Do(req)
	if err != nil {
		// handle error
	}
	defer resp.Body.Close()

	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		// handle error
	}

	fmt.Println(string(body))
}
```

## HttpServer

### 一个简单的Http服务

```golang
func main() {
	http.HandleFunc("/hello", func(rw http.ResponseWriter, r *http.Request) {
		// 请求参数处理
		r.ParseForm()
		for k, v := range r.Form {
			fmt.Printf("%s=%s\n", k, v[0])
		}
		rw.Write([]byte("hello world"))
	})

	http.ListenAndServe(":8080", nil)
}
```

### 自定义server对象的Http服务 

代码调用两个路由的处理函数：Handle和HandleFunc。 Handle的第二个参数是一个handler对象，该对象必须实现ServeHTTP方法，我们在ServeHTTP方法中完成我们的处理逻辑，显然直接使用HandleFunc要方便一些，从源码中可以看到HandleFunc最终也是调用了Handle函数完成操作
```golang
package main

import (
	"log"
	"net/http"
	"time"
)

func main() {
	server := &http.Server{
		Addr:         "127.0.0.1:8000",
		ReadTimeout:  2 * time.Second,
		WriteTimeout: 2 * time.Second,
	}
	mux := http.NewServeMux()           //声明多路复用mux对象
	mux.Handle("/", &TestHandler{"Hi"}) //根路由
	mux.HandleFunc("/login", login)     //login路由
	server.Handler = mux
	server.ListenAndServe()
}

func login(w http.ResponseWriter, r *http.Request) {
	log.Printf("login")
	w.Write([]byte(string("login")))
}

type TestHandler struct {
	str string
}

//ServeHTTP方法，绑定TestHandler
func (th *TestHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	log.Printf(th.str)
	w.Write([]byte(string(th.str)))
}
```
