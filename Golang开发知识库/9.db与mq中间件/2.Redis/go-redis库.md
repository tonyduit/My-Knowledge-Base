# go-redis库

在项目开发中redis的使用也比较频繁，本文介绍了Go语言中`go-redis`库的基本使用。

# Redis

在项目开发中redis的使用也比较频繁，本文介绍了Go语言中`go-redis`库的基本使用。

## Redis介绍

Redis是一个开源的内存数据库，Redis提供了多种不同类型的数据结构，很多业务场景下的问题都可以很自然地映射到这些数据结构上。除此之外，通过复制、持久化和客户端分片等特性，我们可以很方便地将Redis扩展成一个能够包含数百GB数据、每秒处理上百万次请求的系统。

### Redis支持的数据结构

Redis支持诸如字符串（string）、哈希（hashe）、列表（list）、集合（set）、带范围查询的排序集合（sorted set）、bitmap、hyperloglog、带半径查询的地理空间索引（geospatial index）和流（stream）等数据结构。

### Redis应用场景

- 缓存系统，减轻主数据库（MySQL）的压力。
- 计数场景，比如微博、抖音中的关注数和粉丝数。
- 热门排行榜，需要排序的场景特别适合使用ZSET。
- 利用 LIST 可以实现队列的功能。
- 利用 HyperLogLog 统计UV、PV等数据。
- 使用 geospatial index 进行地理位置相关查询。

### 准备Redis环境

读者可以选择在本机安装 redis 或使用云数据库，这里直接使用Docker启动一个 redis 环境，方便学习使用。

使用下面的命令启动一个名为 redis507 的 5.0.7 版本的 redis server环境。

```bash
docker run --name redis507 -p 6379:6379 -d redis:5.0.7
```

**注意：** 此处的版本、容器名和端口号可以根据自己需要设置。

启动一个 redis-cli 连接上面的 redis server。

```bash
docker run -it --network host --rm redis:5.0.7 redis-cli
```

## go-redis库

### 安装

Go 社区中目前有很多成熟的 redis client 库，比如https://github.com/gomodule/redigo 和https://github.com/redis/go-redis，读者可以自行选择适合自己的库。本文使用 go-redis 这个库来操作 Redis 数据库。

使用以下命令下安装 go-redis 库。

安装`v8`版本：

```bash
go get github.com/redis/go-redis/v8
```

安装`v9`版本：

```bash
go get github.com/redis/go-redis/v9
```

### 连接

在项目中导入 `go-redis`库（请根据实际情况导入自己需要的版本）。

```go
import "github.com/redis/go-redis/v9"
```

#### 普通连接模式

go-redis 库中使用 redis.NewClient 函数连接 Redis 服务器。

```go
rdb := redis.NewClient(&redis.Options{
	Addr:     "localhost:6379",
	Password: "", // 密码
	DB:       0,  // 数据库
	PoolSize: 20, // 连接池大小
})
```

除此之外，还可以使用 redis.ParseURL 函数从表示数据源的字符串中解析得到 Redis 服务器的配置信息。

```go
opt, err := redis.ParseURL("redis://<user>:<pass>@localhost:6379/<db>")
if err != nil {
	panic(err)
}

rdb := redis.NewClient(opt)
```

#### TLS连接模式

如果使用的是 TLS 连接方式，则需要使用 tls.Config 配置。

```go
rdb := redis.NewClient(&redis.Options{
	TLSConfig: &tls.Config{
		MinVersion: tls.VersionTLS12,
		// Certificates: []tls.Certificate{cert},
    // ServerName: "your.domain.com",
	},
})
```

#### Redis Sentinel模式

使用下面的命令连接到由 Redis Sentinel 管理的 Redis 服务器。

```go
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
    MasterName:    "master-name",
    SentinelAddrs: []string{":9126", ":9127", ":9128"},
})
```

#### Redis Cluster模式

使用下面的命令连接到 Redis Cluster，go-redis 支持按延迟或随机路由命令。

```go
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},

    // 若要根据延迟或随机路由命令，请启用以下命令之一
    // RouteByLatency: true,
    // RouteRandomly: true,
})
```

## 基本使用

### 执行命令

下面的示例代码演示了 go-redis 库的基本使用。

```go
// doCommand go-redis基本使用示例
func doCommand() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	// 执行命令获取结果
	val, err := rdb.Get(ctx, "key").Result()
	fmt.Println(val, err)

	// 先获取到命令对象
	cmder := rdb.Get(ctx, "key")
	fmt.Println(cmder.Val()) // 获取值
	fmt.Println(cmder.Err()) // 获取错误

	// 直接执行命令获取错误
	err = rdb.Set(ctx, "key", 10, time.Hour).Err()

	// 直接执行命令获取值（不会返回错误，val会返回对应类型的零值）
	value := rdb.Get(ctx, "key").Val()
	fmt.Println(value)
}
```

### 执行任意命令

go-redis 还提供了一个执行任意命令或自定义命令的 Do 方法，特别是一些 go-redis 库暂时不支持的命令都可以使用该方法执行。具体使用方法如下。

```go
// doDemo rdb.Do 方法使用示例
func doDemo() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	// 直接执行命令获取错误
	err := rdb.Do(ctx, "set", "key", 10, "EX", 3600).Err()
	fmt.Println(err)

	// 执行命令获取结果
	val, err := rdb.Do(ctx, "get", "key").Result()
	fmt.Println(val, err)
}
```

### redis.Nil

go-redis 库提供了一个 redis.Nil 错误来表示 Key 不存在的错误。因此在使用 go-redis 时需要注意对返回错误的判断。在某些场景下我们应该区别处理 redis.Nil 和其他不为 nil 的错误。

```go
// getValueFromRedis redis.Nil判断
func getValueFromRedis(key, defaultValue string) (string, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	val, err := rdb.Get(ctx, key).Result()
	if err != nil {
		// 如果返回的错误是key不存在
		if errors.Is(err, redis.Nil) {
			return defaultValue, nil
		}
		// 出其他错了
		return "", err
	}
	return val, nil
}
```

### 关于Result()和Val()

当你调用像 `rdb.Get(ctx, "key")` 这样的方法时，它**并不直接返回值和 error**，而是返回一个 **“命令结果对象”**，比如：

- `*redis.StringCmd`
- `*redis.IntCmd`
- `*redis.BoolCmd`
- `*redis.FloatCmd`
- `*redis.StatusCmd`
- 等等

这些类型都实现了通用接口，并提供了 `.Result()` 和 `.Val()` 方法。

**✅ 1. `.Result()` —— 同时获取值和错误**

```go
val, err := rdb.Get(ctx, "key").Result()
```

- 返回两个值：`(T, error)`
  - `T` 是你期望的类型（如 `string`, `int64`, `float64`, `bool` 等）的值
  - `error`可能是：
    - `nil`（成功）
    - `redis.Nil`（key 不存在，仅对某些命令如 `GET`、`HGET` 等）
    - 其他网络或 Redis 错误

> ✅ **推荐在需要处理错误的场景使用 `.Result()`**

**示例：**

```go
val, err := rdb.Get(ctx, "username").Result()
if err == redis.Nil {
    fmt.Println("key 不存在")
} else if err != nil {
    log.Fatal("Redis 错误:", err)
} else {
    fmt.Println("值是:", val) // val 是 string 类型
}
```

**✅ 2. `.Val()` —— 只获取值，忽略错误（危险！）**

```go
val := rdb.Get(ctx, "key").Val()
```

- **只返回值**（如 `string`），**不返回 error**
- 如果命令出错（包括 key 不存在），`.Val()`会返回该类型的零值：
  - `string` → `""`
  - `int64` → `0`
  - `bool` → `false`
  - `[]string` → `nil`

> ⚠️ **仅在你确定操作一定成功时使用（如测试、初始化数据）**
> 否则可能掩盖 bug！

**示例（不安全）：**

```go
// 如果 "counter" 不存在，Val() 返回 0，你可能误以为计数器就是 0
count := rdb.Get(ctx, "counter").Val()
fmt.Println("当前计数:", count) // 可能是 ""（如果 key 不存在！）
```

> 💥 注意：`GET` 返回的是字符串，如果 key 不存在，`.Val()` 返回空字符串 `""`，**你无法区分“值就是空串”和“key 根本不存在”**！

**🔍 对比表格**

| 方法        | 返回值       | 是否包含错误信息       | 适用场景                           |
| :---------- | :----------- | :--------------------- | :--------------------------------- |
| `.Result()` | `(T, error)` | ✅ 是                   | **生产代码、需要错误处理**         |
| `.Val()`    | `T`          | ❌ 否（出错时返回零值） | 快速原型、测试、确定不会出错的场景 |

**📌 其他常见命令示例**

**ZIncrBy（返回 float64）**

```go
// 安全方式
newScore, err := rdb.ZIncrBy(ctx, "leaderboard", 10, "Alice").Result()
if err != nil { /* 处理错误 */ }

// 危险方式（不推荐）
newScore := rdb.ZIncrBy(ctx, "leaderboard", 10, "Alice").Val() // 出错时返回 0.0
```

**Exists（返回 int64）**

```go
count, err := rdb.Exists(ctx, "key1", "key2").Result() // count 是存在的 key 数量
exists := rdb.Exists(ctx, "key").Val() > 0            // 快速判断是否存在（但忽略错误！）
```

**Set（返回 status，通常是 "OK"）**

```go
status, err := rdb.Set(ctx, "k", "v", 0).Result() // status == "OK"
ok := rdb.Set(ctx, "k", "v", 0).Val()             // ok == "OK"（但出错时返回 ""）
```

### redis数据类型

![img](go-redis%E5%BA%93.assets/139239-20191126141006657-1969131669.png)

```
redis可以理解成一个全局的大字典，key就是数据的唯一标识符。根据key对应的值不同，可以划分成5个基本数据类型。

redis = {
    "name":"yuan",
    "scors":["100","89","78"],
    "info":{
        "name":"rain"
        "age":22
    },
    "s":{item1,itme2,..}
}

1. string类型:
	字符串类型，是 Redis 中最为基础的数据存储类型，它在 Redis 中是二进制安全的，也就是byte类型。
	单个数据的最大容量是512M。
		key: 值
	
2. hash类型:
	哈希类型，用于存储对象/字典，对象/字典的结构为键值对。key、域、值的类型都为string。域在同一个hash中是唯一的。
		key:{
            域（属性）: 值，
            域:值，            
            域:值，
            域:值，
            ...
		}
3. list类型:
	列表类型，它的子成员类型为string。
		key: [值1，值2, 值3.....]
4. set类型:
	无序集合，它的子成员类型为string类型，元素唯一不重复，没有修改操作。
		key: {值1, 值4, 值3, ...., 值5}

5. zset类型(sortedSet):
	有序集合，它的子成员值的类型为string类型，元素唯一不重复，没有修改操作。权重值(score,分数)从小到大排列。
		key: {
			值1 权重值1(数字);
			值2 权重值2;
			值3 权重值3;
			值4 权重值4;
		}
```

#### 4.1. string（字符串）

> - SET/SETEX/MSET/MSETNX
> - GET/MGET
> - GETSET
> - INCR/DECR
> - DEL

**1. 设置键值**

set 设置的数据没有额外操作时，是不会过期的。

```bash
set key value
```

设置键为`name`值为`yuan`的数据

```bash
set name yuan
set name rain # 一个变量可以设置多次
```

![image-20220415103809481](go-redis%E5%BA%93.assets/image-20220415103809481-16499902909211.png)

注意：redis中的所有数据操作，如果设置的键不存在则为添加，如果设置的键已经存在则修改。

设置一个键，当键不存在时才能设置成功，用于一个变量只能被设置一次的情况。

```bash
setnx  key  value
```

一般用于给数据加锁(分布式锁)

```bash
127.0.0.1:6379> setnx goods_1 101
(integer) 1
127.0.0.1:6379> setnx goods_1 102
(integer) 0  # 表示设置不成功

127.0.0.1:6379> del goods_1
(integer) 1
127.0.0.1:6379> setnx goods_1 102
(integer) 1
```

**2. 设置键值的过期时间**

redis中可以对一切的数据进行设置有效期。以秒为单位

```bash
setex key seconds value
```

设置键为`goods_1`值为`name`过期时间为10秒的数据

```bash
setex goods_1 10 name
```

**3. 关于设置保存数据的有效期**

setex 添加保存数据到redis，同时设置有效期，格式：

```bash
setex key time value
```

![image-20220415104126645](go-redis%E5%BA%93.assets/image-20220415104126645-16499904877172.png)

**4. 设置多个键值**

```bash
mset key1 value1 key2 value2 ...
```

例3：设置键为`a1`值为`goland`、键为`a2`值为`java`、键为`a3`值为`c`

```bash
mset a1 goland a2 java a3 c
```

**5. 字符串拼接值**

常见于大文件上传

```bash
append key value
```

向键为`a1`中拼接值`haha`

```bash
set title "我的"
append title "redis"
append title "学习之路"
```

**6. 根据键获取值**

根据键获取值，如果不存在此键则返回`nil`

```bash
get key
```

获取键`name`的值

```bash
get name
```

根据多个键获取多个值

```bash
mget key1 key2 ...
```

获取键`a1、a2、a3`的值

```bash
mget a1 a2 a3
```

getset：设置key的新值，返回旧值

```bash
redis> GETSET db mongodb    # 没有旧值，返回 nil
(nil)
redis> GET db
"mongodb"

redis> GETSET db redis      # 返回旧值 mongodb
"mongodb"

redis> GET db
"redis"
```

**7. 自增自减**

web开发中的电商抢购、秒杀。游戏里面的投票、攻击计数。系统中计算当前在线人数、

```bash
set id 1
incr id   # 相当于id+1
get id    # 2
incr id   # 相当于id+1
get id    # 3

set goods_id_1 10
decr goods_id_1  # 相当于 id-1
get goods_id_1    # "9"
decr goods_id_1   # 相当于id-1
get goods_id_1    # 8

 set age 22
 incrby age 2 # 自增自减大于1的值时候用incrby
```

**8. 获取字符串的长度**

```bash
set name xiaoming
strlen name  # 8 
```

**9. 比特流操作  **

————————

mykey  00000011

1字节=8比特  1kb = 1024字节  1mb = 1024kb 1gb = 1024mb

1个int8就是一个字节，一个中文：3个字节

```bash
SETBIT     # SETBIT key offset value 按从左到右的偏移量设置一个bit数据的值 
GETBIT     # 获取一个bit数据的值
BITCOUNT   # 统计字符串被设置为1的bit数.
BITPOS     # 返回字符串里面第一个被设置为1或者0的bit位。
```

案例1：

```bash
SETBIT mykey 7 1
# 00000001

getbit mykey 7
# 00000001

SETBIT mykey 4 1
# 00001001

# 使用该命令验证这个bit的值
127.0.0.1:6379> bitfield mykey get u8 0
1) (integer) 9

SETBIT mykey 15 1
# 0000100100000001
```

我们知道 'a' 的ASCII码是 97。转换为二进制是：01100001。offset的学名叫做“偏移” 。二进制中的每一位就是offset值啦，比如在这里 offset 0 等于 ‘0’ ，offset 1等于 '1' ，offset 2 等于 '1'，offset 6 等于 '0' ，offset 7 等于 '1'，没错，offset是从左向右计数的，也就是从**高位往低位**。

我们通过SETBIT 命令将 andy中的 'a' 变成 'b' 应该怎么变呢？

也就是将 01100001 变成 01100010 （b的ASCII码是98），这个很简单啦，也就是将'a'中的offset 6从0变成1，将offset 7 从1变成0 。

```bash
127.0.0.1:6379> set name andy
OK

127.0.0.1:6379> setbit name 6 1
(integer) 0

127.0.0.1:6379> setbit name 7 0
(integer) 1

127.0.0.1:6379> get name
"bndy"
```

这里是修改了第一个字符，所以offset的值在0~7之间，如果要修改第二个字符就要修改8~15位，以此类推

案例2：签到系统

````bash
setbit user_1 6 1
setbit user_1 5 1
setbit user_1 4 0
setbit user_1 3 1
setbit user_1 2 0
setbit user_1 1 1
setbit user_1 0 1
BITCOUNT user_1 # 统计一周的打卡情况
````

#### 4.2. key操作

redis中所有的数据都是通过key（键）来进行操作，这里我们学习一下关于任何数据类型都通用的命令。

**（1）查找键**

参数支持简单的正则表达式

```bash
keys pattern
```

查看所有键

```bash
keys *
```

例子：

```bash
# 查看名称中包含`a`的键
keys *a*
# 查看以a开头的键
keys a*
# 查看以a结尾的键
keys *a
```

**（2）判断键是否存在**

如果存在返回`1`，不存在返回`0`

```bash
exists key1
```

判断键`title`是否存在

```bash
exists title
```

**（3）查看键的的值的数据类型**

```bash
type key

# string    字符串
# hash      哈希类型
# list      列表类型
# set       无序集合
# zset      有序集合
```

查看键的值类型

```bash
type a1
# string

sadd member_list xiaoming xiaohong xiaobai
# (integer) 3
type member_list
# set

hset user_1 name xiaobai age 17 sex 1
# (integer) 3
type user_1
# hash

lpush brothers zhangfei guangyu liubei xiaohei
# (integer) 4
type brothers
# list

zadd achievements 61 xiaoming 62 xiaohong 83 xiaobai  78 xiaohei 87 xiaohui 99 xiaolong
# (integer) 6
type achievements
# zset
```

**（4）删除键以及键对应的值**

```bash
del key1 key2 ...
```

**（5）查看键的有效期**

```bash
ttl key

# 结果结果是秒作为单位的整数
# -1 表示永不过期
# -2 表示当前数据已经过期，查看一个不存在的数据的有效期就是-2
```

**（6）设置key的有效期**

给已有的数据重新设置有效期，redis中所有的数据都可以通过expire来设置它的有效期。有效期到了，数据就被删除。

```bash
expire key seconds
```

**（7）清空所有key**

慎用，一旦执行，则redis所有数据库0~15的全部key都会被清除

```bash
flushall
```

**（8）key重命名**

```bash
rename  oldkey newkey
```

把name重命名为username

```bash
set name yuan
rename name username
get username
```

select切换数据库

```bash
redis的配置文件中，默认有0~15之间的16个数据库，默认操作的就是0号数据库
select <数据库ID>
```

操作效果：

```bash
# 默认处于0号库
127.0.0.1:6379> select 1
OK
# 这是在1号库
127.0.0.1:6379[1]> set name xiaoming
OK
127.0.0.1:6379[1]> select 2
OK
# 这是在2号库
127.0.0.1:6379[2]> set name xiaohei
OK
```

#### 4.3. list（数组）

队列，列表的子成员类型为string

> lpush key value
>
> rpush key value
>
> linsert key after|before  指定元素 value
>
> lindex key index
>
> lrange key start stop
>
> lset key index value
>
> lrem key count value

**（1）添加子成员**

```bash
# 在左侧(前)添加一条或多条数据
lpush key value1 value2 ...
# 在右侧(后)添加一条或多条数据
rpush key value1 value2 ...

# 在指定元素的左边(前)/右边（后）插入一个或多个数据
linsert key before 指定元素 value1 value2 ....
linsert key after 指定元素 value1 value2 ....
```

从键为`brother`的列表左侧添加一个或多个数据`liubei、guanyu、zhangfei`

```bash
lpush brother liubei
# [liubei]
lpush brother guanyu zhangfei xiaoming
# [xiaoming,zhangfei,guanyu,liubei]
```

从键为brother的列表右侧添加一个或多个数据，`xiaohong,xiaobai,xiaohui`

```bash
rpush brother xiaohong
# [xiaoming,zhangfei,guanyu,liubei,xiaohong]
rpush brother xiaobai xiaohui
# [xiaoming,zhangfei,guanyu,liubei,xiaohong,xiaobai,xiaohui]
```

从key=brother的xiaohong的列表位置左侧添加一个数据，`xiaoA,xiaoB`

```bash
linsert brother before xiaohong xiaoA
# [xiaoming,zhangfei,guanyu,liubei,xiaoA,xiaohong,xiaobai,xiaohui]
linsert brother before xiaohong xiaoB
# [xiaoming,zhangfei,guanyu,liubei,xiaoA,xiaoB,xiaohong,xiaobai,xiaohui]
```

从key=brother，key=xiaohong的列表位置右侧添加一个数据，`xiaoC,xiaoD`

```bash
linsert brother after xiaohong xiaoC
# [xiaoming,zhangfei,guanyu,liubei,xiaoA,xiaohong,xiaoC,xiaobai,xiaohui]
linsert brother after xiaohong xiaoD
# [xiaoming,zhangfei,guanyu,liubei,xiaoA,xiaohong,xiaoD,xiaoC,xiaobai,xiaohui]
```

注意：当列表如果存在多个成员值一致的情况下，默认只识别第一个。

```bash
127.0.0.1:6379> linsert brother before xiaoA xiaohong
# [xiaoming,zhangfei,guanyu,liubei,xiaohong,xiaoA,xiaohong,xiaoD,xiaoC,xiaobai,xiaohui]
127.0.0.1:6379> linsert brother before xiaohong xiaoE
# [xiaoming,zhangfei,guanyu,liubei,xiaoE,xiaohong,xiaoA,xiaohong,xiaoD,xiaoC,xiaobai,xiaohui]
127.0.0.1:6379> linsert brother after xiaohong xiaoF
# [xiaoming,zhangfei,guanyu,liubei,xiaoE,xiaohong,xiaoF,xiaoA,xiaohong,xiaoD,xiaoC,xiaobai,xiaohui]
```

**（2）基于索引获取列表成员**

根据指定的索引(下标)获取成员的值，负数下标从右边-1开始，逐个递减

```bash
lindex key index
```

获取brother下标为2以及-2的成员

```bash
del brother
lpush brother guanyu zhangfei xiaoming
lindex brother 2
# "guanyu"
lindex brother -2
# "zhangfei"
```

**（3）获取列表的切片**

闭区间[包括stop]

```bash
lrange key start stop
```

操作：

```bash
del brother
rpush brother liubei guanyu zhangfei xiaoming xaiohong
# 获取btother的全部成员
lrange brother 0 -1
# 获取brother的前2个成员
lrange brother 0 1
# 获取brother的后2个成员
lrange brother -2 -1
```

**（4）获取列表的长度**

```bash
llen key
```

获取brother列表的成员个数

```bash
llen brother
```

**（5）按索引设置值**

```bash
lset key index value
# 注意：
# redis的列表也有索引，从左往右，从0开始，逐一递增，第1个元素下标为0
# 索引可以是负数，表示尾部开始计数，如`-1`表示最后1个元素
```

修改键为`brother`的列表中下标为`4`的元素值为`xiaohongmao`

```bash
lset brother 4 xiaohonghong
```

**（6）删除指定成员**

移除并获取列表的第一个成员或最后一个成员

```bash
lpop key  # 第一个成员出列
rpop key  # 最后一个成员出列
```

获取并移除brother中的第一个成员

```bash
lpop brother
# 开发中往往使用rpush和lpop实现队列的数据结构->实现入列和出列
```

```bash
lrem key count value

# 注意：
# count表示删除的数量，value表示要删除的成员。该命令默认表示将列表从左侧前count个value的元素移除
# count==0，表示删除列表所有值为value的成员
# count > 0，表示删除列表左侧开始的前count个value成员
# count < 0，表示删除列表右侧开始的前count个value成员

del brother
rpush brother A B A C A
lrem brother 0 A
["B","C"]

del brother
rpush brother A B A C A
lrem brother -2 A
["A","B","C"]

del brother
rpush brother A B A C A
lrem brother 2 A
["B","C","A"]
```

#### 4.4. hash（哈希）

> hset key field value
>
> hget key field
>
> hgetall info
>
> hmget key field1 field2 ...
>
> hincrby key field number

专门用于结构化的数据信息。对应的就是map/结构体

结构：

```text
键key:{
   	域field: 值value,
   	域field: 值value,
   	域field: 值value,
}
```

**（1）设置指定键的属性/域**

设置指定键的单个属性

```bash
hset key field value
```

设置键 `user_1`的属性`name`为`xiaoming`

```bash
127.0.0.1:6379> hset user_1 name xiaoming   # user_1没有会自动创建
(integer) 1
127.0.0.1:6379> hset user_1 name xiaohei    # user_1中重复的属性会被修改
(integer) 0
127.0.0.1:6379> hset user_1 age 16          # user_1中不存在的属性会被新增
(integer) 1
127.0.0.1:6379> hset user:1 name xiaohui    # user:1会在redis界面操作中以:作为目录分隔符
(integer) 1
127.0.0.1:6379> hset user:1 age 15
(integer) 1
127.0.0.1:6379> hset user:2 name xiaohong age 16  # 一次性添加或修改多个属性
```

**（2）获取指定键的域/属性的值**

获取指定键所有的域/属性

```bash
hkeys key
```

获取键user的所有域/属性

```bash
127.0.0.1:6379> hkeys user:2
1) "name"
2) "age"
127.0.0.1:6379> hkeys user:3
1) "name"
2) "age"
3) "sex"
```

获取指定键的单个域/属性的值

```
hget key field
```

获取键`user:3`属性`name`的值

```bash
127.0.0.1:6379> hget user:3 name
"xiaohong"
```

获取指定键的多个域/属性的值

```bash
hmget key field1 field2 ...
```

获取键`user:2`属性`name`、`age`的值

```bash
127.0.0.1:6379> hmget user:2 name age
1) "xiaohong"
2) "16"
```

获取指定键的所有值

```bash
hvals key
```

获取指定键的所有域值对

```bash
127.0.0.1:6379> hvals user:3
1) "xiaohong"
2) "17"
3) "1"
```

**（3）获取hash的所有域值对**

```bash
127.0.0.1:6379> hset user:1 name xiaoming age 16 sex 1
(integer) 3
127.0.0.1:6379> hgetall user:1
1) "name"
2) "xiaoming"
3) "age"
4) "16"
5) "sex"
6) "1"
```

**（4）删除指定键的域/属性**

```bash
hdel key field1 field2 ...
```

删除键`user:3`的属性`sex/age/name`，当键中的hash数据没有任何属性，则当前键会被redis删除

```bash
hdel user:3 sex age name
```

**（5）判断指定属性/域是否存在于当前键对应的hash中**

```bash
hexists   key  field
```

判断user:2中是否存在age属性

```bash
127.0.0.1:6379> hexists user:3 age
(integer) 0
127.0.0.1:6379> hexists user:2 age
(integer) 1
127.0.0.1:6379> 
```

**（6）属性值自增自减**

```bash
hincrby key field number
```

给user:2的age属性在原值基础上+/-10，然后在age现有值的基础上-2

```bash
# 按指定数值自增
127.0.0.1:6379> hincrby user:2 age 10
(integer) 77
127.0.0.1:6379> hincrby user:2 age 10
(integer) 87

# 按指定数值自减
127.0.0.1:6379> hincrby user:2 age -10
(integer) 77
127.0.0.1:6379> hincrby user:2 age -10
```

#### 4.5. set（集合）

无序集合，重点就是去重和无序。

**（1）添加元素**

```bash
sadd key member1 member2 ...
```

向键`authors`的集合中添加元素`zhangsan`、`lisi`、`wangwu`

```bash
sadd authors zhangsan lisi wangwu
```

**（2）获取集合的所有的成员**

```bash
smembers key
```

获取键`authors`的集合中所有元素

```bash
smembers authors
```

**（3）获取集合的长度**

```bash
scard keys 
```

获取s2集合的长度

```bash
sadd s2 a b c d e

127.0.0.1:6379> scard s2
(integer) 5
```

**（4）随机抽取一个或多个元素**

抽取出来的成员被删除掉 

```bash
spop key [count=1]

# 注意：
# count为可选参数，不填则默认一个。被提取成员会从集合中被删除掉
```

随机获取s2集合的成员

```bash
sadd s2 a c d e

127.0.0.1:6379> spop s2 
"d"
127.0.0.1:6379> spop s2 
"c"
```

**（5）删除指定元素**

```bash
srem key value
```

删除键`authors`的集合中元素`wangwu`

```bash
srem authors wangwu
```

**（6）交集、差集和并集**

推荐、（协同过滤，基于用户、基于物品）

```bash
sinter  key1 key2 key3 ....    # 交集、比较多个集合中共同存在的成员
sdiff   key1 key2 key3 ....    # 差集、比较多个集合中不同的成员
sunion  key1 key2 key3 ....    # 并集、合并所有集合的成员，并去重
```

```bash
del user:1 user:2 user:3 user:4
sadd user:1 1 2 3 4     # user:1 = {1,2,3,4}
sadd user:2 1 3 4 5     # user:2 = {1,3,4,5}
sadd user:3 1 3 5 6     # user:3 = {1,3,5,6}
sadd user:4 2 3 4       # user:4 = {2,3,4}

# 交集
127.0.0.1:6379> sinter user:1 user:2
1) "1"
2) "3"
3) "4"
127.0.0.1:6379> sinter user:1 user:3
1) "1"
2) "3"
127.0.0.1:6379> sinter user:1 user:4
1) "2"
2) "3"
3) "4"

127.0.0.1:6379> sinter user:2 user:4
1) "3"
2) "4"

# 并集
127.0.0.1:6379> sunion user:1 user:2 user:4
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"

# 差集
127.0.0.1:6379> sdiff user:2 user:3
1) "4"  # 此时可以给user:3推荐4

127.0.0.1:6379> sdiff user:3 user:2
1) "6"  # 此时可以给user:2推荐6

127.0.0.1:6379> sdiff user:1 user:3
1) "2"
2) "4"
```

#### 4.6. zset（有序集合）

有序集合（score/value），去重并且根据score权重值来进行排序的。score从小到大排列。

**（1）添加成员**

```bash
zadd key score1 member1 score2 member2 score3 member3 ....
```

设置榜单achievements，设置成绩和用户名作为achievements的成员

```bash
127.0.0.1:6379> zadd achievements 61 xiaoming 62 xiaohong 83 xiaobai  78 xiaohei 87 xiaohui 99 xiaolan
(integer) 6
127.0.0.1:6379> zadd achievements 85 xiaohuang 
(integer) 1
127.0.0.1:6379> zadd achievements 54 xiaoqing
```

**（2）获取score在指定区间的所有成员**

```python
zrangebyscore key min max     # 按score顺序获取分数处于[min, max]区间的成员
zrevrangebyscore key min max  # 按score逆序获取分数处于[min, max]区间的成员
zrange key start stop         # 按scoer顺序获取索引处于[start, stop]区间的成员
zrevrange key start stop      # 按scoer逆序获取索引处于[start, stop]区间的成员
```

```python
zrange achievements 0 -1  # 从低到高全部成员
```

**（3）获取集合长度**

```bash
zcard key
```

获取users的长度

```bash
zcard achievements
```

**（4）获取指定成员的权重值**

```bash
zscore key member
```

获取users中xiaoming的成绩

```bash
127.0.0.1:6379> zscore achievements xiaobai
"93"
127.0.0.1:6379> zscore achievements xiaohong
"62"
127.0.0.1:6379> zscore achievements xiaoming
"61"
```

**（5）获取指定成员在集合中的排名**

排名从0开始计算

```bash
zrank key member      # score从小到大的排名
zrevrank key member   # score从大到小的排名
```

获取achievements中xiaohei的分数排名，从大到小

```bash
127.0.0.1:6379> zrevrank achievements xiaohei
(integer) 4
```

**（6）获取score在指定区间的所有成员数量**

```bash
zcount key min max
```

获取achievements从0~60分之间的人数[闭区间]

```bash
127.0.0.1:6379> zcount achievements 0 60
(integer) 2
127.0.0.1:6379> zcount achievements 54 60
(integer) 2
```

**（7）给指定成员增加增加权重值**

```bash
zincrby key score member
```

给achievements中xiaobai增加10分

```bash
127.0.0.1:6379> ZINCRBY achievements 10 xiaobai
"93
```

**（8）删除成员**

```bash
zrem key member1 member2 member3 ....
```

从achievements中删除xiaoming的数据

```bash
zrem achievements xiaoming
```

**（9）删除指定数量的成员**

```bash
# 删除指定数量的成员，从最低score开始删除
zpopmin key [count]
# 删除指定数量的成员，从最高score开始删除
zpopmax key [count]
```

例子：

```bash
# 从achievements中提取并删除成绩最低的2个数据
127.0.0.1:6379> zpopmin achievements 2
1) "xiaoqing"
2) "54"
3) "xiaolv"
4) "60"

# 从achievements中提取并删除成绩最高的2个数据
127.0.0.1:6379> zpopmax achievements 2
1) "xiaolan"
2) "99"
3) "xiaobai"
4) "93"
```

## 其他示例

### 1.初始化客户端

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/redis/go-redis/v9"
)

var ctx = context.Background()

func initClient() *redis.Client {
    // 创建 Redis 客户端连接
    rdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379", // Redis 服务器地址
        Password: "",               // 密码（若无则为空）
        DB:       0,                // 使用数据库 0
    })

    // 测试连接是否成功
    pong, err := rdb.Ping(ctx).Result()
    if err != nil {
        log.Fatal("连接 Redis 失败:", err)
    }
    fmt.Println("连接成功:", pong)

    return rdb
}
```

### 2.String 类型（字符串）

```go
func stringExample(rdb *redis.Client) {
    key := "user:name"

    // 设置字符串值
    err := rdb.Set(ctx, key, "Alice", 0).Err()
    if err != nil {
        log.Fatal("Set 失败:", err)
    }

    // 获取字符串值
    val, err := rdb.Get(ctx, key).Result()
    if err == redis.Nil {
        fmt.Println("key 不存在")
    } else if err != nil {
        log.Fatal("Get 失败:", err)
    } else {
        fmt.Println("user:name =", val) // 输出: Alice
    }

    // 自增操作（适用于整数字符串）
    rdb.Set(ctx, "counter", 10, 0)
    newCount, _ := rdb.Incr(ctx, "counter").Result()
    fmt.Println("自增后 counter =", newCount) // 11
}
```

### 3.Hash 类型（哈希表）

```go
func hashExample(rdb *redis.Client) {
    key := "user:1000"

    // 设置哈希字段
    err := rdb.HSet(ctx, key, "name", "Bob", "age", 30).Err()
    if err != nil {
        log.Fatal("HSet 失败:", err)
    }

    // 获取单个字段
    name, _ := rdb.HGet(ctx, key, "name").Result()
    fmt.Println("name =", name) // Bob

    // 获取所有字段和值
    all, _ := rdb.HGetAll(ctx, key).Result()
    fmt.Println("全部字段:", all) // map[name:Bob age:30]

    // 检查字段是否存在
    exists, _ := rdb.HExists(ctx, key, "email").Result()
    fmt.Println("email 字段存在?", exists) // false

    // 删除字段
    deleted, _ := rdb.HDel(ctx, key, "age").Result()
    fmt.Println("删除的字段数量:", deleted) // 1
}
```

### 4.List 类型（列表）

```go
func listExample(rdb *redis.Client) {
    key := "tasks"

    // 从左侧推入元素（LPush）
    rdb.LPush(ctx, key, "task3", "task2", "task1")

    // 从右侧弹出一个元素（RPop）
    task, _ := rdb.RPop(ctx, key).Result()
    fmt.Println("处理任务:", task) // task1

    // 获取列表长度
    length, _ := rdb.LLen(ctx, key).Result()
    fmt.Println("剩余任务数:", length) // 2

    // 获取列表中指定范围的元素（0 到 -1 表示全部）
    items, _ := rdb.LRange(ctx, key, 0, -1).Result()
    fmt.Println("当前任务列表:", items) // [task3 task2]
}
```

### 5.Set 类型（集合）

```go
func setExample(rdb *redis.Client) {
    key := "tags:news"

    // 添加多个元素到集合
    rdb.SAdd(ctx, key, "sports", "politics", "tech")

    // 检查元素是否在集合中
    isMember, _ := rdb.SIsMember(ctx, key, "tech").Result()
    fmt.Println("'tech' 在集合中?", isMember) // true

    // 获取集合所有成员
    members, _ := rdb.SMembers(ctx, key).Result()
    fmt.Println("所有标签:", members) // 顺序不定

    // 随机弹出一个元素
    random, _ := rdb.SPop(ctx, key).Result()
    fmt.Println("随机弹出的标签:", random)

    // 计算集合大小
    count, _ := rdb.SCard(ctx, key).Result()
    fmt.Println("剩余标签数量:", count)
}
```

### 6.Sorted Set（ZSet，有序集合）

```go
func zsetExample(rdb *redis.Client) {
    key := "leaderboard"

    // 添加带分数的成员（分数决定排序）
    rdb.ZAdd(ctx, key,
        redis.Z{Score: 100, Member: "Alice"},
        redis.Z{Score: 150, Member: "Bob"},
        redis.Z{Score: 120, Member: "Charlie"},
    )

    // 获取排名前2的成员（按分数从高到低）
    top, _ := rdb.ZRevRangeWithScores(ctx, key, 0, 1).Result()
    fmt.Println("Top 2:")
    for _, z := range top {
        fmt.Printf("  %s: %.0f\n", z.Member, z.Score)
    }

    // 获取某个成员的分数
    score, _ := rdb.ZScore(ctx, key, "Alice").Result()
    fmt.Println("Alice 的分数:", score)

    // 获取成员的排名（从低分到高分）
    rank, _ := rdb.ZRank(ctx, key, "Alice").Result()
    fmt.Println("Alice 的排名（升序）:", rank) // 0（最低分排第0）
}
```

### 7.Key 操作（通用）

```go
func keyExample(rdb *redis.Client) {
    // 设置带过期时间的 key（10 秒后自动删除）
    rdb.Set(ctx, "temp", "data", 10*time.Second)

    // 检查 key 是否存在
    exists, _ := rdb.Exists(ctx, "temp").Result()
    fmt.Println("temp key 存在?", exists)

    // 删除 key
    deleted, _ := rdb.Del(ctx, "temp").Result()
    fmt.Println("删除的 key 数量:", deleted)

    // 查找匹配的 key（谨慎在生产环境使用）
    keys, _ := rdb.Keys(ctx, "user:*").Result()
    fmt.Println("匹配 user:* 的 keys:", keys)
}
```

### 主函数整合示例

```go
func main() {
    rdb := initClient()
    defer rdb.Close()

    stringExample(rdb)
    hashExample(rdb)
    listExample(rdb)
    setExample(rdb)
    zsetExample(rdb)
    keyExample(rdb)
}
```

> ⚠️ 注意：
>
> - 所有操作都应在 `context` 下进行（如 `ctx`）。
> - 实际项目中应妥善处理错误（此处为简洁部分用 `_` 忽略）。
> - `Keys()` 命令在大数据量下可能阻塞 Redis，建议用 `Scan` 替代。

### zset示例

下面的示例代码演示了如何使用 go-redis 库操作 zset。

```go
// zsetDemo 操作zset示例
func zsetDemo() {
	// key
	zsetKey := "language_rank"
	// value
	// 注意：v8版本使用[]*redis.Z；此处为v9版本使用[]redis.Z
	languages := []redis.Z{
		{Score: 90.0, Member: "Golang"},
		{Score: 98.0, Member: "Java"},
		{Score: 95.0, Member: "Python"},
		{Score: 97.0, Member: "JavaScript"},
		{Score: 99.0, Member: "C/C++"},
	}
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	// ZADD
	err := rdb.ZAdd(ctx, zsetKey, languages...).Err()
	if err != nil {
		fmt.Printf("zadd failed, err:%v\n", err)
		return
	}
	fmt.Println("zadd success")

	// 把Golang的分数加10
	newScore, err := rdb.ZIncrBy(ctx, zsetKey, 10.0, "Golang").Result()
	if err != nil {
		fmt.Printf("zincrby failed, err:%v\n", err)
		return
	}
	fmt.Printf("Golang's score is %f now.\n", newScore)

	// 取分数最高的3个
	ret := rdb.ZRevRangeWithScores(ctx, zsetKey, 0, 2).Val()
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}

	// 取95~100分的
	op := &redis.ZRangeBy{
		Min: "95",
		Max: "100",
	}
	ret, err = rdb.ZRangeByScoreWithScores(ctx, zsetKey, op).Result()
	if err != nil {
		fmt.Printf("zrangebyscore failed, err:%v\n", err)
		return
	}
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}
}
```

执行上面的函数将得到如下输出结果。

```bash
zadd success
Golang's score is 100.000000 now.
Golang 100
C/C++ 99
Java 98
Python 95
JavaScript 97
Java 98
C/C++ 99
Golang 100
```

### 扫描或遍历所有key

在Redis中可以使用[`KEYS prefix*`](https://redis.io/commands/keys/) 命令按前缀查询所有符合条件的 key，`go-redis`库中提供了`Keys`方法实现类似查询key的功能。

例如使用以下命令查询以`user:`为前缀的所有key（`user:cart:00`、`user:order:2023`等）。

```go
vals, err := rdb.Keys(ctx, "user:*").Result()
```

但是如果需要扫描数百万的 key ，那速度就会比较慢。这种场景下你可以使用`Scan`命令来遍历所有符合要求的 key。

```go
// scanKeysDemo1 按前缀查找所有key示例
func scanKeysDemo1() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	var cursor uint64
	for {
		var keys []string
		var err error
		// 将redis中所有以prefix:为前缀的key都扫描出来
		keys, cursor, err = rdb.Scan(ctx, cursor, "prefix:*", 0).Result()
		if err != nil {
			panic(err)
		}

		for _, key := range keys {
			fmt.Println("key", key)
		}

		if cursor == 0 { // no more keys
			break
		}
	}
}
```

针对这种需要遍历大量key的场景，`go-redis`中提供了一个简化方法——`Iterator`，其使用示例如下。

```go
// scanKeysDemo2 按前缀扫描key示例
func scanKeysDemo2() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()
	// 按前缀扫描key
	iter := rdb.Scan(ctx, 0, "prefix:*", 0).Iterator()
	for iter.Next(ctx) {
		fmt.Println("keys", iter.Val())
	}
	if err := iter.Err(); err != nil {
		panic(err)
	}
}
```

例如，我们可以写出一个将所有匹配指定模式的 key 删除的示例。

```go
// delKeysByMatch 按match格式扫描所有key并删除
func delKeysByMatch(match string, timeout time.Duration) {
	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel()

	iter := rdb.Scan(ctx, 0, match, 0).Iterator()
	for iter.Next(ctx) {
		err := rdb.Del(ctx, iter.Val()).Err()
		if err != nil {
			panic(err)
		}
	}
	if err := iter.Err(); err != nil {
		panic(err)
	}
}
```

此外，对于 Redis 中的 set、hash、zset 数据类型，`go-redis` 也支持类似的遍历方法。

```go
iter := rdb.SScan(ctx, "set-key", 0, "prefix:*", 0).Iterator()
iter := rdb.HScan(ctx, "hash-key", 0, "prefix:*", 0).Iterator()
iter := rdb.ZScan(ctx, "sorted-hash-key", 0, "prefix:*", 0).Iterator(
```

## Pipeline

Redis Pipeline 允许通过使用单个 client-server-client 往返执行多个命令来提高性能。区别于一个接一个地执行100个命令，你可以将这些命令放入 pipeline 中，然后使用1次读写操作像执行单个命令一样执行它们。这样做的好处是节省了执行命令的网络往返时间（RTT）。

在下面的示例代码中演示了使用 pipeline 通过一个 write + read 操作来执行多个命令。

```go
pipe := rdb.Pipeline()

incr := pipe.Incr(ctx, "pipeline_counter")
pipe.Expire(ctx, "pipeline_counter", time.Hour)

cmds, err := pipe.Exec(ctx)
if err != nil {
	panic(err)
}

// 在执行pipe.Exec之后才能获取到结果
fmt.Println(incr.Val())
```

上面的代码相当于将以下两个命令一次发给 Redis Server 端执行，与不使用 Pipeline 相比能减少一次RTT。

```bash
INCR pipeline_counter
EXPIRE pipeline_counts 3600
```

或者，你也可以使用`Pipelined` 方法，它会在函数退出时调用 Exec。

```go
var incr *redis.IntCmd

cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
	incr = pipe.Incr(ctx, "pipelined_counter")
	pipe.Expire(ctx, "pipelined_counter", time.Hour)
	return nil
})
if err != nil {
	panic(err)
}

// 在pipeline执行后获取到结果
fmt.Println(incr.Val())
```

我们可以遍历 pipeline 命令的返回值依次获取每个命令的结果。下方的示例代码中使用pipiline一次执行了100个 Get 命令，在pipeline 执行后遍历取出100个命令的执行结果。

```go
cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
	for i := 0; i < 100; i++ {
		pipe.Get(ctx, fmt.Sprintf("key%d", i))
	}
	return nil
})
if err != nil {
	panic(err)
}

for _, cmd := range cmds {
    fmt.Println(cmd.(*redis.StringCmd).Val())
}
```

在那些我们需要一次性执行多个命令的场景下，就可以考虑使用 pipeline 来优化。

> **`pipeline.Exec()` 会一次性执行所有排队的命令，并返回一个 `[]redis.Cmder` 切片（包含每个命令的结果对象）。**
>
> **如果需要某个命令的返回值（比如 `GET` 的内容），仍然要对 `Exec()` 返回的对应结果调用 `.Result()`！**

**1.普通命令（非 pipeline）：立即执行**

```go
cmd := rdb.Get(ctx, "key")        // ← 立即发送命令并等待响应
val, err := cmd.Result()          // ← 从响应中提取值和错误
```

- 每条命令**立刻发给 Redis**，立刻拿到结果对象。
- 你需要显式调用 `.Result()` 来获取值/错误。

**2.Pipeline 命令：延迟批量执行**

```go
pipe := rdb.Pipeline()
pipe.Get(ctx, "key1")   // ← 只是把命令“排队”，不发给 Redis！
pipe.Get(ctx, "key2")   // ← 继续排队
cmds, err := pipe.Exec() // ← 一次性发送所有命令，并等待所有结果
```

- `pipe.Get()`、`pipe.ZAdd()` 等**不会立即执行**，只是记录命令。
- **只有 `Exec()` 才真正与 Redis 通信**。
- `Exec()`返回：
  - `[]redis.Cmder`：每个命令对应的结果对象（顺序一致）
  - `error`：**如果整个 pipeline 执行过程中发生网络错误、连接断开等**

> ⚠️ 注意：**Redis 命令本身的错误（如语法错）通常不会让 `Exec()` 返回 error！**
> 而是体现在每个 `Cmder` 对象里，需要用 `.Err()` 检查。

### 如何获取Pipline中各个命令的结果

当你使用 `pipeline.Exec()`（或 `client.Pipeline().Exec()`、`TxPipeline().Exec()`）时：

- 它返回：

  ```go
  cmds []redis.Cmder, err error
  ```

- 其中 `cmds` 是一个 **命令结果对象的切片**，每个元素对应 pipeline 中**按顺序排队的一个命令**。

- 这些 `redis.Cmder` 对象（如 `*redis.StringCmd`, `*redis.IntCmd` 等）**在 `Exec()` 执行后才被填充实际结果**。

- 要从中**取出具体的值或错误**，你必须对**对应的 `Cmder` 对象调用 `.Result()` 或 `.Val()`**。

**✅ 正确做法示例**

```go
pipe := rdb.Pipeline()

// 排队命令（注意：保存返回的 Cmd 对象！）
getCmd := pipe.Get(ctx, "username")     // 返回 *StringCmd
zScoreCmd := pipe.ZScore(ctx, "leaderboard", "Alice") // 返回 *FloatCmd
setCmd := pipe.Set(ctx, "status", "ok", 0)            // 返回 *StatusCmd

// 执行所有命令
cmds, err := pipe.Exec()
if err != nil {
    // 处理网络/连接等系统级错误
    log.Fatal("Pipeline 执行失败:", err)
}

// ✅ 现在可以从各个 Cmd 对象中取值了！
// 方法1：用 .Result()（推荐）
username, err1 := getCmd.Result()       // (string, error)
score, err2 := zScoreCmd.Result()       // (float64, error)
status, err3 := setCmd.Result()         // (string, error)

// 方法2：用 .Val()（不推荐，除非确定不会出错）
username2 := getCmd.Val()   // string，出错时为 ""
score2 := zScoreCmd.Val()   // float64，出错时为 0.0
```

> 🔑 关键点：**你不需要通过 `cmds[0]`, `cmds[1]` 去索引**（虽然也可以），而是直接使用你之前保存的 `getCmd`, `zScoreCmd` 等变量——它们和 `cmds` 中的元素是**同一个对象**（指针引用）。

**❓ 为什么可以不用** `cmds[i]` **而直接用** `getCmd`？**

因为 `pipe.Get()` 返回的是一个指向内部命令对象的指针（如 `*StringCmd`）。当你调用 `pipe.Exec()` 时，**go-redis 会把 Redis 的响应写回到这些对象里**。所以：

```go
getCmd == cmds[0]  // true！是同一个对象
```

因此，**保存命令返回的 `Cmd` 变量是最清晰、最安全的方式**，避免了靠索引容易出错的问题。

**⚠️ 常见错误：忘记保存 Cmd 对象**

```go
// 没有保存返回的 Cmd
pipe.Get(ctx, "key1")
pipe.Get(ctx, "key2")

cmds, _ := pipe.Exec()

// 现在你无法直接获取 key1 的值！只能通过 cmds[0]
val := cmds[0].(*redis.StringCmd).Val() // 麻烦且易错
```

✅ 正确做法是**始终接收并保存命令返回的 Cmd 变量**。

## 事务

Redis 是单线程执行命令的，因此单个命令始终是原子的，但是来自不同客户端的两个给定命令可以依次执行，例如在它们之间交替执行。但是，`Multi/exec`能够确保在`multi/exec`两个语句之间的命令之间没有其他客户端正在执行命令。

在这种场景我们需要使用 TxPipeline 或 TxPipelined 方法将 pipeline 命令使用 `MULTI` 和`EXEC`包裹起来。

```go
// TxPipeline demo
pipe := rdb.TxPipeline()
incr := pipe.Incr(ctx, "tx_pipeline_counter")
pipe.Expire(ctx, "tx_pipeline_counter", time.Hour)
_, err := pipe.Exec(ctx)
fmt.Println(incr.Val(), err)

// TxPipelined demo
var incr2 *redis.IntCmd
_, err = rdb.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
	incr2 = pipe.Incr(ctx, "tx_pipeline_counter")
	pipe.Expire(ctx, "tx_pipeline_counter", time.Hour)
	return nil
})
fmt.Println(incr2.Val(), err)
```

上面代码相当于在一个RTT下执行了下面的redis命令：

```bash
MULTI
INCR pipeline_counter
EXPIRE pipeline_counts 3600
EXEC
```

### Watch

我们通常搭配 `WATCH`命令来执行事务操作。从使用`WATCH`命令监视某个 key 开始，直到执行`EXEC`命令的这段时间里，如果有其他用户抢先对被监视的 key 进行了替换、更新、删除等操作，那么当用户尝试执行`EXEC`的时候，事务将失败并返回一个错误，用户可以根据这个错误选择重试事务或者放弃事务。

Watch方法接收一个函数和一个或多个key作为参数。

```go
Watch(fn func(*Tx) error, keys ...string) error
```

下面的代码片段演示了 Watch 方法搭配 TxPipelined 的使用示例。

```go
// watchDemo 在key值不变的情况下将其值+1
func watchDemo(ctx context.Context, key string) error {
	return rdb.Watch(ctx, func(tx *redis.Tx) error {
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}
		// 假设操作耗时5秒
		// 5秒内我们通过其他的客户端修改key，当前事务就会失败
		time.Sleep(5 * time.Second)
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			pipe.Set(ctx, key, n+1, time.Hour)
			return nil
		})
		return err
	}, key)
}
```

将上面的函数执行并打印其返回值，如果我们在程序运行后的5秒内修改了被 watch 的 key 的值，那么该事务操作失败，返回`redis: transaction failed`错误。

最后我们来看一个 go-redis 官方文档中使用 `GET` 、`SET`和`WATCH`命令实现一个 INCR 命令的完整示例。

```go
// 此处rdb为初始化的redis连接客户端
const routineCount = 100

// 设置5秒超时
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// increment 是一个自定义对key进行递增（+1）的函数
// 使用 GET + SET + WATCH 实现，类似 INCR
increment := func(key string) error {
	txf := func(tx *redis.Tx) error {
		// 获得当前值或零值
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}

		// 实际操作（乐观锁定中的本地操作）
		n++

		// 仅在监视的Key保持不变的情况下运行
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			// pipe 处理错误情况
			pipe.Set(ctx, key, n, 0)
			return nil
		})
		return err
	}

	// 最多重试100次
	for retries := routineCount; retries > 0; retries-- {
		err := rdb.Watch(ctx, txf, key)
		if err != redis.TxFailedErr {
			return err
		}
		// 乐观锁丢失
	}
	return errors.New("increment reached maximum number of retries")
}

// 开启100个goroutine并发调用increment
// 相当于对key执行100次递增
var wg sync.WaitGroup
wg.Add(routineCount)
for i := 0; i < routineCount; i++ {
	go func() {
		defer wg.Done()

		if err := increment("counter3"); err != nil {
			fmt.Println("increment error:", err)
		}
	}()
}
wg.Wait()

n, err := rdb.Get(ctx, "counter3").Int()
fmt.Println("最终结果：", n, err)
```

在这个示例中使用了 `redis.TxFailedErr` 来检查事务是否失败。

更多详情请查阅[官方文档](https://redis.uptrace.dev/zh/)。