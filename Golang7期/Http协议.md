## 写一个简单的HTTP Server

```go
package main

import (
	"fmt"
	"net/http"
)

func HelloHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello")
	fmt.Fprint(w, r.URL)
}

func main() {
	// 定义路由
	http.HandleFunc("/", HelloHandler)

	// ListenAndServe如果不发生error,那么会一直阻塞，给每一个请求开一个协程处理
	http.ListenAndServe(":5001", nil)
}	
```





## 写一个HTTP Client

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
)

func BoyHandler(w http.ResponseWriter, r *http.Request) {
	for k, v := range r.Header {
		// 打印request Header
		fmt.Printf("%s = %v\n", k, v)
	}
}

func Get() {
	resp, err := http.Get("http://localhost:5001/boy")
	if err != nil {
		fmt.Println(err.Error())
		return
	}
	io.Copy(os.Stdout, resp.Body)
	resp.Body.Close() // 一定要关闭,不然会造成协程泄漏

}

func main() {
	Get()
}
```

## 写一个Post

```go
func Post() {
	reader := strings.NewReader("Hello Server")
	resp, err := http.Post("http://localhost:5001/boys", "text/plain", reader)
	if err != nil {
		fmt.Println(err.Error())
		return
	}
	defer resp.Body.Close()
	for k, v := range r.Header {
		// 打印request Header
		fmt.Printf("%s = %v\n", k, v)
	}
}
```

## 自定义一个请求

```go
func complexHttpRequest() {
	rander := strings.NewReader("Hello Server")
	req, err := http.NewRequest("POST", "http://localhost:5002/boy", rander)
	if err != nil {
		fmt.Println(err.Error())
		return
	} else {
		// 自定义请求头
		req.Header.Add("User-Agent", "中国")
		req.Header.Add("Device", "iPhone13 Pro Max")
		// 自定义cookie
		req.AddCookie(&http.Cookie{
			Name:  "mycookie",
			Value: "123",
		})
	}
	client := &http.Client{
		Timeout: 5000 * time.Microsecond,
	}
	// 提交HTTP请求
	if res, err := client.Do(req); err != nil {
		fmt.Println(err.Error())
	} else {
		defer res.Body.Close()
	}
}
```





## Router

- `go get -u github.com/julienschmidt/httprouter`
- Router实现了`http.Handler`接口
- 为各种request、method提供了便捷的路由方式
- 支持`restful`请求方式
- 支持`ServeFiles`访问静态文件
- 可以自定义捕获`panic`方法

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"

	"github.com/julienschmidt/httprouter"
)

func handle(method string, w http.ResponseWriter, r *http.Request) {
	fmt.Println("请求方法：", r.Method)
	fmt.Println("请求头内容：", r.Header)
	io.Copy(os.Stdout, r.Body)

	fmt.Println("New")

}

func get(w http.ResponseWriter, r *http.Request, params httprouter.Params) {
	handle("GET", w, r)
}

func post(w http.ResponseWriter, r *http.Request, params httprouter.Params) {
	handle("POST", w, r)

}
func main() {
	router := httprouter.New()
	router.GET("/", get)
	router.POST("/", post)

	http.ListenAndServe(":5003", router)

}
```



## Validator

- 提供http的参数校验

- `go get -u github.com/go-playground/validator`

## Gin框架

- Gin是一款高性能的、简单轻巧的Web框架
- `go get -u github.com/gin-gonic/gin`
- 学习`gin.Context`，体会一下go中的`Context`接口的典型实现

- [官网](https://gin-gonic.com/zh-cn/)



## 简单的谈一下RESTFul规范

1. Url路径：请均使用名词复数表示

```go
GET /api/users/   # 返回所有用户信息
POST /api/users/ # 新增一个用户
GET /api/users/4 # 返回用户Id为4的用户信息
PUT /api/users/4 # 更新用户Id为4的用户信息
```

2. 请求方式
- 访问同一个URL地址,采用不同的请求方式,代表要执行不同的操作
- 用的有以下4种方式

| 请求方式 | 说明                     |
| -------- | ------------------------ |
| GET      | 获取资源数据(单个或多个) |
| POST     | 新增资源数据             |
| PUT      | 修改资源数据             |
| DELETE   | 删除资源数据             |

3. 过滤信息： 通过在`url`上传参数的形式传递查询条件,一般都是通过`?`进行拼接