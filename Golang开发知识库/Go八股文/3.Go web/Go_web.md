# Go Web

## 你有使用过哪些Go的Web框架？介绍一下它们。

Gin是一个用于构建Web应用和API的轻量级的Go语言框架。拥有高性能和简洁的API设计。

- Gin提供了灵活而简单的路由机制，支持路径参数和通配符。通过Gin，可以轻松定义路由并处理不同的HTTP请求方法。
- Gin支持中间件，可以在请求到达处理程序之前或之后执行额外的逻辑。这使得实现日志记录、身份验证、错误处理等功能变得非常简单。
- Gin提供了简便的方法来处理JSON和XML数据。通过`c.JSON`和`c.XML`等方法，可以方便地构建HTTP响应。

## 说一下 Gin 的拦截器的原理

在 Gin 中，拦截器通常称为中间件（Middleware）。中间件允许在请求到达处理函数之前或之后执行一些预处理或后处理逻辑。Gin 的中间件机制基于 Go 的函数闭包和`gin.Context`的特性。

Gin 的中间件是通过在路由定义中添加中间件函数来实现的，这些中间件函数会在请求到达路由处理函数之前被执行。

1.中间件函数

中间件是一个函数，它接受一个 `gin.Context` 对象作为参数，并执行一些逻辑。中间件可以在处理函数之前或之后修改请求或响应。

```go
func MyMiddleware(c *gin.Context) {
    // 在处理函数之前执行的逻辑
    fmt.Println("Middleware: Before handling request")

    // 执行下一个中间件或处理函数
    c.Next()

    // 在处理函数之后执行的逻辑
    fmt.Println("Middleware: After handling request")
}
```

2.**注册中间件：**

在 Gin 中，通过 `Use` 方法注册中间件。在路由定义中使用 `Use` 方法添加中间件函数，可以对整个路由组或单个路由生效。

```go
r := gin.New()

// 注册中间件
r.Use(MyMiddleware)

// 定义路由
r.GET("/hello", func(c *gin.Context) {
    c.JSON(200, gin.H{"message": "Hello, Gin!"})
})
```

上述例子中的 `MyMiddleware` 就是一个简单的中间件，它会在处理 `/hello` 路由的请求之前和之后输出一些信息。

3.**中间件链：**

可以通过在 `Use` 方法中添加多个中间件函数，形成中间件链。中间件链中的中间件按照添加的顺序依次执行。

```go
r := gin.New()

// 中间件链
r.Use(Middleware1, Middleware2, Middleware3)

// 定义路由
r.GET("/hello", func(c *gin.Context) {
    c.JSON(200, gin.H{"message": "Hello, Gin!"})
})
```

在上述例子中，`Middleware1`、`Middleware2`、`Middleware3` 将会按照它们添加的顺序执行。

4.**中间件的执行顺序：**

中间件的执行顺序非常重要，因为它们可能会相互影响。在执行完一个中间件的逻辑后，通过 `c.Next()` 将控制权传递给下一个中间件或处理函数。如果中间件没有调用 `c.Next()`，后续中间件和处理函数将不会被执行。

```go
func MyMiddleware(c *gin.Context) {
    fmt.Println("Middleware: Before handling request")

    // 如果不调用 c.Next()，当前gin会将当前中间件函数彻底执行完后再进行后续中间件以及接口函数的执行
    // c.Next()

    fmt.Println("Middleware: After handling request")
}

// 有c.Next()时的输出
Middleware: Before handling request
// 接口函数的输出。。。
Middleware: After handling request


// 没有c.Next()时的输出
Middleware: Before handling request
Middleware: After handling request
// 接口函数的输出。。。
```

## Gin 框架内部路由查找机制是什么？为什么它比 net/http 更快？ 

Gin 框架的路由查找机制基于 **Radix Tree（压缩前缀树）** 实现，而标准库 `net/http` 使用的是一个 **线性遍历的路由表**。

在 Gin 中，每个 HTTP 方法（GET、POST 等）都有一棵独立的 Radix Tree，用于存储和查找路径节点。
当请求到来时，Gin 会：

1. 根据请求方法定位对应的 Radix Tree；
2. 从根节点开始按路径层级匹配；
3. 对参数路径（如 `/user/:id`）使用动态节点匹配；
4. 对通配符路径（如 `/static/*filepath`）使用通配节点匹配；
5. 最终直接拿到绑定的 handler。

这种树状结构可以在路径共享前缀较多的情况下，大幅减少匹配次数。
相比之下，`net/http` 的默认 `ServeMux` 是 **线性匹配**：
每次请求都会依次遍历所有注册的路由规则，找到最长前缀匹配的那一条，时间复杂度近似 O(n)。

因此：

- Gin 路由查找是 **基于树的前缀匹配**，查找效率更高；
- `net/http` 是 **顺序遍历匹配**，在路由数量增多时性能会明显下降；
- 同时 Gin 在路由构建阶段会预处理路径参数、缓存节点结构，减少运行时开销。

简单来说：
**Gin 快的原因在于使用了 Radix Tree + 路由预编译 + 减少运行时字符串匹配。**

### **补充：那么为什么不使用哈希表呢？**

理论上哈希表查找是 O(1)，但是在路由匹配这种场景下，**路径不是简单的固定字符串键值对**，而是有层级、参数、通配符等语义的结构。

**举个例子：**

- `/user/:id`
- `/user/profile`
- `/static/filepath`

**这些路径显然没法直接作为哈希表的 key 去匹配，因为：**

1. **存在动态路径参数（`:id`）：**
   **`/user/1`**、**`/user/2` **都要匹配同一个 handler。
   如果用哈希表，就得为所有可能的 id 生成 key，不现实。
2. **存在通配符匹配（`filepath`）：**
   **`/static/css/a.css`**、**`/static/js/b.js` **都走同一逻辑，哈希没法表达“前缀匹配”或“部分通配”。
3. **需要最长前缀匹配逻辑：**
   比如 **`/user/info`** 和 **`/user`**都存在时，Gin 要找到最“具体”的那条规则。
   哈希表没法表达“层级和优先级”关系。

所以 Gin 选择使用 **Radix Tree**（压缩前缀树）：

- **它既能快速匹配共享前缀（节省空间），**
- 又能灵活支持 **`:param`**、**`wildcard`**、**前缀优先**等复杂逻辑。

从性能上看，Radix Tree 查找路径的复杂度是 O(k)，其中 k 是路径层数（比如 **`/a/b/c` ** 就是 3 层），实际上在路由匹配场景下比哈希表更合适也更稳定。

## 介绍一下Go中的context包的作用

`context`可以用来在`goroutine`之间传递上下文信息，相同的`context`可以传递给运行在不同`goroutine`中的函数，上下文对于多个`goroutine`同时使用是安全的，`context`包定义了上下文类型，可以使用`background`、`TODO`创建一个上下文，在函数调用链之间传播`context`，也可以使用`WithDeadline`、`WithTimeout`、`WithCancel` 或 `WithValue` 创建的修改副本替换它，听起来有点绕，其实总结起就是一句话：**`context`的作用就是在不同的`goroutine`之间同步请求特定的数据、取消信号以及处理请求的截止日期**。