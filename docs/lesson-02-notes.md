# 第 2 课学习笔记：ReAct 主循环、并发执行与 Go 并发知识体系

对应代码：`internal/agent/agent.go`、`internal/tools/registry.go`（ExecuteAll/Execute）

## 1. ReAct 是什么

ReAct = **Rea**soning + **Act**ing：模型先"想"（输出推理文本），再"做"（发起 tool call），拿到工具结果后继续想……循环往复直到给出最终答案。这个循环不是框架黑魔法，就是 `RunWithObserver`（agent.go:144）里一个 40 行的 for 循环。

## 2. 主循环逐段拆解（agent.go:144-200）

```go
for iter := 0; iter < 10; iter++ {                    // ① 迭代上限
    resp := a.streamChat(ctx, a.history, tools, ...)  // ② 问 LLM（带全部工具定义）
    if len(resp.ToolCalls) == 0 {                     // ③ 没有工具调用 → 最终答案
        a.history = append(a.history, llm.Assistant(final))
        return final, nil
    }
    // ④ 防死循环护栏
    a.history = append(a.history, llm.AssistantWithTools(resp.Content, resp.ToolCalls))
    results := a.tools.ExecuteAll(ctx, resp.ToolCalls) // ⑤ 并发执行工具
    for _, result := range results {
        a.history = append(a.history, llm.ToolResult(result.ID, result.Name, result.Content))
    }                                                   // ⑥ 结果回填 history，进入下一轮
}
```

五个关键认知：

1. **循环的"油"是 a.history**。消息流方向：user → assistant(tool_calls) → tool(result) → assistant(tool_calls) → …。**终止条件不是外部控制，而是 LLM 自己不再发 tool call**。
2. **迭代上限 10 是硬护栏**（agent.go:158）。超限不报错，而是调 `finalizeWithoutTools`（agent.go:333）——**把工具定义从请求里去掉**再问一次，物理上剥夺模型继续调工具的能力，强制收口。
3. **快照夹住整个 turn**（agent.go:149-151）：pre-turn / post-turn 各一个快照。两个 `_`——快照失败不阻塞主流程，尽力而为的旁路能力。
4. **refreshSystemPrompt 每回合重建**（见第 4 节）。
5. **Observer 是贯穿全项目的 UI 解耦模式**（agent.go:41）。Agent 只 emit 事件，TUI、Runtime API、--once 各自决定怎么消费。Agent 不知道 UI 存在。

## 3. tool_call_id 配对契约（协议核心）

OpenAI tool calling 协议要求：**每个 tool_call 必须有且仅有一条 `role: "tool"` 的消息回应，`tool_call_id` 严格对应**。缺了、漏了、ID 错了，下一轮请求直接被服务端拒绝（400）：

```
400: An assistant message with 'tool_calls' must be followed by tool
messages responding to each 'tool_call_id'.
```

项目里的守约三行（agent.go:183-192）：

```go
a.history = append(a.history, llm.AssistantWithTools(resp.Content, resp.ToolCalls)) // 记下"欠条"
results := a.tools.ExecuteAll(ctx, resp.ToolCalls)                                 // 干活
for _, result := range results {
    a.history = append(a.history, llm.ToolResult(result.ID, ...))                  // 按 ID 还债
}
```

`result.ID` 从 LLM 返回的 `call.ID` 一路透传（registry.go:135,143,162），全程不自己造 ID。

**更深一层：失败的工具也必须回消息**。未知工具、参数解析失败，照样返回 Result，只是 Content 是错误文本。**"工具报错"和"没有工具结果"是两回事**：前者模型能看到错误、可以换方式重试；后者直接违反协议，对话 400 报死。这就是为什么 `ExecuteAll` 保证 `len(results) == len(calls)`——包括失败的那几个。

## 4. refreshSystemPrompt：为什么要每次重建（agent.go:241-246）

```go
func (a *Agent) refreshSystemPrompt(query string) {
    if len(a.history) == 0 {
        a.history = append(a.history, llm.System(""))
    }
    a.history[0] = llm.System(a.systemPrompt(query)) // 覆写固定槽位，不是 append
}
```

时机：每个用户回合开头（agent.go:153），**不是 ReAct 10 轮循环的每轮**——一轮内 query 不变。

systemPrompt 的原料（agent.go:248-275）：

- **日期** `time.Now()`：模型没有时钟，只能靠注入，跨天不重建就一直是"昨天"
- **Provider/Model**：基本不变
- **Skill 索引** `IndexForPrompt(4000)`：load_skill 动态加载会改变启用状态
- **长期记忆** `memory.Relevant(query, 8)`：**随 query 变——这是灵魂**。检索式记忆注入（简易 RAG）：几百条记忆不能全塞，只放相关的

底层认知：**LLM 无状态**。"连续对话"的幻觉是客户端维持的——每轮重发完整 history。system prompt 天生是**动态上下文的装填口**。覆写 history[0] 而不是 append：不膨胀 token、符合 system 打头且只有一条的协议惯例。

## 5. 护栏：模型死循环怎么办（agent.go:348-362）

```go
func toolCallSignature(call llm.ToolCall) string {
    return call.Function.Name + ":" + strings.TrimSpace(string(call.Function.Arguments))
}
```

`repeatedToolCall` 用 `map[string]int` 给「工具名+参数签名」计数，**同一签名出现第二次**判定死循环，走 `finalizeWithoutTools` 收口。处理真实场景：模型反复 grep 不存在的字符串、反复读同一个文件。

**护栏不是熔断，是降级**——从带工具模式降级到纯文本模式，保底给出答案。

## 6. ExecuteAll 并发执行（registry.go:111-128）——本课精华

```go
results := make([]Result, len(calls))     // A 预分配 slice，按索引写
var wg sync.WaitGroup
sem := make(chan struct{}, 4)             // B channel 当信号量
for i, call := range calls {
    wg.Add(1)
    go func(i int, call llm.ToolCall) {   // C 参数显式传递
        defer wg.Done()
        sem <- struct{}{}                 // D 先占坑再干活
        defer func() { <-sem }()
        cctx, cancel := context.WithTimeout(ctx, 90*time.Second) // E 派生超时
        defer cancel()
        results[i] = r.Execute(cctx, call)
    }(i, call)
}
wg.Wait()
```

| # | 惯用法 | 要点 |
|---|---|---|
| A | 预分配 slice，按索引写 | 每个 goroutine 写自己的下标，无竞争；对比共享 slice append 必须加锁 |
| B | channel 当信号量 | `make(chan struct{}, 4)` 容量即并发上限，struct{} 零字节。限流原因：工具碰文件系统/起子进程，无限并发打爆本机 |
| C | 参数显式传递 | Go 1.22 前 for 变量复用是经典坑；显式传参免疫 |
| D | sem 先占后放 | 双 defer 保证异常路径也放坑 |
| E | timeout 派生 | 从父 ctx 派生：Ctrl+C 取消父 ctx 全停；单工具 90s 单独杀。defer cancel() 防泄漏 |

### 为什么 results[i] 无锁安全（内存模型）

不是碰运气，是 **happens-before** 保证：每个 goroutine 的写 → `wg.Done()`（先于）→ `wg.Wait()` 返回（后于）→ 主 goroutine 读 results。**Wait 的返回就是内存屏障**，Done 之前的写对 Wait 之后可见。

## 7. channel 版重写（以通信共享内存）

```go
func (r *Registry) ExecuteAll(ctx context.Context, calls []llm.ToolCall) []Result {
    results := make([]Result, len(calls))          // ① 仍预分配，占位保序
    type envelope struct {
        idx    int
        result Result
    }
    out := make(chan envelope)                     // ② 无缓冲，同步交接

    var wg sync.WaitGroup
    for i, call := range calls {
        wg.Add(1)
        go func(i int, call llm.ToolCall) {
            defer wg.Done()
            cctx, cancel := context.WithTimeout(ctx, 90*time.Second)
            defer cancel()
            select {                               // ③ 消费者死了就别再发
            case out <- envelope{i, r.Execute(cctx, call)}:
            case <-ctx.Done():
            }
        }(i, call)
    }
    go func() {                                    // ④ 旁观者负责关闸
        wg.Wait()
        close(out)
    }

    for env := range out {                         // ⑤ 单 goroutine 消费
        results[env.idx] = env.result
    }
    return results
}
```

设计决策：

- **② 无缓冲**：每条结果一次同步交接，唯一共享点收缩成一个 channel，结构性无 race
- **④ wg.Wait 单独开 goroutine**：close 必须在所有生产者退出后，而主 goroutine 忙着 range 收数据——旁观者关闸，死锁结构上排除
- **③ select 双路**：防消费者提前退场时生产者阻塞泄漏
- **⑤ 收集单线程化**：天然获得流式消费能力——完成一个 emit 一个，不用加锁

| 维度 | 共享 slice 版（项目现状） | channel 版 |
|---|---|---|
| 正确性依赖 | "各写各下标"的人为约定 | 结构性无 race |
| 保序 | 天然按下标 | 靠 envelope.idx 回填 |
| 结果消费 | 必须等全部完成 | 流式：完成一个处理一个 |
| 代码量 | 17 行 | ~25 行 |

## 8. 思考题 1：工具 panic 了会怎样（全项目无 recover，已验证）

**调用链**：tool.Executor panic → ExecuteAll 的 go func（registry.go:117）panic → **整个进程崩溃**。

Go 铁律：**子 goroutine 的 panic 无法被父 goroutine 的 recover 捕获**。recover 只能救自己所在调用栈。子 goroutine panic 没人接，运行时直接杀进程。

后果：TUI 瞬间消失、a.history 内存丢失整个会话、main 里 recover 兜底根本到不了。

修法（加在 panic 的出生地，即 goroutine 内部）：

```go
go func(i int, call llm.ToolCall) {
    defer wg.Done()
    defer func() {
        if p := recover(); p != nil {
            results[i] = Result{                    // 必须补齐 Result！
                ID: call.ID, Name: call.Function.Name,
                Content: fmt.Sprintf("tool panicked: %v", p),
                Error:   fmt.Errorf("panic: %v", p),
            }
        }
    }()
    // ...
}(i, call)
```

**陷阱**：只 recover 不生成 Result → `len(results) < len(calls)` → 某个 tool_call_id 没有回应 → 下轮 400。"工具 panic" 也必须像 "工具报错" 一样还债。

## 9. 思考题 2：Team 加流式，Observer 够用吗

表面答案：勉强够——Event.Title 可以塞角色名，streamChat 有 thinkingTitle 参数（agent.go:312）。

真实答案：**不够，三个语义缺陷**：

1. **EventAnswerDelta 语义被污染**：ReAct 里表示最终答案增量，Team 里 Planner/Worker 的输出是中间产物，UI 按老语义渲染会把半成品当答案
2. **没有结构化元数据**：UI 给角色配颜色只能靠 `Title == "Planner"` 字符串约定，脆
3. **缺事件来源维度**：多 Agent 事件天然带"哪个角色 + 第几轮"两层归属，Event 只有 Type/Title/Content

演进方案（加字段比加事件类型平滑）：

```go
type Event struct {
    Type    EventType
    Title   string
    Content string
    Phase   string  // 新增：planner / worker / reviewer
}
```

兼容性原理同第 1 课 Options 结构体：**加字段不破坏调用方，零值即默认行为**。事件协议演进原则：**语义变化加字段，全新行为才加类型**。

## 10. 三种 RunMode 一图看懂

```text
RunCommandWithObserver ──┬─ /plan 前缀 ─→ PlanAndExecute（先出计划，再走 ReAct 执行）
                         ├─ /team 前缀 ─→ Team（Planner→Worker→Reviewer 串行流水线）
                         └─ 默认      ─→ RunWithObserver（ReAct 主循环）
```

- `PlanAndExecuteWithObserver`（agent.go:216）内部**复用 RunWithObserver**，只是把 plan 文本包进 prompt
- `Team`（agent.go:223）：三角色**串行**传递上下文（前一个输出 append 进下一个输入），每角色独立 Chat 不带循环
- 总结：plan 模式是"先规划后 ReAct"，team 模式是"角色流水线"，两种多步推理形态共享同一套工具和消息协议

## 11. Go 并发知识补充体系

### 11.1 心智模型

- **并发是结构，并行是执行**；单核机器上并发照样成立（Rob Pike: Concurrency is not parallelism）
- Go 哲学：Don't communicate by sharing memory; share memory by communicating

### 11.2 GMP 调度模型

| 概念 | 是什么 | 关键事实 |
|---|---|---|
| G | goroutine | 初始栈仅 **2KB**（线程 1~8MB），按需增长 |
| M | OS 线程 | 真正干活的执行体 |
| P | 调度上下文 | 数量 = GOMAXPROCS（默认核数），各持本地运行队列 |

进阶三点：work stealing（空闲 P 偷 G）；syscall 阻塞时 M 让出 P（所以 execute_command 阻塞不拖垮 CLI）；Go 1.14+ 信号异步抢占（纯 for{} 不再饿死调度器）。

### 11.3 goroutine 泄漏

成因三件套：**channel 没人收、没人 close、等待没有退出条件**。检测：`runtime.NumGoroutine()` 打点、pprof goroutine profile。预防万能公式：

```go
select {
case out <- v:      // 干活
case <-ctx.Done():  // 旁观者散了就撤
}
```

### 11.4 WaitGroup 三条军规

1. `Add` 必须在 `go` **之前**——否则 Wait 可能抢在 Add 前执行
2. `Done` 就是 `Add(-1)`，必须配对（defer wg.Done() 写在第一行）
3. **值复制是重罪**——传参必须 `&wg`

### 11.5 channel 全语义

| 操作 | nil channel | open | closed |
|---|---|---|---|
| 读 `<-ch` | 永久阻塞 | 阻塞或成功 | **零值 + ok=false**（安全，可排干） |
| 写 `ch <-` | 永久阻塞 | 阻塞或成功 | **panic** |
| `close` | panic | 成功 | panic |

配套规则：

- **谁生产谁 close**；多生产者由第三方协程在 `wg.Wait()` 后 close
- 无缓冲 = **同步交接**（rendezvous）；有缓冲 = 异步直到满
- **方向限制**：签名写 `chan<- T` / `<-chan T`，把误用掐死在编译期

### 11.6 select 冷知识

- 多个 case 同时就绪时**随机**选——防饿死，别依赖顺序
- 热循环里别用 `time.After`（每次新造 timer），用 `time.NewTimer` + `Reset`

### 11.7 context 取消树

- 树形传播：cancel 父节点，整棵子树收到
- **WithTimeout 必须 defer cancel()**——否则 timer 和节点挂树上直到超时才释放（registry.go:121-122 每个工具都守了这条）
- ctx.Value 只放请求级元数据（traceID），不是传业务参数的地方
- 项目全景：signal.NotifyContext（根）→ 90s 工具超时 → exec.CommandContext（子进程也吃取消）——**从 Ctrl+C 到子进程一条线**

### 11.8 锁家族与选型顺序

| 工具 | 场景 | 项目实例 |
|---|---|---|
| Mutex | 通用互斥 | — |
| RWMutex | 读多写少 | registry.go:138 查 map 用 RLock，查找后立即释放不包住工具执行——细粒度 |
| sync.Once | 懒加载单次初始化 | — |
| atomic | 单变量 CAS | — |
| channel | 流转数据/编排 | ExecuteAll 信号量 |

选型顺序：**无锁结构（各写各下标）→ channel → 锁 → 原子**。Go 的 Mutex **不可重入**（对比 Java ReentrantLock），同 goroutine 二次 Lock 直接死锁。

### 11.9 errgroup：手写编排的官方轮子

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(4)                     // = 手写的 chan struct{}, 4 信号量
for i, call := range calls {
    i, call := i, call            // Go 1.22+ 可省
    g.Go(func() error { ... })
}
if err := g.Wait(); err != nil {} // 返回第一个 error，ctx 自动取消
```

SetLimit 就是 ExecuteAll 手写信号量的官方版，附带"一个失败、全体取消"语义。

### 11.10 -race 检测器

`go run -race` / `go test -race`，基于 vector clock 运行时拦截内存访问。本地测试和 CI 永远开着，race 检测器报出来的**没有误报**。

## 12. 面试话术合集

**ReAct 总述**：

> "ReAct 循环是我这个项目的核心：一个最多 10 轮的 for 循环，每轮把完整 history 连同全部工具定义发给 LLM。模型返回 tool call 就并发执行（信号量限 4 并发、单工具 90 秒超时、结果按 ID 回填成 tool role 消息），返回纯文本就作为最终答案。护栏有两层：迭代上限和工具签名去重，触发后都降级到无工具模式强制收口。整个循环靠 Observer 事件流与 UI 解耦，所以 TUI、Runtime API、一次性命令能共享同一个 Agent 内核。"

**工具配对契约**：

> "OpenAI 协议要求每个 tool_call 必须有对应 tool_call_id 的 tool 消息回应，否则下轮 400。所以我的 ExecuteAll 保证结果数等于调用数——包括失败的和 panic 的，失败结果以错误文本回填，模型能看到错误并换方式重试。"

**并发安全**：

> "results[i] 按下标写无锁安全靠的是 happens-before：wg.Done 先于 wg.Wait 返回，Wait 的返回就是内存屏障。但我也会主动说它的边界——如果改成 append 或加汇总收集器就会 race，更稳的是 channel 版：worker 发带下标的结果进无缓冲 channel，第三方 goroutine 在 wg.Wait 后 close，主 goroutine range 收集，共享点收缩成一个 channel，还能流式 emit。"

**panic 传播**：

> "子 goroutine 的 panic 父协程 recover 不住，任何工具 panic 会崩掉整个 CLI。修法是在 goroutine 内部 recover 并转成失败的 Result——必须补齐 Result，否则 tool_call_id 没有回应，下轮请求被 400 拒掉。"

**事件协议演进**：

> "给 Team 加流式时，现有 Observer 有语义缺陷：AnswerDelta 不再表示最终答案、缺角色元数据。我会用加字段而不是加事件类型的方式演进——加 Phase 字段，零值不影响老消费者，这和 Options 结构体可选依赖是同一个兼容性原理。"

## 13. 下一课预告

进入 `internal/llm`：OpenAI 兼容流式客户端——SSE 报文解析、tool call 增量拼接、重试与超时。参考 docs/sse-notes.md 提前预习报文格式。
