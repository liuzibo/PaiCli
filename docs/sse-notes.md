# SSE（Server-Sent Events）学习笔记

## 1. SSE 是什么

SSE 是基于 **普通 HTTP 长连接** 的服务器单向推送协议：客户端发一次请求，服务器不结束响应，持续写出数据。

- 方向：**单向**（服务器 → 客户端），客户端只能收
- 传输：纯 HTTP，走 80/443 端口，天然过代理/网关
- 浏览器端有原生 `EventSource` API，**自带断线重连**
- 典型场景：LLM 流式输出（OpenAI/DeepSeek 都用它）、消息通知、进度推送、股票行情

## 2. 报文格式（核心，必背）

SSE 的响应体是纯文本，由一条条"消息"组成，**空行分隔**。每条消息由若干"字段行"构成：

```
data: hello          <- 数据（最常用）

event: chat          <- 自定义事件类型（可选）
data: {"msg":"hi"}

id: 1001             <- 消息编号，用于断点续传（可选）
data: 第 1001 条

: heartbeat          <- 冒号开头是注释，常用作心跳保活

retry: 3000          <- 建议客户端重连间隔毫秒数（可选）
```

规则速记：

| 字段 | 作用 | 缺省 |
|------|------|------|
| `data:` | 数据内容，多条 `data:` 会用 `\n` 拼接成一条消息 | 无 |
| `event:` | 事件类型，对应浏览器 `addEventListener` 的事件名 | `message` |
| `id:` | 消息 ID，重连时浏览器自动带上 `Last-Event-ID` 请求头 | 无 |
| `retry:` | 重连等待毫秒数 | 浏览器默认（约 3s） |
| `:` | 注释行，忽略内容 | - |

**一条消息 = 若干字段行 + 一个空行**。没有空行，数据就不会被"派发"。

## 3. HTTP 层面

服务端响应必须包含：

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

要点：

- 响应**不设置 Content-Length**，chunked 传输，写一段 flush 一段
- 断线后浏览器自动重连，并把最后收到的 `id:` 通过 `Last-Event-ID` 请求头带回，服务端据此续传
- 代理/负载均衡可能缓冲响应导致"收不到流"，注释行心跳可探活

## 4. 浏览器端：EventSource

```js
// 1. 基本用法（GET 请求）
const es = new EventSource("/api/stream");

es.onmessage = (e) => {          // 对应没写 event: 的消息
  console.log(e.data);
};

// 2. 自定义事件：对应 event: chat
es.addEventListener("chat", (e) => {
  console.log(e.data, e.lastEventId);
});

// 3. 错误与关闭
es.onerror = () => { /* 自动重连中；es.readyState === 0 时才彻底关闭 */ };
es.close();                      // 主动关闭，不再重连
```

局限（面试常问）：

- 只支持 **GET**、不能自定义请求体 → 需要 POST/带 body 时用 `fetch` + `ReadableStream` 手动解析（各家 LLM 前端都这么做）
- IE/旧浏览器需要 polyfill
- 同源限制，跨域要服务端开 CORS

## 5. Go 客户端实现（对照本项目 `internal/llm/client.go:83`）

SSE 客户端本质就是：**发 HTTP 请求 → 逐行读 body → 解析 `data:` 行**。

```go
resp, _ := http.Post(url, "application/json", body) // stream: true
scanner := bufio.NewScanner(resp.Body)
for scanner.Scan() {
    line := strings.TrimSpace(scanner.Text())
    // 跳过：空行、":"注释行、非 data: 行
    if line == "" || strings.HasPrefix(line, ":") || !strings.HasPrefix(line, "data:") {
        continue
    }
    data := strings.TrimSpace(strings.TrimPrefix(line, "data:"))
    if data == "[DONE]" { // OpenAI 系约定的结束标记
        break
    }
    var chunk SomeChunk
    json.Unmarshal([]byte(data), &chunk) // 每个 chunk 是一个 JSON
    // ... 增量处理
}
```

本项目 `ChatStream` 的三个值得细读的点：

1. **`bufio.Scanner` 缓冲区调大**（client.go:104）：`4MB` 上限，防止单行 JSON 超长被截断报错
2. **增量拼装**：content/reasoning 是 `strings.Builder` 逐 delta 累加，边解析边通过 `emitStream` 回调给上层
3. **流式 tool call 分片拼装**（client.go:137-152）：函数名和 arguments 按下标分片到达，用 `builder` 按 index 累积，结束后统一组装

## 6. Go 服务端最小实现

关键：拿到 `http.Flusher`，写一段 flush 一段。

```go
func sseHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    flusher := w.(http.Flusher) // 断言失败说明中间件不支持流式

    for i := 0; i < 5; i++ {
        fmt.Fprintf(w, "id: %d\ndata: {\"n\":%d}\n\n", i, i)
        flusher.Flush() // 立刻推给客户端
        time.Sleep(500 * time.Millisecond)
    }
}
```

注意：Go 1.20+ 的 `net/http` 默认会自动 flush `text/event-stream` 响应，但显式 Flush 仍是好习惯。

## 7. 横向对比

| | 轮询 | SSE | WebSocket |
|---|---|---|---|
| 方向 | 单向（客户端拉） | 单向（服务器推） | 全双工 |
| 协议 | HTTP | HTTP | 独立协议（ws://，Upgrade 握手） |
| 重连 | 自管 | **内置自动重连** | 自管 |
| 数据格式 | 任意 | 文本（`data:` 字段） | 文本/二进制 |
| 基础设施 | 无要求 | 普通 HTTP 即可 | 需代理/网关支持 |
| 适用 | 低频 | LLM 流式、通知、进度 | 聊天室、游戏、协同 |

经验法则：**只需服务器→客户端，优先 SSE**；需要客户端也实时推送（且不是发请求），才上 WebSocket。

## 8. LLM 流式 API 中的 SSE 约定

OpenAI-compatible 流式接口（DeepSeek/Kimi/GLM 同款）：

- 请求体带 `"stream": true`
- 每个 `data:` 行是一个 JSON chunk，`choices[].delta` 里带增量内容
- 结束标记：`data: [DONE]`
- 思考型模型额外有 `delta.reasoning_content`（DeepSeek）字段
- usage 通常在最后一个 chunk（需 `stream_options: {"include_usage": true}`）

## 9. 动手练习

1. **肉眼看原始报文**（最有效）：

   ```bash
   curl -N https://api.deepseek.com/chat/completions \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
     -d '{"model":"deepseek-chat","stream":true,"messages":[{"role":"user","content":"讲个笑话"}]}'
   ```

2. 写一个 50 行的 Go SSE server（第 6 节代码），用 `curl -N localhost:8080/sse` 观察
3. 回读 `internal/llm/client.go` 的 `ChatStream` + `internal/llm/client_test.go:44` 的 `TestChatStreamParsesSSE`，看假 SSE server 怎么构造测试

## 10. 参考资料

- MDN 使用服务器发送事件（中文）：https://developer.mozilla.org/zh-CN/docs/Web/API/Server-sent_events/Using_server-sent_events
- MDN EventSource API：https://developer.mozilla.org/zh-CN/docs/Web/API/EventSource
- HTML 规范 SSE 章节（权威）：https://html.spec.whatwg.org/multipage/server-sent-events.html
- OpenAI 流式接口文档：https://platform.openai.com/docs/api-reference/streaming
