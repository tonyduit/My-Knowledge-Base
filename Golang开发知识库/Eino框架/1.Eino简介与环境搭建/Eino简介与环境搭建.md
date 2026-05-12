# Eino 简介和环境搭建

> 参考文档: https://www.cloudwego.io/zh/docs/eino/

## 什么是 Eino？

Eino（发音：美 / 'aino /，近似音: i know）是由字节跳动开源的基于 Golang 的 AI 应用开发框架。Eino 旨在为 Go 开发者提供类似 LangChain、LangGraph 的能力，但更符合 Golang 的编程习惯。

### 核心特性

- **🧩 组件化设计**：提供 ChatModel、Embedding、Retriever、Tool 等丰富的原子组件
- **🔗 灵活编排**：支持 Chain（链式）和 Graph（图式）两种编排方式
- **🔌 生态集成**：支持 OpenAI、ARK、Ollama、Qwen 等多种大模型
- **🎯 高性能**：基于 Go 语言，天然支持高并发场景
- **🛠️ 易扩展**：清晰的接口设计，方便自定义组件

### Eino 架构

```
┌─────────────────────────────────────────┐
│          AI Application Layer           │
│   (React Agent, RAG, Multi-Agent)       │
├─────────────────────────────────────────┤
│        Orchestration Layer              │
│      (Chain, Graph, Workflow)           │
├─────────────────────────────────────────┤
│         Component Layer                 │
│  (ChatModel, Tool, Retriever, etc.)     │
├─────────────────────────────────────────┤
│         Integration Layer               │
│  (OpenAI, ARK, Milvus, Redis, etc.)     │
└─────────────────────────────────────────┘
```

## 环境搭建

### 前置要求

- Go 1.18 或更高版本
- Git
- 一个大模型 API Key (推荐使用 OpenAI 或字节跳动 ARK)

### 安装 Eino

创建一个新的 Go 项目并安装 Eino：

```bash
# 创建项目目录
mkdir eino-tutorial
cd eino-tutorial

# 初始化 Go 模块
go mod init eino-tutorial

# 安装 Eino 核心库
go get github.com/cloudwego/eino

# 安装 Eino 扩展组件库
go get github.com/cloudwego/eino-ext
```

### 环境变量配置

根据你使用的大模型服务，配置相应的环境变量：

#### DeepSeek

```bash
# Windows PowerShell
$env:DEEPSEEK_API_KEY="your-api-key"

# Linux/MacOS
export DEEPSEEK_API_KEY="your-api-key"
```

#### OpenAI（可选）

```bash
# Windows PowerShell
$env:OPENAI_API_KEY="your-api-key"

# Linux/MacOS
export OPENAI_API_KEY="your-api-key"
```

#### 字节跳动 ARK (豆包大模型)

```bash
# Windows PowerShell
$env:ARK_API_KEY="your-api-key"
$env:ARK_MODEL_NAME="your-model-endpoint"  # 例如: ep-xxxxx

# Linux/MacOS
export ARK_API_KEY="your-api-key"
export ARK_MODEL_NAME="your-model-endpoint"
```

豆包大模型申请地址: https://console.volcengine.com/ark/region:ark+cn-beijing/model

## Hello Eino - 第一个程序

创建 `hello.go` 文件：

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
	// 1. 创建上下文
	ctx := context.Background()

	// 2. 创建 ChatModel (使用 DeepSeek)
	chatModel, err := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
		APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
		Model:   "deepseek-chat",
		BaseURL: "https://api.deepseek.com",
	})
	if err != nil {
		log.Fatalf("创建 ChatModel 失败: %v", err)
	}

	// 3. 准备消息
	messages := []*schema.Message{
		schema.SystemMessage("你是一个友好的 AI 助手"),
		schema.UserMessage("你好，请介绍一下 Eino 框架"),
	}

	// 4. 调用模型生成响应
	response, err := chatModel.Generate(ctx, messages)
	if err != nil {
		log.Fatalf("生成响应失败: %v", err)
	}

	// 5. 输出结果
	fmt.Printf("AI 响应: %s\\n", response.Content)
	
	// 6. 输出 token 使用情况
	if response.ResponseMeta != nil && response.ResponseMeta.Usage != nil {
		fmt.Printf("\\nToken 使用统计:\\n")
		fmt.Printf("  输入 Token: %d\\n", response.ResponseMeta.Usage.PromptTokens)
		fmt.Printf("  输出 Token: %d\\n", response.ResponseMeta.Usage.CompletionTokens)
		fmt.Printf("  总计 Token: %d\\n", response.ResponseMeta.Usage.TotalTokens)
	}
}
```

### 运行程序

```bash
# 设置环境变量（如果还没设置）
# Windows PowerShell:
$env:DEEPSEEK_API_KEY="your-api-key"

# Linux/MacOS:
# export DEEPSEEK_API_KEY="your-api-key"

# 运行程序
go run hello.go
```

### 预期输出

```
AI 响应: Eino 是由字节跳动开源的基于 Golang 的 AI 应用开发框架。它提供了丰富的组件和灵活的编排能力，帮助开发者快速构建高性能的 AI 应用...

Token 使用统计:
  输入 Token: 45
  输出 Token: 120
  总计 Token: 165
```

## 使用 ARK (豆包) 的示例

### ARK API Key 申请方法

ARK 是字节跳动推出的豆包大模型 API 服务，申请步骤如下：

#### 1. 访问火山引擎 ARK 控制台

```
https://console.volcengine.com/ark
```

或直接访问模型广场：

```
https://console.volcengine.com/ark/region:ark+cn-beijing/model
```

#### 2. 注册并登录

- 使用手机号注册火山引擎账号
- 完成实名认证（个人或企业）

#### 3. 创建推理接入点

1. 进入控制台后，点击左侧 **"模型广场"**

2. 选择想要使用的模型，推荐：

   - **doubao-pro-4k** - 通用对话，支持 Function Call
   - **doubao-pro-32k** - 支持 32K 长文本
   - **doubao-embedding-large** - 文本向量化

3. 点击 **"推理"** 按钮

4. 选择 **"按量计费"**（新用户通常有免费额度）

5. 创建成功后获得

    

   Endpoint ID

   （格式：

   ```
   ep-xxxxxx
   ```

   ）

   - 这个 Endpoint ID 就是你的模型名称

#### 4. 获取 API Key

1. 在控制台顶部找到 **"API Key 管理"**
2. 点击 **"创建 API Key"**
3. 给 API Key 命名（如：eino-dev）
4. **立即复制保存** API Key（⚠️ 只显示一次）

#### 5. 配置环境变量

```bash
# Windows PowerShell
$env:ARK_API_KEY="your-ark-api-key"
$env:ARK_MODEL_NAME="ep-20241104xxxxxx"

# Linux/MacOS
export ARK_API_KEY="your-ark-api-key"
export ARK_MODEL_NAME="ep-20241104xxxxxx"
```

#### 💰 费用说明

- 新用户通常有免费额度
- doubao-pro-4k 价格约 ¥0.008/千 tokens（比 OpenAI 便宜约 10 倍）
- 可在控制台设置每日费用上限

#### 相关链接

- ARK 控制台: https://console.volcengine.com/ark
- API 文档: https://www.volcengine.com/docs/82379
- 价格说明: https://www.volcengine.com/pricing/ark

### 代码示例

如果你想使用字节跳动的 ARK 服务，只需要修改模型创建部分：

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

	// 使用 ARK ChatModel
	chatModel, err := ark.NewChatModel(ctx, &ark.ChatModelConfig{
		APIKey: os.Getenv("ARK_API_KEY"),
		Model:  os.Getenv("ARK_MODEL_NAME"), // 例如: ep-xxxxx
	})
	if err != nil {
		log.Fatalf("创建 ChatModel 失败: %v", err)
	}

	messages := []*schema.Message{
		schema.SystemMessage("你是一个友好的 AI 助手"),
		schema.UserMessage("你好，请用一句话介绍北京"),
	}

	response, err := chatModel.Generate(ctx, messages)
	if err != nil {
		log.Fatalf("生成响应失败: %v", err)
	}

	fmt.Printf("AI 响应: %s\\n", response.Content)
}
```

## 项目结构建议

对于实际项目，建议使用以下目录结构：

```
eino-project/
├── cmd/
│   └── main.go           # 主程序入口
├── internal/
│   ├── agent/            # Agent 实现
│   ├── tools/            # 自定义工具
│   └── config/           # 配置管理
├── pkg/
│   └── utils/            # 公共工具
├── go.mod
├── go.sum
└── README.md
```

## 常见问题

### 1. 网络连接问题

如果遇到网络问题，可以设置代理：

```bash
# Windows PowerShell
$env:HTTP_PROXY="http://127.0.0.1:7890"
$env:HTTPS_PROXY="http://127.0.0.1:7890"

# Linux/MacOS
export HTTP_PROXY="http://127.0.0.1:7890"
export HTTPS_PROXY="http://127.0.0.1:7890"
```

### 2. 模块下载慢

使用 Go 代理：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

### 3. API Key 错误

确保：

- API Key 格式正确
- 账户有足够的余额或配额
- 模型名称正确