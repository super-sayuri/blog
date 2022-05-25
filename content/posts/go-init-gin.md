---
title: "go mod包管理以及简易gin配置"
date: 2022-05-24T11:50:55+08:00
tags: ["go", "gin", "web服务", "依赖配置"]
category: "技术"
---

## Go mod

go mod是go的依赖管理工具，首先我们用go mod新建一个项目：

```bash
cd project-dir
go mod init your.package.path/name
```

随后可以在目录里找到一个`go.mod`文件，一开始它长这样：

```
module your.package.path/name

go GO_VERION
```

第一行是项目的名字，第二行是go的版本号。

## Go get

由于go mod是去中心化管理，所以在添加包的时候需要给出完整的url地址，例如我们添加一个gin的依赖：

```bash
go get -u github.com/gin-gonic/gin
```

此时我们会看到`go.mod` 文件下会出现一行require，里面会有`github.com/gin-gonic/gin v1.7.7`这样的一行条目，这就说明gin的依赖已经到项目中了，接下来在项目里可以随便使用gin里面的内容。

同时解释一下go get各个参数的意思：

* -d download，仅下载
* -v verbose，显示所有输出
* -u update，更新
* -f force，强制更新，仅在-u的情况下使用
* -t test，同时也下载测试文件
* -fix 修复
* -insecure 当包的链接不是很正常的时候使用，例如没有使用https而是使用http

## Gin 简介

gin是一个go写成的web框架，它能非常简单地生成一个web服务，非常非常简单。

上面我们已经学会如何引用gin依赖了，现在我们简单弄一个服务：

```go
// in main.go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.New()

	r.GET("/ping", func(c *gin.Context) {
		c.String(http.StatusOK, "pong")
	})

	r.Run(":8080")
}
```

随后使用`go run main.go`运行，命令行输入`curl -X GET http://localhost:8080/ping`就可以看到返回值pong了

## 进阶

### http方法
直接把上面的GET换成POST, PATCH等其他HTTP方法就可以了。

### 路由
gin可以这样设置路由：

```go
r := gin.Default()
r1 := r.Group("/r1")
r1.GET("/ping", func(c *gin.Context) {
    c.String(http.StatusOK, "r1 pong")
}) // 这里的uri地址是 /r1/ping

r1 := r.Group("/r2")
r1.GET("/ping", func(c *gin.Context) {
    c.String(http.StatusOK, "r2 pong")
}) // 这里的uri地址是 /r2/ping
```

### 路径变量，query变量，header变量

读取路径中的变量：
```go
r.GET("/param1/:path", func(c *gin.Context) {
    name := c.Param("path")
    c.String(http.StatusOK, "name")
})

r.GET("/param2/*path", func(c *gin.Context) {
    // 这里path是可选项
    name := c.Param("path")
    if name == "" {
        c.String(http.StatusOK,"Hi, stranger!")
    } else {
        c.String(http.StatusOK, "hi " + name)
    }
})

r.GET("/param1/special", func(c *gin.Context) {
    // 这里会单独处理 /param1/special 而不是进 /param1/:path的路由
    c.String(http.StatusOK, "here is a special")
})
```

读取query:
```go
// curl -X GET http://localhost/query?v1=a&v2=b
r.GET("/query", func(c *gin.Context) {
    value1 := c.DefaultQuery("v1", "a") // 读取v1的值，如果没有默认值为a
    value2 := c.Query("v2") // 读取v2的值，如果v2没有赋值这里就是""
    c.String(http.StatusOK, "query")
})
```

读取header:
```go
r.GET("/header", func(c *gin.Context) {
    value1 := c.Request.Header["token"]
    c.String(http.StatusOK, "value1")
})
```

### 如何从body拿数据

读取form:
```go
r.POST("/form", func(c *gin.Context) {
    value1 := c.PostForm("column")
    c.String(http.StatusOK, "value1")
})
```

读取json:
```go
r.POST("/json", func(c *gin.Context) {
    data, err := ioutil.ReadAll(c.Request.Body)
    err := json.unmarshall(data, &obj)
    c.String(http.StatusOK, obj)
})
```

读取文件
```go
r.POST("/file", func(c *gin.Context) {
    file, header, err := c.Request.FormFile("file")
    // --snip--
    c.String(http.StatusOK, obj)
})
```

### 返回

使用`c.JSON(code, jsonObj)`返回json格式，同理，还可以返回XML或者ProtoBuf。

如果要额外加header的话需使用`c.Header("key", "value")`。

跳转使用`c.Redirect(code, url)`。

返回文件使用`c.File(filePath)`或者`c.File(filePath, httpFileSystem)`。

### 中间件

`r.user(MiddleWareFunc)`可以使用诸如`gin.Logger()`, `gin.Recovery()`等包里给的，也可以自定义，例：

```go
// 在request之前和之后打日志
func RequestLogger(c *gin.Context) {
	reqId := uuid.NewString()
	c.Set("requestId", reqId)
	l := GetLog(c)
	l.Info("Req #", reqId, " starts.")
	// url info
	l.Info("Calling ", c.Request.Method, " ", c.Request.RequestURI)
	// user info
	l.Info("User ip:", c.ClientIP(), ", agent:", c.Request.Header.Get("User-Agent"))
	c.Next()
	l.Info("Result: code ", c.Writer.Status(), ", size: ", c.Writer.Size())
	l.Info("Req #", reqId, " ends at ", time.Now())
}
```

使用时直接可以：

```go
func main() {
	r := gin.New()

    r.Use(RequestLogger)

	r.GET("/ping", func(c *gin.Context) {
		c.String(http.StatusOK, "pong")
	})

	r.Run(":8080")
}
```

## 特别注意：代理
gin默认是相信所有代理的，但是这并不推荐。

一般来说，项目需要使用`r.SetTrustedProxiex([]string{})`来配置可以信赖的代理ip。

同时，如果gin也提供了一些特定的cdn，可以配置`r.TrustedPlatform`来配置特定CDN的信任代理ip。
