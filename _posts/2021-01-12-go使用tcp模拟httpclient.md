---
layout: mypost
title: go 使用tcp模拟httpclient
categories: [算法思想]
---

可以根据[rfc1867](https://tools.ietf.org/html/rfc1867){:target="_blank"}协议，轻松改造一个满足各种情况的httpclient  
有空封装一个

````golang
//发送http请求
package main

import (
	"fmt"
	"io"
	"net"
)

func main() {
	//使用Dial建立连接
	conn, err := net.Dial("tcp", "127.0.0.1:80")
	if err != nil {
		fmt.Println("error dialing", err.Error())
		return
	}

	defer conn.Close()

	msg := "GET /api/tools/localip HTTP/1.1\r\n"
	msg += "Host:127.0.0.1\r\n"
	msg += "Connection: close\r\n"
	msg += "\r\n\r\n"

	_, err = io.WriteString(conn, msg)

	if err != nil {
		fmt.Println("write string failed", err)
		return
	}

	buf := make([]byte, 4096)

	for {
		count, err := conn.Read(buf)

		if err != nil {
			break
		}

		fmt.Println(string(buf[0:count]))
	}
}

````