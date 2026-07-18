# RikkaHub API 打通指南

> Operit 通过 HTTP API 直接操控 RikkaHub — 发送消息、管理会话、获取对话列表

---

## 目录

- [📖 第一部分：给人看的（操作指南）](#第一部分给人看的操作指南)
- [🤖 第二部分：给机器看的（API 规范）](#第二部分给机器看的api-规范)
- [📋 第三部分：踩坑全记录](#第三部分踩坑全记录)

---

# 第一部分：给人看的（操作指南）

## 1. 这是什么？

RikkaHub 是一个运行在 Android 设备上的 Web 服务器 APP（包名：`com.example.rikkahub`），监听 `localhost:8080`。它提供了一套 HTTP API，允许外部程序向 Rikka AI 对话发送消息、获取会话列表、读取历史消息。

Operit 作为 AI 助手，可以通过这套 API **完全自动化地**与 RikkaHub 交互——前提是满足下面的环境条件。

## 2. 前置条件（⚠️ 必须全部满足）

| # | 条件 | 为什么 | 谁来做 |
|---|------|--------|--------|
| 1 | **RikkaHub APP 不能杀后台** | API 服务跑在 RikkaHub 进程里，杀后台 = 服务停止 | 👤 人 |
| 2 | **Operit APP 不能杀后台** | Operit 需要保持运行才能执行自动化操作 | 👤 人 |
| 3 | 两个 APP 的**电池优化设为「不优化」** | 防止 Android 系统自动杀后台进程 | 👤 人 |
| 4 | 设备连接网络（本地 localhost 不需要外网） | localhost 通信不经过外部网络 | ✅ 自动满足 |
| 5 | RikkaHub 已启动并完成初始化 | 首次启动可能需要加载模型 | 👤 人 |

### 如何锁定后台 & 关闭电池优化？

**锁定后台：**
1. 打开 RikkaHub → 按最近任务键 → 长按 RikkaHub 卡片 → 点「锁定」
2. 对 Operit 做同样操作

**关闭电池优化：**
1. 系统设置 → 应用 → RikkaHub → 电池 → 不优化
2. 系统设置 → 应用 → Operit → 电池 → 不优化

> ⚠️ **这是整个方案的前提。** 任一个 APP 被系统杀掉，API 调用就会失败。

## 3. 人工操作步骤（只做一次）

以下是初始化阶段**需要人手动完成**的操作：

- [ ] **①** 在手机上安装 RikkaHub APP
- [ ] **②** 打开 RikkaHub，确认 Web 服务已启动（通常会显示 `Server running on :8080`）
- [ ] **③** 锁定 RikkaHub 后台（最近任务 → 长按卡片 → 锁定）
- [ ] **④** 关闭 RikkaHub 的电池优化
- [ ] **⑤** 锁定 Operit 后台
- [ ] **⑥** 关闭 Operit 的电池优化

> ✅ 以上 6 步完成后，接下来的所有操作都可以由 **Operit 完全自动化执行**，无需人工干预。

## 4. Operit 能自动做什么？

初始化完成后，Operit 可以独立完成以下所有操作：

| 操作 | API | Operit 自动化 |
|------|-----|---------------|
| 获取会话列表 | `GET /api/conversations` | ✅ 全自动 |
| 获取某个会话的详细消息 | `GET /api/conversations/{id}` | ✅ 全自动 |
| 创建新会话 | `POST /api/conversations` | ✅ 全自动 |
| 发送消息到会话 | `POST /api/conversations/{id}/messages` | ✅ 全自动 |
| 流式接收 AI 回复 | SSE（Server-Sent Events） | ✅ 全自动 |

## 5. 人的操作清单（Checklist）

**一次性初始化：**
- [ ] 安装 RikkaHub
- [ ] 启动 RikkaHub 并确认服务运行
- [ ] 锁定 RikkaHub 后台
- [ ] 关闭 RikkaHub 电池优化
- [ ] 锁定 Operit 后台
- [ ] 关闭 Operit 电池优化

**日常使用：**
- [ ] 无需任何操作——Operit 自动完成一切

---

# 第二部分：给机器看的（API 规范）

## 1. 基本信息

- **Base URL**: `http://localhost:8080`
- **协议**: HTTP/1.1
- **数据格式**: JSON
- **流式响应**: SSE (Server-Sent Events, `text/event-stream`)
- **字符编码**: UTF-8

## 2. OpenAPI 3.1 规范（YAML）

```yaml
openapi: "3.1.0"
info:
  title: RikkaHub Web API
  description: |
    RikkaHub 本地 Web 服务器 API。
    用于外部程序（如 Operit）向 Rikka AI 发送消息、管理会话。
    所有端点均无需认证（本地 localhost 访问）。
  version: "1.0.0"
  contact:
    name: RikkaHub

servers:
  - url: http://localhost:8080
    description: RikkaHub 本地服务器

paths:
  /api/conversations:
    get:
      operationId: listConversations
      summary: 获取所有对话列表
      description: 返回 RikkaHub 中所有会话的列表，按更新时间倒序排列。
      responses:
        "200":
          description: 会话列表
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/Conversation"
    post:
      operationId: createConversation
      summary: 创建新会话
      description: 创建一个新的空白会话。
      requestBody:
        required: false
        content:
          application/json:
            schema:
              type: object
              properties:
                title:
                  type: string
                  description: 可选，会话标题
      responses:
        "201":
          description: 创建成功
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Conversation"

  /api/conversations/{conversationId}:
    get:
      operationId: getConversation
      summary: 获取单个会话详情
      description: 返回指定会话的完整信息，包含消息列表。
      parameters:
        - name: conversationId
          in: path
          required: true
          schema:
            type: string
      responses:
        "200":
          description: 会话详情
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ConversationDetail"
        "404":
          description: 会话不存在

  /api/conversations/{conversationId}/messages:
    post:
      operationId: sendMessage
      summary: 发送消息
      description: |
        向指定会话发送一条消息，Rikka AI 会生成回复。
        默认返回 SSE 流式响应（`text/event-stream`）。
        可通过 `Accept` 头或 `stream` 参数控制响应格式。
      parameters:
        - name: conversationId
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/SendMessageRequest"
      responses:
        "200":
          description: |
            消息发送成功。响应格式取决于请求：
            - 默认：SSE 流式（`text/event-stream`）
            - `?stream=false` 或 `Accept: application/json`：一次性 JSON 响应
          content:
            text/event-stream:
              schema:
                type: string
                description: SSE 事件流，每个事件包含一个 JSON 对象
            application/json:
              schema:
                $ref: "#/components/schemas/MessageResponse"
        "400":
          description: 请求体格式错误
        "404":
          description: 会话不存在

components:
  schemas:
    Conversation:
      type: object
      properties:
        id:
          type: string
          description: 会话唯一标识
        title:
          type: string
          description: 会话标题
        createdAt:
          type: string
          format: date-time
          description: 创建时间
        updatedAt:
          type: string
          format: date-time
          description: 最后更新时间
        messageCount:
          type: integer
          description: 消息数量

    ConversationDetail:
      allOf:
        - $ref: "#/components/schemas/Conversation"
        - type: object
          properties:
            messages:
              type: array
              items:
                $ref: "#/components/schemas/Message"

    Message:
      type: object
      properties:
        id:
          type: string
        role:
          type: string
          enum: [user, assistant, system]
        parts:
          type: array
          items:
            $ref: "#/components/schemas/MessagePart"
        createdAt:
          type: string
          format: date-time

    MessagePart:
      type: object
      properties:
        type:
          type: string
          enum: [text, image, file]
        text:
          type: string
          description: 当 type=text 时的文本内容

    SendMessageRequest:
      type: object
      required: [parts]
      properties:
        parts:
          type: array
          minItems: 1
          description: 消息内容数组
          items:
            type: object
            required: [type, text]
            properties:
              type:
                type: string
                enum: [text]
              text:
                type: string
                description: 消息文本
      example:
        parts:
          - type: text
            text: "你好，Rikka！"

    MessageResponse:
      type: object
      properties:
        id:
          type: string
        conversationId:
          type: string
        message:
          $ref: "#/components/schemas/Message"

    Error:
      type: object
      properties:
        error:
          type: string
          description: 错误描述
```

## 3. JSON Schema：SendMessageRequest

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://rikkahub.local/schemas/send-message-request.json",
  "title": "SendMessageRequest",
  "description": "向 RikkaHub 会话发送消息的请求体结构",
  "type": "object",
  "required": ["parts"],
  "properties": {
    "parts": {
      "type": "array",
      "minItems": 1,
      "description": "消息内容数组，每个元素是一个消息片段",
      "items": {
        "type": "object",
        "required": ["type", "text"],
        "properties": {
          "type": {
            "type": "string",
            "enum": ["text"],
            "description": "消息片段类型，目前仅支持 text"
          },
          "text": {
            "type": "string",
            "description": "消息文本内容"
          }
        },
        "additionalProperties": false
      }
    }
  },
  "additionalProperties": false,
  "examples": [
    {
      "parts": [
        {"type": "text", "text": "你好，请帮我翻译这段话"}
      ]
    },
    {
      "parts": [
        {"type": "text", "text": "总结以下内容："},
        {"type": "text", "text": "第一段..."},
        {"type": "text", "text": "第二段..."}
      ]
    }
  ]
}
```

### 无效请求示例（❌ 会被拒绝）

```json
// ❌ 缺少 parts 字段
{}

// ❌ parts 不是数组
{"parts": "hello"}

// ❌ parts 为空数组
{"parts": []}

// ❌ 缺少 type 字段
{"parts": [{"text": "hello"}]}

// ❌ 缺少 text 字段
{"parts": [{"type": "text"}]}

// ❌ type 值不合法
{"parts": [{"type": "image", "text": "hello"}]}
```

## 4. 一键 Shell 脚本

保存为 `rikkahub.sh`，用法：

```bash
#!/bin/bash
# rikkahub.sh — RikkaHub API 一键操作脚本
# 用法:
#   ./rikkahub.sh send <conversationId> "消息内容"
#   ./rikkahub.sh list_convs
#   ./rikkahub.sh get_conv <conversationId>
#   ./rikkahub.sh create_conv [标题]

BASE="http://localhost:8080"

send_message() {
  local conv_id="$1"
  local text="$2"
  curl -s -N -X POST "${BASE}/api/conversations/${conv_id}/messages" \
    -H "Content-Type: application/json" \
    -d "{\"parts\":[{\"type\":\"text\",\"text\":\"${text}\"}]}"
}

list_conversations() {
  curl -s "${BASE}/api/conversations" | python3 -m json.tool 2>/dev/null || curl -s "${BASE}/api/conversations"
}

get_conversation() {
  local conv_id="$1"
  curl -s "${BASE}/api/conversations/${conv_id}" | python3 -m json.tool 2>/dev/null || curl -s "${BASE}/api/conversations/${conv_id}"
}

create_conversation() {
  local title="${1:-}"
  if [ -n "$title" ]; then
    curl -s -X POST "${BASE}/api/conversations" \
      -H "Content-Type: application/json" \
      -d "{\"title\":\"${title}\"}"
  else
    curl -s -X POST "${BASE}/api/conversations" \
      -H "Content-Type: application/json"
  fi
}

case "${1:-}" in
  send)
    if [ -z "$2" ] || [ -z "$3" ]; then
      echo "用法: $0 send <conversationId> \"消息内容\""
      exit 1
    fi
    send_message "$2" "$3"
    ;;
  list_convs)
    list_conversations
    ;;
  get_conv)
    if [ -z "$2" ]; then
      echo "用法: $0 get_conv <conversationId>"
      exit 1
    fi
    get_conversation "$2"
    ;;
  create_conv)
    create_conversation "$2"
    ;;
  *)
    echo "RikkaHub API 工具"
    echo ""
    echo "命令:"
    echo "  send <id> \"msg\"  — 发送消息"
    echo "  list_convs        — 列出所有会话"
    echo "  get_conv <id>     — 获取会话详情"
    echo "  create_conv [标题] — 创建新会话"
    ;;
esac
```

## 5. 快速测试命令

```bash
# 1. 检查 RikkaHub 是否在线
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/conversations
# 期望输出: 200

# 2. 列出所有会话
curl -s http://localhost:8080/api/conversations | python3 -m json.tool

# 3. 发送消息（替换 YOUR_CONV_ID）
curl -s -N -X POST http://localhost:8080/api/conversations/YOUR_CONV_ID/messages \
  -H "Content-Type: application/json" \
  -d '{"parts":[{"type":"text","text":"你好"}]}'

# 4. 非流式获取完整回复
curl -s -X POST 'http://localhost:8080/api/conversations/YOUR_CONV_ID/messages?stream=false' \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"parts":[{"type":"text","text":"你好"}]}'
```

---

# 第三部分：踩坑全记录

## 探索时间线

### 尝试 1：直接 HTTP POST（失败）
```bash
curl -X POST http://localhost:8080/api/message -d '{"message":"hello"}'
# → 404 Not Found
```
**原因**：端点路径错误。

### 尝试 2：猜测端点（失败）
```bash
curl http://localhost:8080/api/chat -d '{"prompt":"hello"}'
# → 404
```
**原因**：API 结构不是常见的 `/api/chat` 模式。

### 尝试 3：读取 RikkaHub 源码（关键突破）

通过分析 RikkaHub APK 内的源码（Kotlin/ktor），定位到真正的路由定义：

```kotlin
// 路由注册
route("/api") {
    route("/conversations") {
        get { ... }           // 列表
        post { ... }          // 创建
        route("/{id}") {
            get { ... }       // 详情
            route("/messages") {
                post { ... }  // 发送消息 ← 关键！
            }
        }
    }
}
```

### 尝试 4：破解消息 DTO（关键突破）

发送消息的请求体结构在源码中定义为：

```kotlin
@Serializable
data class SendMessageRequest(
    val parts: List<MessagePart>
)

@Serializable
data class MessagePart(
    val type: String,  // "text"
    val text: String
)
```

### 尝试 5：首次成功

```bash
curl -s -X POST http://localhost:8080/api/conversations/CONV_ID/messages \
  -H "Content-Type: application/json" \
  -d '{"parts":[{"type":"text","text":"你好"}]}'
# → 200 OK + SSE 流式回复 ✅
```

## 关键教训

| # | 教训 | 说明 |
|---|------|------|
| 1 | **不要猜测 API** | 盲目猜测端点浪费时间，直接读源码最高效 |
| 2 | **Kotlin `@Serializable` 直接映射 JSON** | `data class` 字段名 = JSON key，无需额外配置 |
| 3 | **parts 数组是最小单元** | 即使只有一条文本也必须包在 `[{"type":"text","text":"..."}]` 里 |
| 4 | **默认 SSE 流式** | 不指定 `Accept` 或 `stream=false` 时返回 SSE 事件流 |
| 5 | **localhost 不需要认证** | RikkaHub 设计为本地服务，无 token/API key 机制 |

---

## 附录：环境要求摘要

```
┌──────────────────────────────────────────┐
│           RikkaHub ←→ Operit             │
│                                          │
│  ✓ 两个 APP 均不杀后台                    │
│  ✓ 两个 APP 均关闭电池优化                 │
│  ✓ 同一设备 localhost:8080               │
│  ✓ RikkaHub 已启动 Web 服务              │
│  ✓ 无需外网连接                           │
│  ✓ 无需 API Key / Token                  │
└──────────────────────────────────────────┘
```

---

*探索过程由 Operit AI 驱动 · 文档版本 2.0 · 2026-07*
