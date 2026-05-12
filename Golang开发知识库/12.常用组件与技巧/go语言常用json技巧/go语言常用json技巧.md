# go语言常用json技巧

本文总结了我平时在项目中遇到的那些关于go语言中数据与JSON格式之间相互转换的问题及解决办法。

### 基本的序列化

首先我们来看一下Go语言中`json.Marshal`（序列化）与`json.Unmarshal`（反序列化）的基本用法。

```go
type Person struct {
	Name   string
	Age    int64
	Weight float64
}

func main() {
	p1 := Person{
		Name:   "七米",
		Age:    18,
		Weight: 71.5,
	}
	// struct -> json string
	b, err := json.Marshal(p1)
	if err != nil {
		fmt.Printf("json.Marshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
	// json string -> struct
	var p2 Person
	err = json.Unmarshal(b, &p2)
	if err != nil {
		fmt.Printf("json.Unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("p2:%#v\n", p2)
}
```

输出：

```bash
str:{"Name":"七米","Age":18,"Weight":71.5}
p2:main.Person{Name:"七米", Age:18, Weight:71.5}
```

### 结构体tag介绍

`Tag`是结构体的元信息，可以在运行的时候通过反射的机制读取出来。 `Tag`在结构体字段的后方定义，由一对**反引号**包裹起来，具体的格式如下：

```bash
`key1:"value1" key2:"value2"`
```

结构体tag由一个或多个键值对组成。键与值使用**冒号**分隔，值用**双引号**括起来。同一个结构体字段可以设置多个键值对tag，不同的键值对之间使用**空格**分隔。

### 使用json tag指定字段名

序列化与反序列化默认情况下使用结构体的字段名，我们可以通过给结构体字段添加tag来指定json序列化生成的字段名。

```go
// 使用json tag指定序列化与反序列化时的行为
type Person struct {
	Name   string `json:"name"` // 指定json序列化/反序列化时使用小写name
	Age    int64
	Weight float64
}
```

### 忽略某个字段

如果你想在json序列化/反序列化的时候忽略掉结构体中的某个字段，可以按如下方式在tag中添加`-`。

```go
// 使用json tag指定json序列化与反序列化时的行为
type Person struct {
	Name   string `json:"name"` // 指定json序列化/反序列化时使用小写name
	Age    int64
	Weight float64 `json:"-"` // 指定json序列化/反序列化时忽略此字段
}
```

### 忽略空值字段

当 struct 中的字段没有值时， `json.Marshal()` 序列化的时候不会忽略这些字段，而是默认输出字段的类型零值（例如`int`和`float`类型零值是 0，`string`类型零值是`""`，对象类型零值是 nil）。如果想要在序列序列化时忽略这些没有值的字段时，可以在对应字段添加`omitempty` tag。

举个例子：

```go
type User struct {
	Name  string   `json:"name"`
	Email string   `json:"email"`
	Hobby []string `json:"hobby"`
}

func omitemptyDemo() {
	u1 := User{
		Name: "七米",
	}
	// struct -> json string
	b, err := json.Marshal(u1)
	if err != nil {
		fmt.Printf("json.Marshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
}
```

输出结果：

```go
str:{"name":"七米","email":"","hobby":null}
```

如果想要在最终的序列化结果中去掉空值字段，可以像下面这样定义结构体：

```go
// 在tag中添加omitempty忽略空值
// 注意这里 hobby,omitempty 合起来是json tag值，中间用英文逗号分隔
type User struct {
	Name  string   `json:"name"`
	Email string   `json:"email,omitempty"`
	Hobby []string `json:"hobby,omitempty"`
}
```

此时，再执行上述的`omitemptyDemo`，输出结果如下：

```bash
str:{"name":"七米"} // 序列化结果中没有email和hobby字段
```

### 忽略嵌套结构体空值字段

首先来看几种结构体嵌套的示例：

```go
type User struct {
	Name  string   `json:"name"`
	Email string   `json:"email,omitempty"`
	Hobby []string `json:"hobby,omitempty"`
	Profile
}

type Profile struct {
	Website string `json:"site"`
	Slogan  string `json:"slogan"`
}

func nestedStructDemo() {
	u1 := User{
		Name:  "七米",
		Hobby: []string{"足球", "双色球"},
	}
	b, err := json.Marshal(u1)
	if err != nil {
		fmt.Printf("json.Marshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
}
```

匿名嵌套`Profile`时序列化后的json串为单层的：

```bash
str:{"name":"七米","hobby":["足球","双色球"],"site":"","slogan":""}
```

想要变成嵌套的json串，需要改为具名嵌套或定义字段tag：

```go
type User struct {
	Name    string   `json:"name"`
	Email   string   `json:"email,omitempty"`
	Hobby   []string `json:"hobby,omitempty"`
	Profile `json:"profile"`
}
// str:{"name":"七米","hobby":["足球","双色球"],"profile":{"site":"","slogan":""}}
```

想要在嵌套的结构体为空值时，忽略该字段，仅添加`omitempty`是不够的：

```go
type User struct {
	Name     string   `json:"name"`
	Email    string   `json:"email,omitempty"`
	Hobby    []string `json:"hobby,omitempty"`
	Profile `json:"profile,omitempty"`
}
// str:{"name":"七米","hobby":["足球","双色球"],"profile":{"site":"","slogan":""}}
```

还需要使用嵌套的结构体指针：

```go
type User struct {
	Name     string   `json:"name"`
	Email    string   `json:"email,omitempty"`
	Hobby    []string `json:"hobby,omitempty"`
	*Profile `json:"profile,omitempty"`
}
// str:{"name":"七米","hobby":["足球","双色球"]}
```

### 不修改原结构体忽略空值字段

我们需要json序列化`User`，但是不想把密码也序列化，又不想修改`User`结构体，这个时候我们就可以使用创建另外一个结构体`PublicUser`匿名嵌套原`User`，同时指定`Password`字段为匿名结构体指针类型，并添加`omitempty`tag，示例代码如下：

```go
type User struct {
	Name     string `json:"name"`
	Password string `json:"password"`
}

type PublicUser struct {
	*User             // 匿名嵌套
	Password *struct{} `json:"password,omitempty"`
}

func omitPasswordDemo() {
	u1 := User{
		Name:     "七米",
		Password: "123456",
	}
	b, err := json.Marshal(PublicUser{User: &u1})
	if err != nil {
		fmt.Printf("json.Marshal u1 failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)  // str:{"name":"七米"}
}
```

### 优雅处理字符串格式的数字

有时候，前端在传递来的json数据中可能会使用字符串类型的数字，这个时候可以在结构体tag中添加`string`来告诉json包从字符串中解析相应字段的数据：

```go
type Card struct {
	ID    int64   `json:"id,string"`    // 添加string tag
	Score float64 `json:"score,string"` // 添加string tag
}

func intAndStringDemo() {
	jsonStr1 := `{"id": "1234567","score": "88.50"}`
	var c1 Card
	if err := json.Unmarshal([]byte(jsonStr1), &c1); err != nil {
		fmt.Printf("json.Unmarsha jsonStr1 failed, err:%v\n", err)
		return
	}
	fmt.Printf("c1:%#v\n", c1) // c1:main.Card{ID:1234567, Score:88.5}
}
```

### 整数变浮点数

因为在 JSON 协议中是没有整型和浮点型之分的，它们统称为number。json字符串中的数字经过Go语言中的json包反序列化之后都会成为`float64`类型。

通常这并不会有什么问题，但是在某些特殊场景下就会产生意想不到的结果。比如，将JSON格式的数据反序列化为`map[string]interface{}`时，数字都变成科学计数法表示的浮点数。

```go
// useNumberDemo 使用json.UseNumber
// 解决将JSON数据反序列化成map[string]interface{}时
// 数字变为科学计数法表示的浮点数问题
func useNumberDemo(){
	type student struct {
		ID int64 `json:"id"`
		Name string `json:"q1mi"`
	}
	s := student{ID: 123456789,Name: "q1mi"}
	b, _ := json.Marshal(s)
	var m map[string]interface{}
	// decode
	json.Unmarshal(b, &m)
	fmt.Printf("id:%#v\n", m["id"])  // 1.23456789e+08
	fmt.Printf("id type:%T\n", m["id"])  //float64

	// use Number decode
	decoder := json.NewDecoder(bytes.NewReader(b))
	decoder.UseNumber()
	decoder.Decode(&m)
	fmt.Printf("id:%#v\n", m["id"])  // "123456789"
	fmt.Printf("id type:%T\n", m["id"]) // json.Number
}
```

这种问题通常出现在将JSON格式数据反序列化为`map[string]interface{}`时，再来一个示例。

```go
func jsonDemo() {
	// map[string]interface{} -> json string
	var m = make(map[string]interface{}, 1)
	m["count"] = 1 // int
	b, err := json.Marshal(m)
	if err != nil {
		fmt.Printf("marshal failed, err:%v\n", err)
	}
	fmt.Printf("str:%#v\n", string(b))
	// json string -> map[string]interface{}
	var m2 map[string]interface{}
	err = json.Unmarshal(b, &m2)
	if err != nil {
		fmt.Printf("unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("value:%v\n", m2["count"]) // 1
	fmt.Printf("type:%T\n", m2["count"])  // float64
}
```

这种场景下如果想更合理的处理数字就需要使用`decoder`去反序列化，示例代码如下：

```go
func decoderDemo() {
	// map[string]interface{} -> json string
	var m = make(map[string]interface{}, 1)
	m["count"] = 1 // int
	b, err := json.Marshal(m)
	if err != nil {
		fmt.Printf("marshal failed, err:%v\n", err)
	}
	fmt.Printf("str:%#v\n", string(b))
	// json string -> map[string]interface{}
	var m2 map[string]interface{}
	// 使用decoder方式反序列化，指定使用number类型
	decoder := json.NewDecoder(bytes.NewReader(b))
	decoder.UseNumber()
	err = decoder.Decode(&m2)
	if err != nil {
		fmt.Printf("unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("value:%v\n", m2["count"]) // 1
	fmt.Printf("type:%T\n", m2["count"])  // json.Number
	// 将m2["count"]转为json.Number之后调用Int64()方法获得int64类型的值
	count, err := m2["count"].(json.Number).Int64()
	if err != nil {
		fmt.Printf("parse to int64 failed, err:%v\n", err)
		return
	}
	fmt.Printf("type:%T\n", int(count)) // int
}
```

`json.Number`的源码定义如下：

```go
// A Number represents a JSON number literal.
type Number string

// String returns the literal text of the number.
func (n Number) String() string { return string(n) }

// Float64 returns the number as a float64.
func (n Number) Float64() (float64, error) {
	return strconv.ParseFloat(string(n), 64)
}

// Int64 returns the number as an int64.
func (n Number) Int64() (int64, error) {
	return strconv.ParseInt(string(n), 10, 64)
}
```

我们在处理number类型的json字段时需要先得到`json.Number`类型，然后根据该字段的实际类型调用`Float64()`或`Int64()`。

### 自定义解析时间字段

Go语言内置的 json 包使用 `RFC3339` 标准中定义的时间格式，对我们序列化时间字段的时候有很多限制。

```go
type Post struct {
	CreateTime time.Time `json:"create_time"`
}

func timeFieldDemo() {
	p1 := Post{CreateTime: time.Now()}
	b, err := json.Marshal(p1)
	if err != nil {
		fmt.Printf("json.Marshal p1 failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
	jsonStr := `{"create_time":"2020-04-05 12:25:42"}`
	var p2 Post
	if err := json.Unmarshal([]byte(jsonStr), &p2); err != nil {
		fmt.Printf("json.Unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("p2:%#v\n", p2)
}
```

上面的代码输出结果如下：

```go
str:{"create_time":"2020-04-05T12:28:06.799214+08:00"}
json.Unmarshal failed, err:parsing time ""2020-04-05 12:25:42"" as ""2006-01-02T15:04:05Z07:00"": cannot parse " 12:25:42"" as "T"
```

也就是内置的json包不识别我们常用的字符串时间格式，如`2020-04-05 12:25:42`。

不过我们通过实现 `json.Marshaler`/`json.Unmarshaler` 接口实现自定义的事件格式解析。

```go
type CustomTime struct {
	time.Time
}

const ctLayout = "2006-01-02 15:04:05"

var nilTime = (time.Time{}).UnixNano()

func (ct *CustomTime) UnmarshalJSON(b []byte) (err error) {
	s := strings.Trim(string(b), "\"")
	if s == "null" {
		ct.Time = time.Time{}
		return
	}
	ct.Time, err = time.Parse(ctLayout, s)
	return
}

func (ct *CustomTime) MarshalJSON() ([]byte, error) {
	if ct.Time.UnixNano() == nilTime {
		return []byte("null"), nil
	}
	return []byte(fmt.Sprintf("\"%s\"", ct.Time.Format(ctLayout))), nil
}

func (ct *CustomTime) IsSet() bool {
	return ct.UnixNano() != nilTime
}

type Post struct {
	CreateTime CustomTime `json:"create_time"`
}

func timeFieldDemo() {
	p1 := Post{CreateTime: CustomTime{time.Now()}}
	b, err := json.Marshal(p1)
	if err != nil {
		fmt.Printf("json.Marshal p1 failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
	jsonStr := `{"create_time":"2020-04-05 12:25:42"}`
	var p2 Post
	if err := json.Unmarshal([]byte(jsonStr), &p2); err != nil {
		fmt.Printf("json.Unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("p2:%#v\n", p2)
}
```

### 自定义MarshalJSON和UnmarshalJSON方法

上面那种自定义类型的方法稍显啰嗦了一点，下面来看一种相对便捷的方法。

首先你需要知道的是，如果你能够为某个类型实现了`MarshalJSON()([]byte, error)`和`UnmarshalJSON(b []byte) error`方法，那么这个类型在序列化（MarshalJSON）/反序列化（UnmarshalJSON）时就会使用你定制的相应方法。

```go
type Order struct {
	ID          int       `json:"id"`
	Title       string    `json:"title"`
	CreatedTime time.Time `json:"created_time"`
}

const layout = "2006-01-02 15:04:05"

// MarshalJSON 为Order类型实现自定义的MarshalJSON方法
func (o *Order) MarshalJSON() ([]byte, error) {
	type TempOrder Order // 定义与Order字段一致的新类型
	return json.Marshal(struct {
		CreatedTime string `json:"created_time"`
		*TempOrder         // 避免直接嵌套Order进入死循环
	}{
		CreatedTime: o.CreatedTime.Format(layout),
		TempOrder:   (*TempOrder)(o),
	})
}

// UnmarshalJSON 为Order类型实现自定义的UnmarshalJSON方法
func (o *Order) UnmarshalJSON(data []byte) error {
	type TempOrder Order // 定义与Order字段一致的新类型
	ot := struct {
		CreatedTime string `json:"created_time"`
		*TempOrder         // 避免直接嵌套Order进入死循环
	}{
		TempOrder: (*TempOrder)(o),
	}
	if err := json.Unmarshal(data, &ot); err != nil {
		return err
	}
	var err error
	o.CreatedTime, err = time.Parse(layout, ot.CreatedTime)
	if err != nil {
		return err
	}
	return nil
}

// 自定义序列化方法
func customMethodDemo() {
	o1 := Order{
		ID:          123456,
		Title:       "《七米的Go学习笔记》",
		CreatedTime: time.Now(),
	}
	// 通过自定义的MarshalJSON方法实现struct -> json string
	b, err := json.Marshal(&o1)
	if err != nil {
		fmt.Printf("json.Marshal o1 failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
	// 通过自定义的UnmarshalJSON方法实现json string -> struct
	jsonStr := `{"created_time":"2020-04-05 10:18:20","id":123456,"title":"《七米的Go学习笔记》"}`
	var o2 Order
	if err := json.Unmarshal([]byte(jsonStr), &o2); err != nil {
		fmt.Printf("json.Unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("o2:%#v\n", o2)
}
```

输出结果：

```bash
str:{"created_time":"2020-04-05 10:32:20","id":123456,"title":"《七米的Go学习笔记》"}
o2:main.Order{ID:123456, Title:"《七米的Go学习笔记》", CreatedTime:time.Time{wall:0x0, ext:63721678700, loc:(*time.Location)(nil)}}
```

### 使用匿名结构体添加字段

使用内嵌结构体能够扩展结构体的字段，但有时候我们没有必要单独定义新的结构体，可以使用匿名结构体简化操作：

```go
type UserInfo struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

func anonymousStructDemo() {
	u1 := UserInfo{
		ID:   123456,
		Name: "七米",
	}
	// 使用匿名结构体内嵌User并添加额外字段Token
	b, err := json.Marshal(struct {
		*UserInfo
		Token string `json:"token"`
	}{
		&u1,
		"91je3a4s72d1da96h",
	})
	if err != nil {
		fmt.Printf("json.Marsha failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
	// str:{"id":123456,"name":"七米","token":"91je3a4s72d1da96h"}
}
```

### 使用匿名结构体组合多个结构体

同理，也可以使用匿名结构体来组合多个结构体来序列化与反序列化数据：

```go
type Comment struct {
	Content string
}

type Image struct {
	Title string `json:"title"`
	URL   string `json:"url"`
}

func anonymousStructDemo2() {
	c1 := Comment{
		Content: "永远不要高估自己",
	}
	i1 := Image{
		Title: "赞赏码",
		URL:   "https://www.liwenzhou.com/images/zanshang_qr.jpg",
	}
	// struct -> json string
	b, err := json.Marshal(struct {
		*Comment
		*Image
	}{&c1, &i1})
	if err != nil {
		fmt.Printf("json.Marshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
	// json string -> struct
	jsonStr := `{"Content":"永远不要高估自己","title":"赞赏码","url":"https://www.liwenzhou.com/images/zanshang_qr.jpg"}`
	var (
		c2 Comment
		i2 Image
	)
	if err := json.Unmarshal([]byte(jsonStr), &struct {
		*Comment
		*Image
	}{&c2, &i2}); err != nil {
		fmt.Printf("json.Unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("c2:%#v i2:%#v\n", c2, i2)
}
```

输出：

```go
str:{"Content":"永远不要高估自己","title":"赞赏码","url":"https://www.liwenzhou.com/images/zanshang_qr.jpg"}
c2:main.Comment{Content:"永远不要高估自己"} i2:main.Image{Title:"赞赏码", URL:"https://www.liwenzhou.com/images/zanshang_qr.jpg"}
```

### 处理不确定层级的json

如果json串没有固定的格式导致不好定义与其相对应的结构体时，我们可以使用`json.RawMessage`原始字节数据保存下来。

```go
type sendMsg struct {
	User string `json:"user"`
	Msg  string `json:"msg"`
}

func rawMessageDemo() {
	jsonStr := `{"sendMsg":{"user":"q1mi","msg":"永远不要高估自己"},"say":"Hello"}`
	// 定义一个map，value类型为json.RawMessage，方便后续更灵活地处理
	var data map[string]json.RawMessage
	if err := json.Unmarshal([]byte(jsonStr), &data); err != nil {
		fmt.Printf("json.Unmarshal jsonStr failed, err:%v\n", err)
		return
	}
	var msg sendMsg
	if err := json.Unmarshal(data["sendMsg"], &msg); err != nil {
		fmt.Printf("json.Unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("msg:%#v\n", msg)
	// msg:main.sendMsg{User:"q1mi", Msg:"永远不要高估自己"}
}
```

### 序列化时不转义

json包中的`encoder`可以通过`SetEscapeHTML`指定是否应该在JSON字符串中转义有问题的HTML字符。其默认行为是将`&`、`<`和`>`转义为`\u0026`、`\u003c`和`\u003e`，以避免在HTML中嵌入JSON时可能出现的某些安全问题。

如果是非HTML场景下不想被转义，可以通过`SetEscapeHTML(false)`禁用此行为。

例如有些业务场景下可能需要序列化带查询参数的URL，这种场景下我们并不希望转义`&`符号。

```go
// URLInfo 一个包含URL字段的结构体
type URLInfo struct {
	URL string
	// ...
}

// JSONEncodeDontEscapeHTML json序列化时不转义 &, < 和 >
// & \u0026
// < \u003c
// > \u003e
func JSONEncodeDontEscapeHTML(data URLInfo) {
	b, err := json.Marshal(data)
	if err != nil {
		fmt.Printf("json.Marshal(data) failed, err:%v\n", err)
	}
	fmt.Printf("json.Marshal(data) result:%s\n", b)

	buf := bytes.Buffer{}
	encoder := json.NewEncoder(&buf)
	encoder.SetEscapeHTML(false) // 告知encoder不转义
	if err := encoder.Encode(data); err != nil {
		fmt.Printf("encoder.Encode(data) failed, err:%v\n", err)
	}
	fmt.Printf("encoder.Encode(data) result:%s\n", buf.String())
}
```

输出：

```bash
json.Marshal(data) result:{"URL":"https://liwenzhou.com?name=q1mi\u0026age=18"}
encoder.Encode(data) result:{"URL":"https://liwenzhou.com?name=q1mi&age=18"}
```

### 雪花算法生成的 ID 失真（精度丢失）

1. 后端（Go/Java 等） ： int64 类型是 64 位有符号整数，最大值约为`9.22 × 10^18`（即 9223372036854775807 ）。雪花算法生成的 ID 也是 64 位的，通常会用到 16-19 位数字。
2. 前端（JavaScript） ：JS 中的 Number 类型遵循 IEEE 754 标准，本质上是 双精度浮点数 (double) 。它能精确表示的最大安全整数（ Number.MAX_SAFE_INTEGER ）是 (2^53) − 1 ，即 9007199254740991 （约 16 位数字）。
3. 冲突 ：当后端传给前端一个大于 2^53 的 ID（例如 14283784123846656 ，这是 17 位数），JS 在解析 JSON 时会因为它超出了精度范围而将其近似存储，导致最后几位数字变成 0 或其他错误的数字。例如`...6656`可能变成`...6600`。
4. 后果 ：前端拿这个错误的 ID 去请求详情页或进行操作时，后端查不到对应数据，导致 Bug。

**解决方案**

解决这个问题的核心思路是： 在后端序列化成 JSON 时，把 ID 转成字符串（String）。JS 处理字符串是完全没问题的。

 **方案一：在 Struct Tag 中添加 string 选项（推荐）**

Go 语言的 encoding/json 库提供了一个便捷的 tag 选项`,string`。加上它之后，Go 在序列化时会自动把数值转为字符串，反序列化时自动把字符串转为数值。

优点 ：改动最小，原生支持。 缺点 ：所有用到该 ID 的地方都需要加 tag。

你需要修改的文件 ：

检查所有包含 int64 类型 ID 的结构体（主要是 Post 和 Community 相关）。

models/post.go

```go
type Post struct {
	ID          int64     `json:"id,string" db:"post_id"` // 这个字段加一个,string
	AuthorID    int64     `json:"author_id,string" db:"author_id"`
	CommunityID int64     `json:"community_id,string" db:"community_id" binding:"required"`
	Status      int32     `json:"status" db:"status"`
	Title       string    `json:"title" db:"title" binding:"required"`
	Content     string    `json:"content" db:"content" binding:"required"`
	CreateTime  time.Time `json:"create_time" db:"create_time"`
}
```

models/community.go

```go
type Community struct {
	ID   int64  `json:"id,string" db:"community_id"` //
	Name string `json:"name" db:"community_name"`
}

type CommunityDetail struct {
	ID           int64     `json:"id,string" db:"community_id"` // 
	Name         string    `json:"name" db:"community_name"`
	Introduction string    `json:"introduction,omitempty" db:"introduction"`
	CreateTime   time.Time `json:"create_time" db:"create_time"`
}
```

**方案二：自定义类型 + 实现 json.Marshaler**

为 ID 类型定义一个别名（如 type ID int64 ），并为该类型实现 MarshalJSON 接口，手动转为字符串。这比较麻烦，通常不如方案一直接。

"`json.Marshaler` 是 Go 标准库 `encoding/json` 包里的一个接口，就定义了一个方法 `MarshalJSON() ([]byte, error)`。

当调用 `json.Marshal()` 序列化的时候，Go 会用反射检查每个字段是否实现了这个接口。如果实现了，就调用它的 `MarshalJSON()` 方法，用返回的字节作为 JSON 输出。

所以我们自定义一个 `Int64String` 类型，实现这个方法，返回带双引号的字符串，序列化的时候就自动转成 JSON 字符串了。

这其实是 Go 接口多态的一个典型应用，类似的设计在标准库里还有很多，比如 `fmt.Stringer`、`io.Reader` 这些。"

```go
// 自定义一个 Int64String 类型
type Int64String int64

// 实现 json.Marshaler 接口
func (i Int64String) MarshalJSON() ([]byte, error) {
    return []byte(`"` + strconv.FormatInt(int64(i), 10) + `"`), nil
}

// 使用的时候
type UserDTO struct {
    UserID Int64String `json:"userId"`
    Name   string      `json:"name"`
}
```

**优点**：一次定义，到处使用，不容易忘
**缺点**：需要多一层类型转换

**方案三：前端使用 BigInt 库**

让前端引入 json-bigint 等库来替代原生的 JSON.parse 。这会增加前端的负担和包体积，通常作为后端无法修改时的备选方案。

**知识点讲解**

- 问题根源 ：JavaScript 的 Number 类型最大安全整数是 [ o bj ec tO bj ec t ] 2 53 − 1 ，而 Go 的 int64 （雪花算法 ID）范围远超此值。超过安全范围的整数在 JS 中会发生精度丢失（低位变 0）。
- 解决方法 ：在数据传给前端前，将大整数转换为 字符串 。JS 处理字符串没有精度问题。
- Go 实践 ：在结构体标签中使用 json:"field_name,string" 。这个 ,string 选项告诉 JSON 序列化器：
  - 序列化（Marshal）：把数字转成字符串输出（如 123 -> "123" ）。
  - 反序列化（Unmarshal）：把字符串解析回数字（如 "123" -> 123 ）。

参考链接：

https://stackoverflow.com/questions/25087960/json-unmarshal-time-that-isnt-in-rfc-3339-format

https://colobu.com/2017/06/21/json-tricks-in-Go/

https://stackoverflow.com/questions/11066946/partly-json-unmarshal-into-a-map-in-go

http://choly.ca/post/go-json-marshalling/