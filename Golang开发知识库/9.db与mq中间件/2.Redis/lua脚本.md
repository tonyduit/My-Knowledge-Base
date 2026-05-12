# 使用go-redis执行lua脚本

Redis 是开发中常用的高性能缓存数据库。除了常规的 GET/SET 操作，Redis 还支持通过 Lua 脚本实现复杂的原子操作。本文将带你循序渐进地学习如何在 Go 语言中，利用 [go-redis](https://github.com/redis/go-redis) 执行 Lua 脚本，并进一步讲解脚本缓存（script load）与 Go 的 `embed` 特性的结合使用。

在日常的 Go 项目开发中，Redis 无疑是我们最常用的缓存和数据存储中间件之一。它性能强大，数据结构丰富。然而，在一些复杂的业务场景下，我们可能需要执行多个 Redis 命令，并期望这些操作能像数据库事务一样保证原子性。此外，频繁地与 Redis 进行网络通信也会带来不小的开销。

如何解决这些问题呢？今天，我们就来聊聊 Redis 中的一个大杀器 —— Lua 脚本，并深入探讨如何在 go-redis 中从入门到优雅地使用它。

## 一、Lua 脚本与 Redis 

**Lua** 是一种小巧灵活的脚本语言，Redis 内置了 Lua 脚本引擎，允许你在服务端原子性地执行一系列命令。这样可以避免网络多次往返、保证操作的原子性。

### 为什么要在 Redis 中使用 Lua 脚本？ 

在深入代码之前，我们先要明白“为什么用”。天下没有免费的午餐，引入一项新技术通常是为了解决特定的痛点。在 Redis 中使用 Lua 脚本主要有以下几个核心优势：

- 原子性：这是最重要的原因。Redis 会将整个 Lua 脚本作为一个不可分割的命令来执行。在脚本执行期间，不会有其他客户端的命令插入进来。这为我们提供了一种简单高效的方式来实现复杂的原子操作，例如“读取、判断、写入”这类经典的 check-and-set 场景，从而避免了使用 `WATCH`/`MULTI`/`EXEC` 事务块的复杂性。
- 性能提升：将多个命令打包到一个脚本中，可以显著减少客户端与 Redis 服务器之间的网络往返次数（RTT）。在高并发场景下，这种优化带来的性能提升是非常可观的。原本需要 5 次网络请求的操作，现在 1 次就能完成。
- 复用性：编写好的 Lua 脚本可以被多个客户端复用，将业务逻辑下沉到 Redis 端，简化客户端代码。

Lua 脚本常见使用场景示例：

- 原子性扣减库存：避免并发条件下超卖问题。
- 多 Key 操作：如先检查 Key1 是否存在再删除 Key2。
- 限流控制：如固定窗口限流、滑动窗口限流。

例如：我们来看一个常见的限流场景：限制某个 IP 在 60 秒内只能访问 3 次。如果用传统命令，我们需要先 GET，判断，再 INCR，这并非原子操作。但用 Lua 脚本就能轻松搞定。

```lua
-- KEYS[1]: 限流的 key, 比如 ip:127.0.0.1
-- ARGV[1]: 过期时间（秒）
-- ARGV[2]: 限制次数

-- 获取当前 key 的访问次数
local current = redis.call('get', KEYS[1])
if current and tonumber(current) >= tonumber(ARGV[2]) then
    -- 如果超过限制，返回 0
    return 0
end

-- 访问次数加 1
local count = redis.call('incr', KEYS[1])

-- 如果是第一次访问，设置过期时间
if count == 1 then
    redis.call('expire', KEYS[1], ARGV[1])
end

-- 访问未超限，返回 1
return 1
```

上面的 Lua 脚本会为一个 key（比如 IP 地址）计数。如果是第一次访问，就设置 60 秒的过期时间。如果访问次数超过 limit，则返回 0；否则返回 1。

## 二、go-redis 执行 Lua 脚本 

[go-redis](https://github.com/redis/go-redis) 是 Go 语言中操作 Redis 的高性能客户端库。它支持通过 `Eval` 方法执行 Lua 脚本。

先安装 go-redis：

```bash
go get github.com/redis/go-redis/v9
```

### 在 Go 中执行 Lua 脚本 

```go
package main

import (
	"context"
	"fmt"
	"github.com/redis/go-redis/v9"
)

var (
	rdb *redis.Client
	ctx = context.Background()
)

func init() {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})
	if _, err := rdb.Ping(ctx).Result(); err != nil {
		panic(err)
	}
}

func main() {
	// 将 Lua 脚本定义为字符串
	luaScript := `
		local current = redis.call('get', KEYS[1])
		if current and tonumber(current) >= tonumber(ARGV[2]) then
			return 0
		end
		
		local count = redis.call('incr', KEYS[1])
		
		if count == 1 then
			redis.call('expire', KEYS[1], ARGV[1])
		end
		
		return 1
	`

	// 定义 key 和参数
	key := "ip:127.0.0.1"
	expire := "60" // 60s
	limit := "3"   // 3次

	// 执行脚本
  // func (c *Client) Eval(ctx context.Context, script string, keys []string, args ...interface{}) *Cmd
	result, err := rdb.Eval(ctx, luaScript, []string{key}, expire, limit).Result()
	if err != nil {
		panic(err)
	}

	fmt.Printf("执行结果: %v, 类型: %T\n", result, result)

	// 根据返回结果判断是否允许访问
	if result.(int64) == 1 {
		fmt.Println("访问成功！")
	} else {
		fmt.Println("访问被拒绝，已达到访问上限！")
	}
}
```

这种执行方式非常直观，易于理解。但它有一个明显的缺点：每次调用 `Eval` 时，都需要将完整的 Lua 脚本字符串从客户端发送到 Redis 服务器。 如果脚本很长，这会造成不必要的网络带宽浪费。

那么，有没有更优化的方法呢？当然有，这就是 `SCRIPT LOAD` 大显身手的时候了。

## 三、Redis 的 Script Load 及其应用 

### Script Load 是什么？ 

当 Lua 脚本频繁执行时，每次发送完整脚本字符串到 Redis 性能并不高。Redis 支持“脚本缓存”，即通过 `SCRIPT LOAD` 命令将脚本提前加载到服务端，返回一个 `SHA1` 校验码。之后可以用 `EVALSHA` 命令，仅发送 SHA1 值和参数（比完整的脚本短得多），因此可以大大减少网络开销。

### Script Load 的应用场景 

`SCRIPT LOAD` 的主要应用场景就是对于需要被频繁调用的、相对固定的 Lua 脚本。我们可以应用启动时就将脚本加载到 Redis 中，获取其 SHA1 值并缓存在应用内存里。后续的每次调用，都直接使用 `EVALSHA` 命令，从而实现性能的最大化。

### Go-redis 示例：使用 Script Load 优化 

`go-redis` 提供了 `redis.NewScript` 函数，它会返回一个 `*redis.Script` 对象。这个对象非常智能，它的 Run 方法会优先尝试使用 `EVALSHA` 执行脚本。如果 Redis 返回 `NOSCRIPT` 错误（表示脚本缓存中不存在该 SHA1 对应的脚本），它会自动回退（fallback），改用 `EVAL` 命令执行脚本原文，并将脚本重新加载到缓存中。后续的调用又会回到 `EVALSHA` 的轨道上。

这对开发者来说是完全透明的，我们只需要使用 `NewScript` 即可享受 `EVALSHA` 带来的性能优势。

让我们来改造一下上面的例子：

```go
package main

import (
	"context"
	"fmt"
	"github.com/redis/go-redis/v9"
)

var (
	rdb *redis.Client
	ctx = context.Background()

	// 使用 redis.NewScript 创建一个脚本对象
	limiterScript = redis.NewScript(`
		local current = redis.call('get', KEYS[1])
		if current and tonumber(current) >= tonumber(ARGV[2]) then
			return 0
		end
		local count = redis.call('incr', KEYS[1])
		if count == 1 then
			redis.call('expire', KEYS[1], ARGV[1])
		end
		return 1
	`)
)

func init() {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})
	if _, err := rdb.Ping(ctx).Result(); err != nil {
		panic(err)
	}
}

func main() {
	key := "ip:127.0.0.1"
	expire := 60 // 60s
	limit := 3   // 3次

	for i := 0; i < 5; i++ {
		// 使用脚本对象的 Run 方法执行
		// func (s *Script) Run(ctx context.Context, c Scripter, keys []string, args ...interface{}) *Cmd
		result, err := limiterScript.Run(ctx, rdb, []string{key}, expire, limit).Result()
		if err != nil && err != redis.Nil {
			panic(err)
		}

		if result.(int64) == 1 {
			fmt.Printf("第 %d 次访问: 成功！\n", i+1)
		} else {
			fmt.Printf("第 %d 次访问: 被拒绝！\n", i+1)
		}
	}
}
```

现在的代码结构是不是更清晰了？我们将脚本的定义和业务逻辑分离开来。`limiterScript` 可以在包级别定义，随处复用。`go-redis` 在底层为我们处理了 `SHA1` 的缓存和 `EVALSHA` 的调用，非常优雅。

## 四、Go 语言中的 embed 特性 

虽然 `redis.NewScript` 解决了性能问题，但将大段的 Lua 脚本以字符串形式硬编码在 Go 文件中，仍然有些不妥：

- 代码可读性差：Go 代码和 Lua 脚本混杂在一起。
- 维护困难：IDE 对字符串中的 Lua 脚本无法提供语法高亮和格式化支持。
- 职责不清：Go 文件应该主要关注业务逻辑，而不是存储大段的脚本。

Go 1.16 版本引入了一个非常实用的新特性：新增了 `embed` 标准库，可以将文件内容在编译时打包进二进制文件中，避免运行时依赖外部文件。

### embed 是什么？ 

`//go:embed` 是一个编译器指令，可以让我们将文件的内容读取到变量中。它可以将文件内容加载到 `string`、`[]byte` 或 `embed.FS` 类型的变量中。

### embed 的应用场景 

`embed` 特别适合用于嵌入模板文件（HTML）、静态资源（JS, CSS）、配置文件以及我们今天的主角——SQL 或 Lua 脚本文件。这使得我们的应用程序可以打包成一个独立的二进制文件，无需在部署时携带一堆零散的静态资源文件。

### 使用 go:embed 优化 Lua 脚本管理 

现在，我们使用 `embed` 来最终优化我们的代码。

1、在项目根目录下创建一个 `scripts` 文件夹，并在其中创建 `limiter.lua` 文件，内容如下：

```lua
-- file: scripts/limiter.lua

-- KEYS[1]: 限流的 key, 比如 ip:127.0.0.1
-- ARGV[1]: 过期时间（秒）
-- ARGV[2]: 限制次数

local current = redis.call('get', KEYS[1])
if current and tonumber(current) >= tonumber(ARGV[2]) then
    return 0
end

local count = redis.call('incr', KEYS[1])

if count == 1 then
    redis.call('expire', KEYS[1], ARGV[1])
end

return 1
```

2、修改 Go 代码以嵌入脚本， 使用 `//go:embed` 指令来加载 `limiter.lua` 文件的内容。

**注意：// 和 go:embed 之间没有空格**

```go
package main

import (
	"context"
	_ "embed" // 注意：需要导入 embed 包
	"fmt"
	"github.com/redis/go-redis/v9"
)

//go:embed scripts/limiter.lua
var limiterScriptSource string // lua 脚本内容将被加载到这个字符串变量中

var (
	rdb *redis.Client
	ctx = context.Background()

	// 使用从文件中加载的脚本源码创建 Script 对象
	limiterScript = redis.NewScript(limiterScriptSource)
)

func init() {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})
	if _, err := rdb.Ping(ctx).Result(); err != nil {
		panic(err)
	}
}

func main() {
	key := "ip:192.168.1.1" // 换个IP测试
	expire := 60
	limit := 3

	fmt.Println("使用 embed 方式加载的 Lua 脚本:")
	fmt.Println("--------------------------------")
	fmt.Println(limiterScriptSource)
	fmt.Println("--------------------------------")

	for i := 0; i < 5; i++ {
		result, err := limiterScript.Run(ctx, rdb, []string{key}, expire, limit).Result()
		if err != nil && err != redis.Nil {
			panic(err)
		}

		if result.(int64) == 1 {
			fmt.Printf("第 %d 次访问: 成功！\n", i+1)
		} else {
			fmt.Printf("第 %d 次访问: 被拒绝！\n", i+1)
		}
	}
}
```

现在的项目结构清晰明了：

```bash
.
├── go.mod
├── go.sum
├── main.go
└── scripts
    └── limiter.lua
```

通过 `embed`，我们成功地将 Lua 脚本与 Go 业务逻辑代码解耦。`.lua` 文件可以得到更好的 IDE 支持，也更便于单独维护和测试。同时，编译后的 Go 程序依然是一个独立的二进制文件，部署起来干净利落。这可以说是在 `go-redis` 中使用 Lua 脚本的“最终形态”了，兼顾了性能、可读性、可维护性和部署的便捷性。

## 五、总结 

今天，我们循序渐进地探讨了如何在 Go 语言中通过 `go-redis` 库高效地使用 Lua 脚本。

1. 我们从为什么需要 Lua 脚本出发，理解了它在**原子性**和**性能**上的巨大优势。
2. 接着，我们学习了最基础的 `Eval` 命令，用于直接执行脚本字符串，简单粗暴但有效。
3. 然后，我们引入了 `SCRIPT LOAD` 的概念，并利用 `go-redis` 提供的 `redis.NewScript` 对象，以一种更智能、更高性能的方式来执行脚本，让库为我们自动处理 `EVALSHA` 的逻辑。
4. 最后，我们使用 Go 1.16+ 的 `embed` 特性，将 Lua 脚本从 Go 代码中分离到独立的文件里，实现了**代码解耦**和**关注点分离**，使得整个项目结构更加清晰和专业。

在实际项目中，推荐将频繁变化、需要独立维护的 Lua 脚本写在单独文件，用 `go:embed` 加载，并用 `redis.NewScript` 做缓存优化。这样既能保证性能，又能让代码更优雅、易维护！

希望这篇教程对你有所帮助，如果你喜欢这类 Go 实战教程，欢迎关注和转发！