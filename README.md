# RikkaHub API 打通指南

> Operit ↔ RikkaHub：通过 HTTP API 向 Rikka AI 发消息

---

## 👤 人要做的（一次性初始化）

- [ ] 安装 RikkaHub APP
- [ ] 打开 RikkaHub，确认 Web 服务启动（显示 `Server running on :8080`）
- [ ] 锁定 RikkaHub 后台（最近任务 → 长按卡片 → 锁定）
- [ ] 关闭 RikkaHub 电池优化（设置 → 应用 → RikkaHub → 电池 → 不优化）
- [ ] 锁定 Operit 后台
- [ ] 关闭 Operit 电池优化

> ⚠️ 两个 APP 任何一个被系统杀后台，API 就断。这是整个方案的前提。

---

## 🤖 机要做的（Operit 全自动）

人做完以上 6 步后，以下全部由 Operit 独立完成。

### API 端点

| 操作 | 方法 | 路径 |
|------|------|------|
| 会话列表 | GET | `/api/conversations` |
| 会话详情 | GET | `/api/conversations/{id}` |
| 创建会话 | POST | `/api/conversations` |
| 发送消息 | POST | `/api/conversations/{id}/messages` |

Base URL: `http://localhost:8080`

### 发送消息 DTO

```json
{
  "parts": [
    {"type": "text", "text": "你的消息"}
  ]
}
```

- `parts` 必填，数组，至少一个元素
- 每个元素必须有 `type`（目前仅 `"text"`）和 `text`
- 默认返回 SSE 流式响应，加 `?stream=false` 或 `Accept: application/json` 获取一次性 JSON

### 快速测试

```bash
# 检查服务是否在线
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/conversations
# 期望: 200

# 发消息
curl -s -N -X POST http://localhost:8080/api/conversations/{会话ID}/messages \
  -H "Content-Type: application/json" \
  -d '{"parts":[{"type":"text","text":"你好"}]}'
```

---

## 📋 踩坑与注意事项

1. **别猜 API 端点**。RikkaHub 路由是 `/api/conversations/{id}/messages`，不是常见的 `/api/chat` 或 `/api/message`。
2. **parts 是最小单元**。哪怕只发一句话也必须 `[{"type":"text","text":"..."}]`，不能直接传字符串。
3. **默认 SSE 流式**。不指定 Accept 头时返回 `text/event-stream`，程序解析需处理 SSE。
4. **localhost 无需认证**。RikkaHub 定位本地服务，无 API Key / Token 机制。
5. **两个 APP 都不能杀后台** — 这是最容易被忽略但最致命的点。

---

*Turing · 2026-07*
