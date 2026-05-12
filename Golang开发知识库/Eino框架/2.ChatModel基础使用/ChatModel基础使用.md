# ChatModel 基础使用

> 参考文档: https://www.cloudwego.io/zh/docs/eino/core_modules/components/chat_model_guide/

## 什么是 ChatModel？

ChatModel 是 Eino 中用于与大语言模型交互的核心组件。它提供了统一的接口来调用不同的大模型服务（如 OpenAI、Claude、ARK 等）。

### ChatModel 的核心能力

- 💬 **对话生成**: Generate() - 一次性获取完整响应
- 🌊 **流式输出**: Stream() - 实时流式获取响应（类似打字机效果）
- 🔧 **工具调用**: 支持 Function Calling / Tool Calling
- 🎛️ **参数控制**: Temperature、Top-P、Max Tokens 等

## 基础示例：Generate 生成

### 单轮对话

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	"github.com/cloudwego/eino-ext/components/model/deepseek"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	// 创建 ChatModel
	chatModel, err := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
		APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
		Model:   "deepseek-chat",
		BaseURL: "https://api.deepseek.com",
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	// 构建消息
	messages := []*schema.Message{
		schema.SystemMessage("你是一个专业的 Go 语言工程师"),
		schema.UserMessage("请解释 Go 语言中的 goroutine 是什么？"),
	}

	// 生成响应
	response, err := chatModel.Generate(ctx, messages)
	if err != nil {
		log.Fatalf("生成失败: %v", err)
	}

	fmt.Printf("回答:\\n%s\\n", response.Content)
}
```

### 多轮对话

```go
package main

import (
	"bufio"
	"context"
	"fmt"
	"log"
	"os"
	"strings"

	"github.com/cloudwego/eino-ext/components/model/deepseek"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	chatModel, err := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
		APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
		Model:   "deepseek-chat",
		BaseURL: "https://api.deepseek.com",
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	// 对话历史
	messages := []*schema.Message{
		schema.SystemMessage("你是一个友好的 AI 助手"),
	}

	scanner := bufio.NewScanner(os.Stdin)
	fmt.Println("开始对话（输入 'exit' 退出）：")

	for {
		fmt.Print("\\n你: ")
		if !scanner.Scan() {
			break
		}

		userInput := strings.TrimSpace(scanner.Text())
		if userInput == "exit" {
			fmt.Println("再见！")
			break
		}

		if userInput == "" {
			continue
		}

		// 添加用户消息
		messages = append(messages, schema.UserMessage(userInput))

		// 生成 AI 响应
		response, err := chatModel.Generate(ctx, messages)
		if err != nil {
			log.Printf("生成失败: %v", err)
			continue
		}

		// 添加 AI 响应到历史
		messages = append(messages, response)

		fmt.Printf("\\nAI: %s\\n", response.Content)
	}
}
```

## 流式输出：Stream

流式输出可以让用户实时看到 AI 的回复过程，提升用户体验。

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"log"
	"os"

	"github.com/cloudwego/eino-ext/components/model/deepseek"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	chatModel, err := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
		APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
		Model:   "deepseek-chat",
		BaseURL: "https://api.deepseek.com",
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	messages := []*schema.Message{
		schema.SystemMessage("你是一个专业的技术博主"),
		schema.UserMessage("请写一篇关于 Go 并发编程的短文（200字左右）"),
	}

	// 流式生成
	stream, err := chatModel.Stream(ctx, messages)
	if err != nil {
		log.Fatalf("流式生成失败: %v", err)
	}
	defer stream.Close() // 记得关闭流

	fmt.Print("AI 回复: ")

	// 逐块接收并打印
	for {
		chunk, err := stream.Recv()
		if err != nil {
			if errors.Is(err, io.EOF) {
				// 流结束
				break
			}
			log.Fatalf("接收失败: %v", err)
		}

		// 打印内容（打字机效果）
		fmt.Print(chunk.Content)
	}

	fmt.Println("\\n\\n完成！")
}
```

### 流式输出 - 收集完整响应

有时我们需要在流式接收的同时，也保存完整的响应内容：

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"log"
	"os"
	"strings"

	"github.com/cloudwego/eino-ext/components/model/deepseek"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	chatModel, err := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
		APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
		Model:   "deepseek-chat",
		BaseURL: "https://api.deepseek.com",
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	messages := []*schema.Message{
		schema.UserMessage("请列举 5 个 Go 语言的特点"),
	}

	stream, err := chatModel.Stream(ctx, messages)
	if err != nil {
		log.Fatalf("流式生成失败: %v", err)
	}
	defer stream.Close()

	var fullContent strings.Builder
	fmt.Print("AI 回复: ")

	for {
		chunk, err := stream.Recv()
		if err != nil {
			if errors.Is(err, io.EOF) {
				break
			}
			log.Fatalf("接收失败: %v", err)
		}

		// 实时打印
		fmt.Print(chunk.Content)
		
		// 同时收集完整内容
		fullContent.WriteString(chunk.Content)
	}

	fmt.Println("\\n\\n======== 完整响应 ========")
	fmt.Println(fullContent.String())
}
```

## 参数配置

ChatModel 支持多种参数来控制生成行为。

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	"github.com/cloudwego/eino-ext/components/model/deepseek"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	// 示例1: 基础配置
	fmt.Println("=== 示例1: 基础配置 ===")
	basicExample(ctx)

	// 示例2: 高级配置
	fmt.Println("\\n=== 示例2: 高级配置 ===")
	advancedExample(ctx)

	// 示例3: 创意写作配置
	fmt.Println("\\n=== 示例3: 创意写作配置 ===")
	creativeExample(ctx)
}

// 基础配置示例
func basicExample(ctx context.Context) {
	chatModel, err := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
		APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
		Model:   "deepseek-chat",
		BaseURL: "https://api.deepseek.com",
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	messages := []*schema.Message{
		schema.SystemMessage("你是一个友好的 AI 助手"),
		schema.UserMessage("用一句话介绍 Eino 框架"),
	}

	response, err := chatModel.Generate(ctx, messages)
	if err != nil {
		log.Fatalf("生成失败: %v", err)
	}

	fmt.Printf("AI 响应: %s\\n", response.Content)
	printTokenUsage(response)
}

// 高级配置示例 - 精确控制输出
func advancedExample(ctx context.Context) {
	chatModel, err := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
		// 基础配置
		APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
		Model:   "deepseek-chat",
		BaseURL: "https://api.deepseek.com",
		Timeout: 30 * time.Second,

		// 生成参数
		Temperature: 0.7, // 控制输出随机性，范围 [0.0, 2.0]，越高越随机
		TopP:        0.9, // 核采样参数，范围 [0.0, 1.0]，越低越聚焦
		MaxTokens:   500, // 限制最大生成 token 数量，范围 [1, 8192]

		// 停止序列 - 遇到这些文本时停止生成
		Stop: []string{"\\n\\n", "总结:"},

		// 惩罚参数 - 控制重复度
		PresencePenalty:  0.6, // 存在惩罚，范围 [-2.0, 2.0]，正值增加新话题可能性
		FrequencyPenalty: 0.5, // 频率惩罚，范围 [-2.0, 2.0]，正值减少重复词语
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	messages := []*schema.Message{
		schema.SystemMessage("你是一个专业的技术文档撰写专家"),
		schema.UserMessage("详细介绍 Eino 框架的核心特性，包括架构、组件和优势"),
	}

	response, err := chatModel.Generate(ctx, messages)
	if err != nil {
		log.Fatalf("生成失败: %v", err)
	}

	fmt.Printf("AI 响应: %s\\n", response.Content)
	printTokenUsage(response)
}

// 创意写作配置示例 - 高随机性
func creativeExample(ctx context.Context) {
	chatModel, err := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
		APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
		Model:   "deepseek-chat",
		BaseURL: "https://api.deepseek.com",

		// 高温度设置，适合创意写作
		Temperature: 1.2,  // 更高的随机性
		TopP:        0.95, // 保留更多可能性
		MaxTokens:   800,

		// 减少重复惩罚，允许一定的重复（适合故事情节）
		PresencePenalty:  0.3,
		FrequencyPenalty: 0.3,
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	messages := []*schema.Message{
		schema.SystemMessage("你是一个富有创造力的故事作家"),
		schema.UserMessage("创作一个关于 AI 框架变成超级英雄的有趣故事开头"),
	}

	response, err := chatModel.Generate(ctx, messages)
	if err != nil {
		log.Fatalf("生成失败: %v", err)
	}

	fmt.Printf("AI 响应: %s\\n", response.Content)
	printTokenUsage(response)
}

// 打印 Token 使用情况
func printTokenUsage(response *schema.Message) {
	if response.ResponseMeta != nil && response.ResponseMeta.Usage != nil {
		fmt.Printf("\\nToken 使用统计:\\n")
		fmt.Printf("  输入 Token: %d\\n", response.ResponseMeta.Usage.PromptTokens)
		fmt.Printf("  输出 Token: %d\\n", response.ResponseMeta.Usage.CompletionTokens)
		fmt.Printf("  总计 Token: %d\\n", response.ResponseMeta.Usage.TotalTokens)
		if response.ResponseMeta.Usage.PromptTokenDetails.CachedTokens > 0 {
			fmt.Printf("  缓存 Token: %d\\n", response.ResponseMeta.Usage.PromptTokenDetails.CachedTokens)
		}
	}
}
```

### 参数说明

| 参数                 | 范围         | 说明                     | 建议使用场景                                                 |
| :------------------- | :----------- | :----------------------- | :----------------------------------------------------------- |
| **Temperature**      | 0.0 - 2.0    | 控制输出的随机性和创造性 | 0.0-0.3: 事实性任务、代码生成0.7-1.0: 通用对话1.0-2.0: 创意写作 |
| **TopP**             | 0.0 - 1.0    | 核采样，控制候选词范围   | 通常设置 0.9，与 Temperature 配合使用                        |
| **MaxTokens**        | 1 - 模型上限 | 限制生成的最大 token 数  | 根据需求设置，防止过长输出                                   |
| **FrequencyPenalty** | -2.0 - 2.0   | 降低重复词频率           | 需要避免重复时使用                                           |
| **PresencePenalty**  | -2.0 - 2.0   | 鼓励谈论新话题           | 需要多样性时使用                                             |

## 使用不同的模型服务

### ARK (字节跳动豆包)

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	"github.com/cloudwego/eino-ext/components/model/ark"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	chatModel, err := ark.NewChatModel(ctx, &ark.ChatModelConfig{
		APIKey: os.Getenv("ARK_API_KEY"),
		Model:  os.Getenv("ARK_MODEL_NAME"), // ep-xxxxx
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	messages := []*schema.Message{
		schema.UserMessage("你好，请介绍一下自己"),
	}

	response, err := chatModel.Generate(ctx, messages)
	if err != nil {
		log.Fatalf("生成失败: %v", err)
	}

	fmt.Printf("回答: %s\\n", response.Content)
}
```

### Ollama (本地模型)

Ollama 是一个开源工具，可以在本地运行大语言模型，无需联网，完全免费。

#### Ollama 安装步骤

##### 1. 下载安装 Ollama

**Windows:**

```bash
# 访问 Ollama 官网下载安装包
https://ollama.com/download/windows

# 或使用 winget 安装
winget install Ollama.Ollama
```

**MacOS:**

```bash
# 下载 .dmg 安装包
https://ollama.com/download/mac

# 或使用 Homebrew
brew install ollama
```

**Linux:**

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

##### 2. 启动 Ollama 服务

安装完成后，Ollama 会自动在后台运行。

**手动启动**（如果需要）：

```bash
# Windows/Linux/MacOS
ollama serve
```

默认运行在 `http://localhost:11434`

##### 3. 下载模型

Ollama 支持多种开源模型，常用模型推荐：

```bash
# 下载 Llama 3.2（Meta 最新模型，推荐）
ollama pull llama3.2

# 下载 Qwen 2.5（阿里通义千问，中文友好）
ollama pull qwen2.5

# 下载 Mistral（轻量级，速度快）
ollama pull mistral

# 下载 Gemma 2（Google 出品）
ollama pull gemma2

# 下载 Llama 2（较旧，兼容性好）
ollama pull llama2
```

**模型大小对比**:

| 模型        | 大小 | 特点         | 适合场景           |
| :---------- | :--- | :----------- | :----------------- |
| llama3.2:1b | ~1GB | 超小，速度快 | 简单对话、资源受限 |
| qwen2.5:7b  | ~4GB | 中文优秀     | 中文应用           |
| llama3.2:3b | ~2GB | 平衡         | 通用对话           |
| mistral:7b  | ~4GB | 英文强       | 英文应用           |
| llama2:13b  | ~7GB | 质量高       | 复杂任务           |

##### 4. 测试模型

```bash
# 命令行测试
ollama run llama3.2

# 输入问题测试
>>> 你好
>>> exit  # 退出
```

##### 5. 查看已安装的模型

```bash
# 列出所有已下载的模型
ollama list

# 删除不需要的模型
ollama rm llama2
```

##### 6. 模型管理

```bash
# 查看模型信息
ollama show llama3.2

# 复制模型（创建自定义版本）
ollama cp llama3.2 my-llama

# 更新模型
ollama pull llama3.2
```

#### 💡 性能优化建议

1. **显卡加速**（可选）
   - 有 NVIDIA 显卡：自动使用 GPU 加速（需要 CUDA）
   - 有 AMD 显卡：支持 ROCm（Linux）
   - 纯 CPU 运行：也可以使用，速度较慢
2. **内存要求**
   - 7B 模型：至少 8GB RAM
   - 13B 模型：至少 16GB RAM
   - 使用量化模型可降低内存需求
3. **选择合适的模型**
   - 开发测试：使用小模型（1B-3B）
   - 生产环境：根据需求选择（7B-13B）
   - 中文应用：优先 Qwen 系列

#### 🌐 Ollama 优势

- ✅ **完全免费** - 无 API 调用费用
- ✅ **数据隐私** - 所有数据在本地处理
- ✅ **离线使用** - 不依赖网络
- ✅ **快速响应** - 本地运行，延迟低
- ✅ **易于使用** - 一键安装和部署

#### 代码示例

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"log"

	"github.com/cloudwego/eino-ext/components/model/ollama"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	// Ollama 默认运行在 http://localhost:11434
	chatModel, err := ollama.NewChatModel(ctx, &ollama.ChatModelConfig{
		BaseURL: "http://localhost:11434",
		Model:   "llama2", // 或其他已下载的模型
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	messages := []*schema.Message{
		schema.UserMessage("golang是什么?"),
	}

	stream, err := chatModel.Stream(ctx, messages)
	if err != nil {
		log.Fatalf("流式生成失败: %v", err)
	}
	defer stream.Close()

	fmt.Print("AI 回复: ")

	for {
		chunk, err := stream.Recv()
		if err != nil {
			if errors.Is(err, io.EOF) {
				break
			}
			log.Fatalf("接收失败: %v", err)
		}

		// 实时打印
		fmt.Print(chunk.Content)
	}
}
```

## 错误处理最佳实践

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"os"
	"time"

	"github.com/cloudwego/eino-ext/components/model/deepseek"
	"github.com/cloudwego/eino/schema"
)

func generateWithRetry(ctx context.Context, chatModel *deepseek.ChatModel, messages []*schema.Message, maxRetries int) (*schema.Message, error) {
	var lastErr error
	
	for i := 0; i < maxRetries; i++ {
		response, err := chatModel.Generate(ctx, messages)
		if err == nil {
			return response, nil
		}
		
		lastErr = err
		log.Printf("尝试 %d/%d 失败: %v", i+1, maxRetries, err)
		
		// 指数退避
		if i < maxRetries-1 {
			backoff := time.Duration(1<<uint(i)) * time.Second
			log.Printf("等待 %v 后重试...", backoff)
			time.Sleep(backoff)
		}
	}
	
	return nil, fmt.Errorf("重试 %d 次后仍然失败: %w", maxRetries, lastErr)
}

func main() {
	ctx := context.Background()

	chatModel, err := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
		APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
		Model:   "deepseek-chat",
		BaseURL: "https://api.deepseek.com",
		// 设置超时
		Timeout: 30 * time.Second,
	})
	if err != nil {
		log.Fatalf("创建失败: %v", err)
	}

	messages := []*schema.Message{
		schema.UserMessage("你好"),
	}

	// 带重试的生成
	response, err := generateWithRetry(ctx, chatModel, messages, 3)
	if err != nil {
		if errors.Is(err, context.DeadlineExceeded) {
			log.Fatalf("请求超时")
		}
		log.Fatalf("生成失败: %v", err)
	}

	fmt.Printf("成功! 回答: %s\\n", response.Content)
}
```

## 实战示例：智能翻译助手

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	"github.com/cloudwego/eino-ext/components/model/deepseek"
	"github.com/cloudwego/eino/schema"
)

type Translator struct {
	chatModel *openai.ChatModel
}

func NewTranslator(apiKey string) (*Translator, error) {
	ctx := context.Background()
	chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey:      apiKey,
		Model:       "gpt-3.5-turbo",
		Temperature: func() *float32 { t := float32(0.3); return &t }(), // 低温度保证准确性
	})
	if err != nil {
		return nil, err
	}
	return &Translator{chatModel: chatModel}, nil
}

func (t *Translator) Translate(ctx context.Context, text, targetLang string) (string, error) {
	messages := []*schema.Message{
		schema.SystemMessage(fmt.Sprintf("你是一个专业的翻译助手。请将用户输入的文本翻译成%s，只返回翻译结果，不要添加任何解释。", targetLang)),
		schema.UserMessage(text),
	}

	response, err := t.chatModel.Generate(ctx, messages)
	if err != nil {
		return "", err
	}

	return response.Content, nil
}

func main() {
	translator, err := NewTranslator(os.Getenv("OPENAI_API_KEY"))
	if err != nil {
		log.Fatalf("创建翻译器失败: %v", err)
	}

	// 测试翻译
	texts := []struct {
		content string
		target  string
	}{
		{"Hello, how are you?", "中文"},
		{"Eino 是一个强大的 AI 开发框架", "English"},
		{"Les roses sont rouges", "中文"},
	}

	for _, item := range texts {
		result, err := translator.Translate(context.Background(), item.content, item.target)
		if err != nil {
			log.Printf("翻译失败: %v", err)
			continue
		}
		fmt.Printf("原文: %s\\n翻译: %s\\n\\n", item.content, result)
	}
}
```

## 总结

在本章中，我们学习了：

✅ ChatModel 的基本概念和创建方法
✅ Generate() 和 Stream() 两种调用方式
✅ 多轮对话的实现
✅ Temperature、TopP 等参数的使用
✅ 不同模型服务的接入（OpenAI、ARK、Ollama）
✅ 错误处理和重试机制
✅ 实战案例：智能翻译助手



