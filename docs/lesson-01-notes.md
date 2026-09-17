# 第 1 课学习笔记：入口、启动链路、DI 与资源管理

对应代码：`cmd/paicli/main.go`、`internal/agent/agent.go`、`internal/tools/registry.go`、`internal/llm/types.go`、`internal/tools/mcp.go`

## 1. 顶层调用链

```text
main()
  ├─ signal.NotifyContext(...)          ← 优雅退出的根
  └─ rootCommand().ExecuteContext(ctx)  ← cobra 接管，ctx 一路向下传
        └─ RunE
             ├─ bootstrap(ctx)          ← 组装所有依赖（组合根）
             │    └─ environment{Config, Client, Agent, Tools, Skills, RAG, Snapshot}
             ├─ --once  → runAgentInput → ag.RunCommand()   → 打印结果退出
             ├─ --plain → runPlain()    → bufio.Scanner REPL 循环
             └─ 默认    → tui.Run(...)  → 全屏 Bubble Tea 界面
```

核心思想：**核心逻辑与界面分离**。`bootstrap()` 造出一台完整的 Agent 机器（内核），`--once` / `--plain` / TUI 是共享同一内核的三种外壳。

## 2. 三个必考细节

### 2.1 优雅退出：signal.NotifyContext（main.go:31）

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
```

- 用户 Ctrl+C → `ctx` 被 cancel → 整条调用链（HTTP 请求、Agent 循环、TUI）都能通过 `ctx.Done()` 感知并停止
- Go 1.16+ 的标准姿势，取代老写法 `signal.Notify(ch)` + 手动 select 转发，代码量减半

### 2.2 RunE vs Run（main.go:50 / main.go:188）

- `RunE: func(...) error`：错误向上返回，main 统一打印 + `os.Exit(1)`，主路径做法
- `Run: func(...)`：无返回值，只用于不可能失败的命令（如 `version`）
- 原则：能让错误冒泡就不要各自打印

### 2.3 ExecuteContext 而不是 Execute（main.go:35）

ctx 不是中途塞进去的，而是从最顶层注入，`RunE` 里用 `cmd.Context()` 取出。**取消信号从出生到死亡一条线**。

## 3. bootstrap() 是组合根（main.go:113-150）

组装顺序本身就是一张依赖图：

```text
config.Load()                     ① 读环境变量/配置 → Config
llm.NewClient(cfg)                ② 依赖 ① → LLM 客户端
skill.NewRegistry + Reload        ③ 独立 → Skill 扫描
rag.NewIndex + Load               ④ 独立 → 代码索引
agent.NewMemoryStore              ⑤ 独立 → 长期记忆
snapshot.NewService               ⑥ 独立 → 快照服务
tools.NewRegistry(Options{...})   ⑦ 依赖 ①③④⑤⑥ → 工具注册表
reg.LoadMCP(ctx)                  ⑧ 依赖 ⑦ → 动态挂载 MCP 远端工具（起子进程）
agent.New(client, reg, ...)       ⑨ 依赖 ②⑦ → Agent 本体
```

三个设计点：

1. **手工依赖注入**：8 个依赖 40 行手工组装，依赖图一目了然，没有反射黑魔法
2. **environment + Close()（main.go:95-111）**：所有需要释放的资源收拢，`defer env.Close()` 一行搞定
3. **PostWriteHook: tools.GoDiagnosticsHook**：写完 Go 文件自动跑诊断——工具行为可插拔的扩展点

## 4. DI 模式的三个层次实例

### 4.1 接口注入：agent.New()（agent.go:51）

```go
func New(client llm.Client, registry *tools.Registry, memory *MemoryStore, skills *skill.Registry) *Agent
```

- 第一个参数是 `llm.Client` **接口**（llm/types.go:8），不是具体结构体
- Agent 要的不是"DeepSeek 客户端"，而是"会说 Chat/ChatStream 的任何东西"
- 收益：换 provider 零改动；测试塞 mock Client，不需要真实 API key
- Go 谚语：**accept interfaces, return structs**

### 4.2 Options 结构体注入：tools.NewRegistry()（registry.go:71）

依赖从 1 个涨到 6 个，位置参数会变成噩梦（顺序记不住、加一个参数全体调用方报错）。收拢为 `Options` 结构体（registry.go:45-52）：

- 字段自带名字，调用处自文档化：`MemorySaver: memoryStore.Save`
- 以后加字段不破坏任何调用方（零值即默认行为）——没有泛型时代的"可选依赖"方案

### 4.3 函数注入解开 import cycle：MemorySaver（registry.go:49）

```text
agent 包拥有 MemoryStore（memory.go）
agent 包 import tools（要调用工具）
∴ tools 不能反过来 import agent —— import cycle，编译直接失败
但 tools 的 save_memory 工具确实需要"存记忆"这个能力
```

解法：tools 包自己声明最窄的函数签名 `func(content, scope string) error`——**只描述它需要什么，不描述提供者是谁**。在组合根完成对接：

```go
memoryStore := agent.NewMemoryStore(...)          // agent 包的产物
reg := tools.NewRegistry(project, tools.Options{
    MemorySaver: memoryStore.Save,                // method value → 函数，两个包互不知晓
})
```

`memoryStore.Save` 是 **method value**（绑定接收者的方法可直接当函数传）。

**DI 的本质：依赖的"声明"在 tools，依赖的"实现"在 agent，"接线"只发生在 main。三个包解耦，编译器满意，测试也满意（传 `func(c, s string) error { return nil }` 即可）。**

## 5. 什么时候才值得上 DI 框架

先分清两个概念（面试加分第一步）：

- **依赖注入是模式**：`agent.New(client, ...)` 就是 DI，项目到处在用
- **DI 框架是工具**：wire / dig / fx。问题从来不是"要不要 DI"，而是"要不要框架"

| 维度 | 手工组装（本项目） | 该上框架了 |
|---|---|---|
| 组件数量 | < 20 个，bootstrap 40 行读完 | 几十个 service，组装文件几百行 |
| 依赖深度 | 扁平，两层以内 | 依赖链深，改一处构造函数连锁改五处 |
| 组装根数量 | 1 个（once/plain/serve 复用同一 bootstrap） | 多个且不同构：HTTP、gRPC、worker、CLI 各一套 |
| 生命周期 | 只有 Close()，defer 一行搞定 | 启动顺序、健康检查、优雅停机编排、per-request 作用域 |
| 团队协作 | 一人/小组 | 多团队往中央组装文件提 PR 天天冲突 |

三种递进方案：

1. **手工组装**：显式、零魔法，依赖图肉眼可读
2. **google/wire**：编译期**代码生成**组装代码，本质还是显式，错了编译期报，不引入反射
3. **uber/fx**（基于 dig）：运行时反射解析依赖图 + lifecycle 管理。代价：依赖关系藏进注册表、报错是反射堆栈、新人上手成本高。适合超大团队统一服务模型

Go 社区文化倾向 1 和 2：**clear is better than clever**。

## 6. bootstrap 中途失败：cleanup list 模式

### 6.1 先审计：当前到底泄不泄漏？

| 组件 | 持有资源？ | 证据 |
|---|---|---|
| cfg / codeIndex / skillRegistry / memoryStore / snapshots | 否 | 纯内存对象，文件读写即开即关 |
| client | 否 | `llm.Client` 接口根本没有 Close 方法 |
| reg（Registry） | **是** | `mcpClosers`（mcp.go:45,256）—— stdio MCP 是 `cmd.Start()` 起的**子进程** |

失败路径分析：

- `llm.NewClient` 失败 → 此刻一无所有，安全
- `skillRegistry.Reload` 失败 → client 无需关闭，安全
- `reg.LoadMCP(ctx)` 之后再无返回 error 的步骤 → **当前恰好不会泄漏**

结论：**当前代码实际上不泄漏——但这是"偶然安全"**：唯一持有 OS 资源的操作之后恰好没有失败路径。结构是脆弱的，明天有人加一个可失败步骤就泄漏了。**先审计再动手，比上来就写修复代码高一档。**

### 6.2 修法：named return + cleanup list

```go
func bootstrap(ctx context.Context) (env *environment, err error) {
	var cleanups []func()
	defer func() {
		if err != nil {                          // 失败时才清理
			for i := len(cleanups) - 1; i >= 0; i-- {
				cleanups[i]()                    // LIFO 逆序执行
			}
		}
	}()

	reg := tools.NewRegistry(project, tools.Options{...})
	cleanups = append(cleanups, reg.Close)       // 每创建一个资源，登记一个清理动作

	mcpReady, mcpTools := reg.LoadMCP(ctx)
	ag := agent.New(client, reg, memoryStore, skillRegistry)
	return &environment{/* ... */}, nil          // 成功：cleanups 不执行，责任移交
}
```

三个细节：

1. **named return `(env *environment, err error)` 是整个模式的支点**——defer 闭包能读到函数最终的 err。没有它，defer 里不知道该不该清理
2. **逆序执行（LIFO）**：与 defer 语义一致。资源有依赖关系时必须先关 B 再关 A（先创建的后关闭）
3. **所有权转移**：成功时 cleanups 不执行，清理责任经 `environment.Close()` 移交调用方（`defer env.Close()`）；失败时 bootstrap 自己收拾残局。一句话：**谁创建谁负责到底，除非成功交货**

模式出处：**cleanup list**，etcd 和 Kubernetes（kube-apiserver 启动链）源码里到处是。变体 success flag 法（`ok := false; defer func(){ if !ok { cleanup() } }()`）效果相同，但 named return 判断 err 的写法更 Go 风格——语义直接绑在"是否出错"上。

## 7. 怎么判断"是否持有资源"

原则：**Go 的 GC 只回收堆内存，其他一概不管**。判断的等价问题："这个对象的生命周期里，有没有东西逃出了 GC 的管辖范围？"

| 资源 | 释放方式 | 泄漏后果 |
|---|---|---|
| 文件描述符 | `f.Close()` | fd 耗尽，`too many open files` |
| 子进程 | `cmd.Wait()` / 杀进程 | 僵尸进程占 PID |
| 网络连接 | `conn.Close()` | 连接堆积，对端打满 |
| goroutine | 退出条件 / cancel | goroutine 泄漏，内存涨 |
| 定时器/取消函数 | `stop()` / `defer cancel()` | 定时器空转 |

审计四问（拿到一个类型依次问）：

1. **看字段**：结构体里存了 `io.Closer` / `*os.File` / `net.Conn` / `*exec.Cmd` / `sql.DB` / 喂 goroutine 的 channel 吗？
2. **看构造函数**：内部调了 `Start`（而非 `Run`）、`Dial`、`Open`（且结果存字段）吗？起的后台 goroutine 比函数活得久吗？
3. **看签名**：类型自己有 `Close` / `Stop` / `Shutdown` 方法吗？（`http.Client` 没有，`sql.DB` 有——标准库会用签名诚实告诉你）
4. **想生命周期**：这东西的"非内存部分"交给谁了？

项目实证（三条 grep 扫全项目）：

- `os.Open/OpenFile` 三处（policy.go:96 / snapshot.go:130 / config.go:222）→ 全部函数内即开即关，fd 没存进结构体字段 → 这些包的构造函数**不持有 fd**
- `cmd.Start` 且 cmd 存字段 → `stdioMCPClient` **必须 Close**（也确实实现了）
- `go func` 三处：registry.go:117 调用内起、调用内 Wait 收尾；runtime/server.go:44 自有停机编排 → 无 goroutine 泄漏

## 8. 面试话术合集

**优雅退出**：

> "我在 CLI 里用 `signal.NotifyContext` 统一处理中断，让 context 取代信号 channel，这样取消语义能传播到每一层，包括正在流式输出的 LLM 请求。"

**DI 实践**：

> "依赖注入我在项目里到处用——所有组件都通过构造函数接收依赖，`bootstrap()` 就是唯一的组合根，40 行手工组装 9 个组件，依赖图一目了然。我用三种粒度做 DI：单依赖直接收接口，如 `agent.New(client llm.Client)`，方便换 provider 和 mock；多依赖用 Options 结构体，加字段不破坏调用方；最典型的是 `tools.Options.MemorySaver`——tools 包需要存记忆能力，但 MemoryStore 属于 agent 包，而 agent 又 import tools，直接引用会 import cycle，所以我声明了一个函数类型，在 main 的组合根里把 `memoryStore.Save` 接进去。依赖声明、实现、接线三处分离。"

**DI 框架时机**：

> "上 DI 框架的时机取决于两个信号：一是依赖图大到人工维护组装代码成为负担，二是需要生命周期编排和多组装根。真到那一步我会优先选 wire 这种编译期生成的方案，保持显式性；fx 那种运行时容器适合超大团队统一服务模型，小项目用了是负收益。"

**cleanup list**：

> "这个 bootstrap 当前恰好不泄漏，因为唯一持有 OS 资源的是 Registry 里的 MCP 子进程，而它之后的步骤都不会失败——但这是偶然安全。我的修法是 cleanup list：函数签名改成 named return，维护一个 `[]func()` 清理队列，每创建一个资源就 append 它的 Close，defer 里判断 `err != nil` 时逆序执行。成功时清理责任通过 environment.Close 移交给调用方。这个模式在 etcd 和 Kubernetes 启动链里是标配，核心思想是'谁创建谁释放，交货即转移所有权'。"

**资源审计**：

> "我判断一个组件是否需要显式释放，核心原则是 Go 的 GC 只回收堆内存。审计时看三处：结构体字段是否持有 fd、进程、连接或长生命周期 goroutine；构造函数是否做了 `cmd.Start`、`net.Dial` 这类操作且把句柄存进了字段；类型是否暴露 Close/Stop 签名。以这个项目为例，我 grep 了 `os.Open`/`net.Dial`/`go func`，发现所有文件操作都是函数内即开即关，唯一把子进程存进字段的是 MCP 客户端——所以 bootstrap 里只有 tools.Registry 需要走 cleanup list，其他组件交给 GC 就够了。"

## 9. 下一课预告

进入 `agent.RunWithObserver` 的 ReAct 主循环：LLM 流式调用、tool calling 协议、`ExecuteAll` 的并发信号量（`chan struct{}, 4`）、ctx 取消如何中途打断循环。
