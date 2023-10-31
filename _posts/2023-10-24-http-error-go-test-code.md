---
layout: post
title: "HTTP error emulator"
description: "HTTP error emulator"
date: 2023-10-24
tags: [HTTP]
---


# 1. 模拟 HTTP 上游服务器不响应 (网关 504 超时)

以下 Go 程序作为源站

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
)

func handleConnection(c net.Conn) {
	fmt.Printf("Serving %s\n", c.RemoteAddr().String())
	for {
		_, err := bufio.NewReader(c).ReadString('\n')
		if err != nil {
			fmt.Println(err)
			return
		}
	}
	c.Close()
}

func main() {
	arguments := os.Args
	if len(arguments) == 1 {
		fmt.Println("Please provide a port number!")
		return
	}

	PORT := ":" + arguments[1]
	l, err := net.Listen("tcp4", PORT)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer l.Close()

	for {
		c, err := l.Accept()
		if err != nil {
			fmt.Println(err)
			return
		}
		go handleConnection(c)
	}
}
```

nginx 配置

```conf
location /http_504 {
    proxy_connect_timeout 1s;
    proxy_read_timeout 3s;
    proxy_pass http://127.0.0.1:9999;
}
```

# 2. 模拟上游服务器发送不完整的 HTTP 头部， 网关（502）

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
	"time"
)

func handleConnection(c net.Conn) {
	fmt.Printf("Serving %s\n", c.RemoteAddr().String())
        reader := bufio.NewReader(c)
	for {
		line, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println(err)
			return
		}

		if line == "\r\n" {
			break
		}
	}
        resp := "HTTP/1.1 200 OK\r\n" +
                   "Date: Tue, 24 Oct 2023 13:13:05 GMT\r\n" +
                   "Content-Type: text/css\r\n" +
                   "Content-Length: 31022\r\n" +
                   "Connection: keep-alive\r\n" +
                   "Server: openresty+\r\n" +
                   "Req-ID: 000000800004577246e8004b\r\n" +
                   "Accept-Ranges: bytes\r\n"

        c.Write([]byte(resp))

	time.Sleep(time.Second)
	c.Close()
}

func main() {
	arguments := os.Args
	if len(arguments) == 1 {
		fmt.Println("Please provide a port number!")
		return
	}

	PORT := ":" + arguments[1]
	l, err := net.Listen("tcp4", PORT)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer l.Close()

	for {
		c, err := l.Accept()
		if err != nil {
			fmt.Println(err)
			return
		}
		go handleConnection(c)
	}
}
```

nginx 配置

```config
location /http_502 {
    proxy_connect_timeout 1s;
    proxy_read_timeout 3s;
    proxy_pass http://127.0.0.1:9998;
}
```

# 3. 模拟客户的提前断开连接

以下 Go 程序作为客户的

```go
package main

import (
	"net"
	"os"
	"fmt"
	"time"
)

func main() {

	arguments := os.Args
	if len(arguments) == 1 {
		fmt.Println("Please provide a port number!")
		return
	}

	servAddr := arguments[1]

	tcpAddr, err := net.ResolveTCPAddr("tcp", servAddr)
	if err != nil {
		println("ResolveTCPAddr failed:", err.Error())
		os.Exit(1)
	}

	conn, err := net.DialTCP("tcp", nil, tcpAddr)
	if err != nil {
		println("Dial failed:", err.Error())
		os.Exit(1)
	}

	reqdata := "GET /http_499 HTTP/1.1\r\nHost: test.com\r\nUser-Agent: curl/7.76.1\r\nAccept: */*\r\n\r\n"
	_, err = conn.Write([]byte(reqdata))
	if err != nil {
		println("Write to server failed:", err.Error())
		os.Exit(1)
	}

	println("write to server = ", reqdata)

        time.Sleep(time.Second * 1)
	conn.Close()
}
```

nginx 配置

```conf
location /http_499 {
    lua_check_client_abort on;
    content_by_lua_block {
        ngx.sleep(2)
        ngx.say("Hello world")
    }
}
```

# 4. 模拟 HTTP 服务器，返回指定的文件

以下 Go 程序作为源站

```go
package main

import (
	"fmt"
	"io/ioutil"
	"net"
	"os"
)

func main() {
	arguments := os.Args
	if len(arguments) != 3 {
		fmt.Printf("usage: %s addr file", arguments[0])
		return
	}

	servAddr := arguments[1]
	file := arguments[2]

	l, err := net.Listen("tcp", servAddr)

	if err != nil {
		fmt.Println("Error listening:", err.Error())
		os.Exit(1)
	}
	defer l.Close()
	fmt.Println("Listening on " + servAddr)
	for {
		conn, err := l.Accept()
		if err != nil {
			fmt.Println("Error accepting: ", err.Error())
			os.Exit(1)
		}
		go handleRequest(conn, file)
	}
}

// Handles incoming requests.
func handleRequest(conn net.Conn, file string) {
	buf := make([]byte, 1024)
	_, err := conn.Read(buf)
	if err != nil {
		fmt.Println("Error reading:", err.Error())
	}

	content, err := ioutil.ReadFile(file)
	if err != nil {
		fmt.Println("读取文件时发生错误:", err)
		return
	}

	conn.Write(content)
	conn.Close()
}
```
