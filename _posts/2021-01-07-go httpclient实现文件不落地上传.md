---
layout: mypost
title: go httpclient 实现文件不落地上传
categories: [Golang]
---

自己封装了一个go语言的HTTP请求客户端；可以自定义HEADER信息，BODY信息；对GET，POST，PUT， DELETE等请求做了简单封装；

支持多种形式的POST请求，可以上传本地文件，或者不落地上传远程文件；

可以直接使用， 想了解详细可以直接看源码， 很简单

### Installation
go get -u  github.com/yuyu888/golibs/httpclient

### Documentation
[https://pkg.go.dev/github.com/yuyu888/golibs/httpclient](https://pkg.go.dev/github.com/yuyu888/golibs/httpclient)

### Basic Usage

````go
package main

import (
	"fmt"

	"github.com/yuyu888/golibs/httpclient"
)

func main() {
	httpcli := httpclient.NewRequest()
	postData := "a=111&b=222"
	headers := map[string]string{"Content-Type": "application/x-www-form-urlencoded", "myheader": "test"}
	resp, _ := httpcli.SetMethod("POST").SetUrl("http://127.0.0.1:8199/test?c=666").SetHeaders(headers).SetStringPostdata(postData).Send()
	fmt.Println(resp.Body)
}
````

### Get

````go
    httpcli := httpclient.NewRequest()
	resp, _ := httpcli.Get("http://127.0.0.1:8199/test?c=666")
	fmt.Println(resp.Body)
````

### Post

基于：Content-Type: application/x-www-form-urlencoded

````go
    httpcli := httpclient.NewRequest()
	postData := url.Values{"aaa": {"111"}, "id": {"1233"}}
	// postData := make(url.Values)
	// postData["name"] = []string{"xiaoming"}
	// postData["id"] = []string{"123456"}
	resp, _ := httpcli.Post("http://127.0.0.1:8199/test?c=666", postData)
	fmt.Println(resp.Body)
````

### PostForm

模拟Form表单的post， 支持上传文件，该文件可以是本地的可以是网络文件；
相关的 HTTP 协议：[rfc1867](https://tools.ietf.org/html/rfc1867){:target="_blank"}

````go
httpcli := httpclient.NewRequest()
	postData := map[string]string{
		"id":                 "123456",
		"@upload_param_name": "upload_file", // 与上传接口的上传字段明一致
		"@upload_file_path":  "https://yuyu888.github.io/static/img/logo.jpg", // 上传的文件地址， 可以是本地文件也可以是远程文件
	}
	resp, _ := httpcli.PostForm("http://127.0.0.1:8199/test?c=666", postData)
	fmt.Println(resp.Body)
````

### Put

````go
    httpcli := httpclient.NewRequest()
	postData := "a=111&b=222"
	resp, _ := httpcli.Put("http://127.0.0.1:8199/test?c=666", postData)
	fmt.Println(resp.Body)
````

### Delete

````go
    httpcli := httpclient.NewRequest()
	resp, _ := httpcli.Delete("http://127.0.0.1:8199/test?c=666")
	fmt.Println(resp.Body)
````

