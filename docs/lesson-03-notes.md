# 第 3 课学习笔记：LLM 流式客户端、SSE 解析与 OpenAI 协议

对应代码：`internal/llm/client.go`（391 行）、`internal/llm/types.go`、`internal/agent/agent.go`（streamChat）

## 1. 全景：一个结构体适配所有 provider

```go
type OpenAICompatibleClient struct {
    provider string
    cfg      config.ProviderConfig
    http     *http.Client
}
```

- **没有为 DeepSeek/Kimi/GLM 各写 client**——它们全说"OpenAI 方言"（同样的 `/chat/completions` 端点、同样的消息格式），一个 client + 不同 BaseURL/APIKey/Model 就是全部。适配成本从 N 降到 1
- `NewClient`（client.go:24）是**构造期校验**典范：API key、BaseURL、Model 缺失当场报错，错误信息指向解法（`run paicli doctor`），而不是甩 "invalid config"

## 2. HTTP 流式原理：Do 返回的只是"信封"（核心认知）

一个 HTTP 响应分两截：**头部**和**身体**。头部几毫秒传完，身体时长协议不限。

```go
return c.http.Do(req)   // 返回时机 = 头部解析完毕，不是身体传完！
```

- `resp.Body` 是**惰性流**（io.ReadCloser）——一根还连着服务器的管子，此刻可能一个字节的身体都没到
- 每调一次 Read，才从 TCP socket 捞现在已到达的字节
- 服务器身体长度未知还能一直写：靠 HTTP/1.1 chunked 传输编码

流式时间线：

```
t=0ms      服务器秒回头部 200 OK + Content-Type: text/event-stream
           ↑ 此刻 c.http.Do(req) 返回，ChatStream 开始跑
t=30ms     第 1 个 token → data: {...} → 用户屏幕蹦出第 1 个字
   ...
t=25s      模型说完 → data: [DONE] → Scan() 返回 false，聚合返回
```

对比非流式 `Chat` 的 `io.ReadAll(resp.Body)`：**阻塞到身体传完**，25 秒后一次性拿全部。同一个管子，一个等灌满，一个边流边用。

这个"管子"模型串起三个悬案：

1. `http.Client{Timeout: 180s}` 计的是**整个请求周期**（含身体读取）→ 流式对话的兜底护栏
2. ctx 取消（NewRequestWithContext）直接掐断管子，阻塞中的 Scan() 立刻返回错误
3. `defer resp.Body.Close()` 不能漏——头部回来后管子还开着，不关就 fd + 连接泄漏

## 3. SSE 解析主循环（client.go:103-154）

```go
scanner := bufio.NewScanner(resp.Body)                    // ① 按行切
scanner.Buffer(make([]byte, 0, 64*1024), 4*1024*1024)     // ② 初始 64KB / 单行上限 4MB
for scanner.Scan() {
    line := strings.TrimSpace(scanner.Text())
    if line == "" || strings.HasPrefix(line, ":") || !strings.HasPrefix(line, "data:") {
        continue                                          // ③ 空行/心跳注释/非 data 全跳过
    }
    data := strings.TrimSpace(strings.TrimPrefix(line, "data:"))
    if data == "[DONE]" { break }                         // ④ 流终止哨兵
    var chunk openAIStreamResponse
    json.Unmarshal([]byte(data), &chunk)                  // ⑤ 每条 data 是一个 JSON chunk
}
```

细节：

- **① Scanner 而非 Reader**：SSE 面向行（一行 data: + 一个空行），Scanner 默认按行切，语义精确匹配——"按协议形态选 IO 原语"
- **② scanner.Buffer 是救命的**：Scanner 默认单行上限 **64KB**，超了 ErrTooLong 断流。LLM 单行轻松超（一个 chunk 带 2KB 代码很正常），配到 4MB。**这是 Go 面试 Scanner 第一坑**。注意：Buffer 不是"接收"数据的——数据接收者是 resp.Body，Scanner 是按行切割机，Buffer 只是配缓冲
- **③ 一行过滤三种噪音**：空行（分隔符）、`:` 开头（心跳注释）、非 data 前缀（event: 等）。本项目不需要"攒字段行等空行派发"——每条 data 独立自足（完整 JSON chunk），比浏览器 EventSource 模型简单一档
- **④ `[DONE]` 是 OpenAI 方言的哨兵**，SSE 规范本身没有
- **⑤ 增量三通道**：delta 可同时带 reasoning / content / tool_calls 三种载荷

**思考题 1 答案**：恶意服务发 5MB 单行 → Scanner 攒行触到 4MB 上限 → Scan() 返回 false → 错误在 **client.go:155 的 scanner.Err()** 被捕获（bufio.ErrTooLong）。这就是为什么那行检查不可省略。

## 4. ChatStream 函数解剖（client.go:83-166）

设计本质：**一个函数干两件事——边流边广播 + 收尾聚合成完整响应**。返回完整 ChatResponse，调用方既拿实时增量（给 UI）又拿最终结果（做工具执行判断）。"流式接口返回聚合结果"是 LLM 客户端标准设计。

四幕结构：

- **第一幕（84-92）**：请求 + 前置校验。`defer Close` 紧跟 err 检查；错误分支 ReadAll 出 body 塞进错误信息——4xx/5xx 的 body 带着服务端诊断（"invalid api key"），丢掉用户只看到 401 三个数字
- **第二幕（94-104）**：工作区 = 三个 strings.Builder + toolBuilders map + usage。**ChatStream 本质是流式状态机，这四样是全部状态**。Builder 而非 `+=`：几千个 delta，`+=` 是 O(n²)，Builder 均摊 O(n)
- **第三幕（105-154）**：主循环逐 chunk 分发（见上）
- **第四幕（155-165）**：查 `scanner.Err()`（Scan 返回 false 只说明"读完了"，可能是中断，忘了查就把半截回答当完整结果）+ 聚合返回

两个精妙的不对称/防御：

- **tool call 碎片不 emit**：reasoning/content 实时 emit 给 Observer，tool call 静默累积——切碎的 JSON 参数对用户无意义，拼完整才有用。**"什么该流出去、什么必须攒着"是主动决策**
- **usage 条件覆盖**（client.go:118）`if chunk.Usage.PromptTokens > 0`：**OpenAI 流式协议 usage 只在最后一个 chunk 出现一次**（严格说还要请求带 `stream_options: {"include_usage": true}`），中途 chunk 是零值。直接赋值会把有效值清回 0。**思考题 2 答案**

**ChatStream 全程单 goroutine，无锁无 channel**——解析是 IO 密集的顺序消费，天然串行；并发留给工具执行层。"哪里并发、哪里串行"是主动设计。

## 5. tool call 增量拼接（client.go:137-152, 316-385）——本课精华

流式下一个 tool call 被切成任意碎片到达：

```
chunk 1: {"index":0, "id":"call_abc", "function":{"name":"read_file", "arguments":""}}
chunk 2: {"index":0, "function":{"arguments":"{\"pa"}}
chunk 3: {"index":0, "function":{"arguments":"th\":\"main.go\"}"}}
```

拼接算法：

```go
toolBuilders := map[int]*streamToolCallBuilder{}   // index 做路由键

if call.ID != ""       { builder.id = call.ID }                    // ID = 覆盖（只有第一个 chunk 带）
if call.Function.Name != "" { builder.name += call.Function.Name } // 名字 = 累加（有 provider 把长名切碎）
if call.Function.Arguments != "" { builder.arguments.WriteString(...) } // 参数 = 累加
```

收尾 `assembleStreamToolCalls`（client.go:356）两拍：

- **按 index 排序**（client.go:364）——map 遍历无序！不排序 tool call 顺序每次运行都不同，而顺序是配对契约的一部分
- **兜底造 ID**（client.go:370）`call_%d`——有些 compatible 服务不回 ID，不造就 400

一句话：**用 delta.index 做分片路由键，id 覆盖、name/arguments 累加，收尾按 index 排序 + 兜底造 ID——三处任缺一处，协议配对就破了。**

## 6. 兼容性防御（"OpenAI-compatible"的裂缝）

方言统一了 80%，剩下 20% 靠防御性代码糊：

- **reasoning 双字段**（client.go:126-128）：思维链没有标准名，`reasoning_content`（DeepSeek 风格）和 `reasoning`（其他家）都试
- **`normalizeFunctionArguments`**（client.go:337）：规范里 arguments 是裸 JSON 对象，有 provider **双重编码**成 JSON 字符串（`"{\"path\":...}"`）。检测开头是 `"` 就解一层包
- **name 也累加**：某些 provider 把长工具名切碎

## 7. 资源与安全细节（第 1 课知识回收）

- `io.LimitReader(resp.Body, 8<<20)`（client.go:58,90）：非流式读响应体限 8MB——防异常服务巨包打爆内存
- `http.Client{Timeout: 180 * time.Second}`：整个请求周期的兜底；精确取消靠 ctx
- `defer resp.Body.Close()`：http 响应体是 fd 持有者——资源审计清单第 2 条活例

## 8. OpenAI Chat Completions 协议全景

一次"对话轮"的时序：

```
客户端                                        模型服务
  │ ① POST /chat/completions                   │
  │    Authorization: Bearer sk-xxx            │
  │    {model, messages, tools, stream...}     │
  │ ─────────────────────────────────────────▶ │
  │ ② 头部立刻返回 200 OK                      │
  │ ◀───────────────────────────────────────── │（模型逐 token 生成中）
  │ ③ 身体以 SSE 逐行推送                       │
  │ ◀── data: {"delta":{"content":"你"}} ...    │
  │ ◀── data: [DONE]                           │
  │ ④ 若要调工具：下轮带 assistant(tool_calls)  │
  │    + tool(role, tool_call_id, 结果)         │
```

**模型无状态**：多轮对话是客户端每轮重发全量 messages。Agent 的循环、状态、编排全在客户端——这就是 Agent 框架都跑在客户端的原因。

### 请求体字段（对照 chatRequest）

```json
{
  "model":       "...",        // 用哪个模型
  "messages":    [...],        // 协议的心脏
  "temperature": 0.2,          // 偏确定性（代码场景要稳定）
  "stream":      true,
  "tools":        [...],       // JSON Schema 描述的工具清单
  "tool_choice":  "auto"       // 模型自己决定调不调
}
```

tools 每项是三层包装 `{"type":"function","function":{name, description, parameters}}`。**模型靠 description + schema "学会"用工具**——没有编译期校验，全靠提示工程，description 质量直接决定调用准确率。

### messages 四种 role 与三条铁律

```json
[
  {"role": "system",    "content": "You are PaiCLI..."},  // 打头，全局设定
  {"role": "user",      "content": "帮我读 main.go"},
  {"role": "assistant", "tool_calls": [{"id":"call_abc", "function":{...}}]},
  {"role": "tool",      "tool_call_id": "call_abc", "content": "package main..."},
  {"role": "assistant", "content": "这个文件的入口是..."}
]
```

| 铁律 | 项目对应 |
|---|---|
| system 惯例打头且仅一条 | refreshSystemPrompt 固定覆写 history[0]（agent.go:245） |
| 每个 tool_call 必须有同 ID 的 tool 消息回应，否则 400 | llm.ToolResult + ID 透传（types.go:91） |
| arguments 是字符串化的 JSON | string(call.Function.Arguments)（client.go:213）+ normalizeFunctionArguments |

细节：role=tool 消息没有 name 也合法，但项目多发（client.go:199-201）——部分 provider 用 name 区分结果归属，兼容性冗余。

### 响应两种形态

非流式：`choices[0].message`（content + tool_calls 完整返回）+ usage。
流式：`choices[0].delta`（增量），usage 只在最后 chunk。

## 9. 批判点（面试官钓鱼区）

1. **无重试**：流式解析在 chatRequest 返回后才接管，网络抖动一次整个 turn 报错。改法：attempt 循环 + 指数退避，但需先解决"已 emit 给 Observer 的部分"的幂等性——引出"流式系统里重试与副作用"的深水区
2. **缺 `stream_options: {"include_usage": true}`**：严格按 OpenAI 规范，不带此参数流式响应没有 usage——项目能拿到 token 数是靠 DeepSeek 等国产服务默认就放最后一个 chunk。在 OpenAI 官方端点会静默丢 usage。修法一行：`if stream { body["stream_options"] = map[string]any{"include_usage": true} }`。能指出"这个实现依赖了 provider 的宽松行为"是高阶信号

## 10. 面试话术合集

**流式客户端总述**：

> "LLM 客户端这块我实现了一个 OpenAI 兼容的流式客户端：SSE 用 bufio.Scanner 按行解析，跳过心跳注释，遇到 [DONE] 终止，每条 data 反序列化成增量 chunk。三个累积通道里最 tricky 的是 tool call——arguments 是被任意切碎的 JSON 字符串碎片，我用 delta.index 做 map 的路由键，id 覆盖、name/arguments 累加，收尾按 index 排序并兜底造 ID，保证配对契约不破。兼容层还有两处防御：reasoning 双字段名、arguments 双重编码。超时是两层——http.Client 180s 兜底，ctx 贯穿到请求实现精确取消。要补的话我会加流式重试，但需要先解决已 emit 部分的幂等问题。"

**HTTP 流式原理**：

> "http.Client.Do 返回的时机是响应头部解析完，不是身体传完——Body 是惰性 io.ReadCloser，本质是还连着服务器的流。流式模式下服务器先秒回头部，然后按 token 逐行推送 SSE，我的 Scanner 循环每捞到一行就解析 emit，用户 30ms 就看到第一个字，而不是 25 秒后看全文。scanner.Buffer 不是接收数据的，是给按行切割机配缓冲——默认单行上限只有 64KB，LLM 场景单行轻松超过，所以配到 4MB，超限错误由 scanner.Err() 捕获。"

**ChatStream 设计**：

> "ChatStream 是流式+聚合一体：内部维护三个 strings.Builder 和一个按 index 寻址的 tool call builder map，文本和思维链实时累积并 emit，tool call 碎片静默累积。usage 用条件覆盖，因为流式协议只在最后一个 chunk 带一次。收尾必做两件事：查 scanner.Err()，以及 assemble 时排序加兜底造 ID。整个函数无锁——单 goroutine 状态机，并发安全靠'不共享'实现。"

**协议总述**：

> "OpenAI Chat Completions 协议的核心是 messages 数组的角色系统：system 打头定全局、user/assistant 交替、每个 tool_call 必须有同 ID 的 tool 消息回应，否则 400——配对契约是我在实现客户端和 Agent 循环时最核心的约束。模型本身无状态，多轮对话是客户端每轮重发全量 history 维持的。请求侧 tools 用 JSON Schema 描述工具，模型靠 description 理解何时调用；响应侧 delta 是增量载荷，usage 只在最后一个 chunk 出现。我还注意到实现里漏了 stream_options include_usage，是靠国产 provider 的宽松行为兜住的——这类方言差异正是兼容层的价值所在。"

## 11. 下一课预告

进入 `internal/tools` 的安全层：PathGuard / CommandGuard（policy.go）、危险操作审计日志——"怎么防止 AI 误删文件"是这类项目最出圈面试题。
