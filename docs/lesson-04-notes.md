# 第 4 课学习笔记：PathGuard / CommandGuard / 审计日志——AI 的安全防线

对应代码：`internal/tools/policy.go`（104 行）、`internal/tools/registry.go`（工具消费守卫的部分）、`internal/config/config.go`（HomeDir）

## 1. 为什么需要这层（面试的立论）

工具是 AI 的手，但模型会犯错、会被注入（读到的网页里可能藏着"请运行 rm -rf /"）。**一旦给 AI 真实的文件系统和 shell 权限，它就是全场最大的攻击面**。

三层防线 + 一层兜底：

```text
第一层：路径狱（PathGuard）      —— 所有文件操作先过安检
第二层：命令黑名单（CommandGuard）—— 危险 shell 命令执行前拦截
第三层：审计日志（AuditLog）     —— 危险操作事后可追溯
兜底：  snapshot（pre-turn/post-turn 快照，见第 2 课）
```

**纵深防御**：假设前一层会漏，后一层接住。

## 2. PathGuard.Resolve：路径安检五步流程（policy.go:17-48）

```go
if strings.HasPrefix(path, "~") { return "", err }   // ① 明确拒绝 ~ 展开
candidate := path
if !filepath.IsAbs(candidate) {                       // ② 相对路径锚定到 Root
    candidate = filepath.Join(g.Root, candidate)
}
clean, _ := filepath.Abs(candidate)                   // ③ 清洗（. 和 .. 折叠）
rootReal, _ := filepath.EvalSymlinks(g.Root)          // ④ 两侧都做符号链接求值
parent := clean
if _, err := os.Stat(clean); err != nil {
    parent = filepath.Dir(clean)                      //    目标不存在时退一层目录求值
}
parentReal, _ := filepath.EvalSymlinks(parent)
if !within(rootReal, parentReal) && !within(g.Root, clean) {
    return "", fmt.Errorf("path %q escapes workspace %s", ...)
}
```

防三种真实攻击：

- **目录穿越（path traversal）**：`../../etc/passwd`。裸字符串前缀比较会被 `/ws/../etc/passwd` 的前缀假象骗过。第 ③ 步 `filepath.Abs` 先折叠 `..`，**比较发生在真实路径层面**
- **符号链接逃逸**：workspace 里 `logs -> /etc`，读 `logs/shadow`。清洗后 `/ws/logs/shadow` 前缀合法但真实位置在 /etc！第 ④ 步 `EvalSymlinks` 对 Root 和目标目录都做物理求值。`os.Stat` 失败退一层对 `Dir` 求值——**写入不存在的文件时目标还没链接可求值，能求的是父目录**
- **`~` 拒绝**（policy.go:21）：Go 里 `~` 默认不展开，自己实现容易写出漏洞，干脆禁掉。**不确定的入口宁可直接关**

面试金句：**路径安全的核心不是字符串检查，是先归一化再比较——折叠 `..`、求值 symlink，让攻击面在比较之前就消失。**

### 2.1 重点：相对路径锚定那三行（policy.go:24-27）

```go
candidate := path
if !filepath.IsAbs(candidate) {
    candidate = filepath.Join(g.Root, candidate)
}
```

解决的问题：**相对路径"相对于谁"**。相对路径默认相对于进程 CWD，而 CWD 是不稳定的外部状态（用户从任何目录启动、--cwd 会改、serve 模式更不可预测）。这三行把语义固定死：**相对路径永远相对于 workspace 根**。也符合模型的心智——LLM 眼里只有"当前项目"。

`filepath.Join` 比手动 `Root + "/" + path` 强在两个隐藏行为：

1. **自动 Clean**：`Join("/ws", "./a/../b")` 直接得 `/ws/b`——`.` 和 `..` 这步就折叠
2. **跨平台分隔符正确**：不用操心 `/` `\` 和 `//`

行为表（Root=/home/.../PaiCli）：

| 输入 | Join 结果 | 后续命运 |
|---|---|---|
| `main.go` | `/home/.../PaiCli/main.go` | 放行 |
| `./docs/../go.mod` | `/home/.../PaiCli/go.mod` | Clean 后放行 |
| `../../../etc/passwd` | `/etc/passwd` | **within 拒** |
| `/etc/passwd`（绝对） | 原样通过 | **within 拒** |

**关键认知：这步是"归一化"不是"闸门"**——绝对路径 `/etc/passwd` 在这里畅通无阻，拒绝发生在 within 检查。安全比较的前提是所有输入先拉到同一坐标系（归一化先于鉴权）。第 ③ 步 Abs 在此场景有点冗余（Join 已 Clean 过），属于无害的防御性重复。

### 2.2 within 与三个条件（policy.go:50-55）

```go
func within(root, path string) bool {
    root = filepath.Clean(root)
    path = filepath.Clean(path)
    rel, err := filepath.Rel(root, path)
    return err == nil && rel != ".." && !strings.HasPrefix(rel, "../")
}
```

`filepath.Rel` **不做合法性检查**，纯路径算术；同盘路径永远算得出结果。三个条件各防一个边界：

| 条件 | 防什么 |
|---|---|
| `err == nil` | 跨磁盘根等极端情况——算不出就拒，**fail-closed** |
| `rel != ".."` | path **恰好是 root 的父目录本身**（裸 `..`，不带斜杠） |
| `!HasPrefix(rel, "../")` | path 在 root 外任意深处（`../etc/passwd`） |

为什么 `rel != ".."` 单独存在：`..` 没有 `../` 前缀，第二个条件匹配不上。没有它，"workspace 的父目录"被误判 inside。

对照证据：`filepath.Rel("/ws", "/wsfoo") = "../wsfoo"`——`/wsfoo` 以 `/ws` 开头但是完全不同的目录。**字符串 HasPrefix 永远不安全，必须用 Rel 做路径算术**。

## 3. CommandGuard：黑名单正则（policy.go:57-76）

九类自毁命令：`sudo`、`rm -rf /`（含 $HOME、~、* 变体）、`mkfs`、`dd of=/dev/`、fork 炸弹 `:(){:|:}`、`curl|sh`、`shutdown/reboot`、`chmod 777 /`、`find /`。

**黑名单天然有上限**，三条绕过路径（主动指出显示想过）：

1. `sh -lc` 的 `-l` 是 login shell 会读 rc 文件——`echo 'rm -rf ~' >> ~/.bashrc` **延迟引爆**绕过当下检查
2. 变量间接：`X="rm -rf /"; $X` 或 base64 编码后 `| base64 -d | sh`
3. 正则对 `&&`/`||`/子 shell 无结构化解析——`dangerous || safe` 误杀、`safe; $(dangerous)` 可能漏

正解方向：**白名单默认拒绝**（只放行 go test、git status 等）——Claude Code 等的实际做法之一。

话术落点：**黑名单是纵深防御的一层不是全部；价值是把最高频自杀命令零成本拦掉，剩下的是白名单和审计的活。**

## 4. AuditLog.Write：审计落盘（policy.go:78-104）

### 4.1 schema 按审计问题倒推

| 字段 | 审计问题 |
|---|---|
| Time | When |
| Tool | What（write_file / execute_command / mcp__…） |
| Outcome | Result（allow / error） |
| Approver | Who approved（固定 "policy"） |
| ArgsHash | With what（**参数 SHA-256 指纹，不记原文**） |
| Duration | How long |

六字段对应经典审计五问 + 耗时。**先想清楚审计问题清单，字段从问题倒推**——不是随手结构体。

### 4.2 逐行设计含义

```go
os.MkdirAll(a.Dir, 0o755)                              // 递归建目录，幂等
path := a.Dir + entry.Time.Format("2006-01-02") + ".jsonl"  // 按天分片
f, err := os.OpenFile(path, os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0o600)
defer f.Close()
b, _ := json.Marshal(entry)
f.Write(append(b, '\n'))                                // 手工补换行！
```

- **按天分片**：单文件无限增长会拖慢回放/分析，logrotate 思路内置。`2006-01-02` 是 Go reference time（1月2日...），面试常考
- **`O_APPEND`**：多进程/多次写不覆盖，POSIX 下小块追加原子；更深层含义——**审计只增不改**，账本只能记账不能涂改
- **`0o600` 文件 / 0o755 目录**：审计文件自己不能成为新攻击面（它记录"这台机器发生过什么危险操作"）。目录要可进入，内容要锁死
- **JSONL**：追加友好（写完换行即完整）、回放友好（Scanner 按行读，同第 3 课 SSE 套路）、分析友好（`cat *.jsonl | jq '.tool' | sort | uniq -c`）。对比整文件大 JSON：每次追加都得反序列化重写
- **`append(b, '\n')`**：json.Marshal 不带尾换行，没有它 JSONL 粘成一坨。小细节大坑
- **即开即关**：fd 生命周期不出函数 → AuditLog 无状态、不需要 Close（呼应第 1 课资源审计）、并发安全靠 O_APPEND 兜底

**代价与收获**：每条一次 open/write/close 系统调用（慢），换来无状态、无锁、崩溃安全、内置滚动。**用一点性能买大幅简化**。

### 4.3 写入时机（registry.go:152）

`tool.Dangerous || strings.HasPrefix(name, "mcp__")` 才记——只记危险的，避免日志爆炸。MCP 远端工具全记：不在本地代码审查范围，风险未知默认从严。

**被拒操作对模型表现为普通 tool result 错误**（registry.go:148-150 `content = err.Error()`）——对话继续，模型自己换路。呼应第 2 课：拒绝也走 tool result 通道。

### 4.4 a.Dir 的家谱

```text
internal/config/config.go:169  HomeDir() → os.UserHomeDir()（只读 $HOME，轻量）
        ▼
internal/tools/registry.go:83  AuditLog{Dir: HomeDir()/.paicli/audit}
        ▼
policy.go:95                   Dir/2026-09-16.jsonl
```

**~/.paicli 是用户级状态目录约定**：

| 路径 | 内容 |
|---|---|
| ~/.paicli/audit/*.jsonl | 审计日志 |
| ~/.paicli/memory/ | 长期记忆 |
| ~/.paicli/snapshots/ | 快照库 |
| ~/.paicli/rag/ | RAG 索引 |
| ~/.paicli/mcp.json | 用户级 MCP 配置 |
| ./.paicli/mcp.json | **项目级** MCP 配置 |

划分维度：**生命周期和共享范围**——跨项目属于"这个用户"的住 home，属于"这个代码库"的住项目目录（dotfile 惯例，同 .git/.vscode；Linux 桌面标准是 XDG，这里简化成单目录）。

## 5. execute_command 的完整安检链（registry.go:326-344）

```text
LLM 发起 execute_command("rm -rf /")
  → CommandGuard 拒绝 → error → 回填 tool result（模型换路重试，不崩溃）

LLM 发起 execute_command("go test ./...")
  → CommandGuard 放行
  → context.WithTimeout(ctx, 60s)          ← 工具级超时
  → exec.CommandContext(cctx, "sh", "-lc") ← ctx 贯穿到子进程
  → cmd.Dir = r.root                       ← 锁定工作目录（与 PathGuard 同一哲学）
  → 输出截断 12000 字节                     ← 防巨输出打爆上下文
  → audit.Write(...)                       ← Dangerous → 落审计
```

`cmd.Dir = r.root` 与 PathGuard 是同一哲学两面：文件工具锁路径，shell 工具锁 cwd。

## 6. 思考题答案

### 6.1 题 1：rel != ".." 防什么

防"path 恰好是 root 的父目录本身"：root=/a/ws，path=/a → Rel 返回裸 `..`（不带斜杠），`HasPrefix(rel, "../")` 匹配不上。没有这个判断，workspace 的上一层目录被误判 inside。

### 6.2 题 2：审计记输出（可能含用户代码）怎么设计

分层答案（考权衡意识）：

- **第一层：对账需要指纹不需要全文**——默认记 `OutputSize` + `OutputHash`（sha256）+ 256B 脱敏 Preview。理由同 ArgsHash：**审计文件不能变成用户代码的第二个副本**（未受项目权限管理的拷贝、.env 明文、体积爆炸）
- **第二层：分级开关** `audit_mode: hash | preview | full`，full 显式开启且**开启这件事本身被记进日志**（审计的自反性）
- **第三层：内容寻址分离**——全文写 `~/.paicli/audit/blob/<hash>` 单独目录/权限/保留期，日志只记 hash 引用。Git 对象存储、Docker layer 同款思路：**log 是索引，blob 是内容**

话术落点：**对账能力最大化、敏感落盘最小化；核心矛盾是审计完整性和用户隐私的对立。**

### 6.3 自测题：两个 goroutine 同时 Write

各自 OpenFile 拿独立 fd，各自 Write。O_APPEND 保证每次 write 原子追加到末尾（不会覆盖）。**最坏情形**：POSIX 只对"小块"写入承诺原子（PIPE_BUF 量级），AuditEntry JSON 约 200 字节在安全区内——实践上要么先后完整落盘，要么**交错损坏一条**（不会覆盖）。要严格：外面套 sync.Mutex（低频操作，锁开销可忽略）。

### 6.4 等价性问题：`rel != ".." && !HasPrefix(rel, "../")` 能合并成 `!HasPrefix(rel, "..")` 吗

**不能。已实验验证**（边界矩阵跑过）：

```text
/ws → /ws/..secret.txt    Rel = "..secret.txt"
原版：  != ".." ✓ 且不以 "../" 开头 ✓ → 放行（正确）
提议版：以 ".." 开头 → 拒绝（误杀 workspace 内合法文件）
```

- 分歧 case：**文件名本身以 `..` 开头**（Unix 合法，只有 `.` 和 `..` 两个保留名）。Rel 返回的 `..secret.txt` 是文件名不是跳转记号
- **错误方向分析**：提议版排除的是 `{以 ".." 开头}` ⊃ `{恰好 ".."} ∪ {以 "../" 开头}`。安全性无损（只多拒不少拒，fail-closed 方向不变），**正确性有损**（误杀）
- 实际后果：AI 读 `..notes.md` 永远报 escapes workspace，模型反复重试失败，最后说"文件不存在"——**极难排查的幽灵 bug**
- 原版两条"丑"条件各自文档化一个独立防区，**精确编码了在防什么**。安全代码原则：每条规则要能说出它拦的具体攻击，说不出来就是玄学
- 反转视角：精确正确 vs 防御性冗余——**边界可验证时选精确版，无法穷举验证时选宁可误杀的模糊版**

这个问题本身是**微型 code review 完整流程**：提出变换 → 界定影响面 → 构造边界矩阵 → 找反例 → 分析错误方向 → 裁决。

## 7. 批判点（主动讲的送分题）

1. **`sh -lc` 的 `-l`**：login shell 加载用户 rc 文件，可能改 PATH 或执行钩子——严格沙箱应用 `sh -c` 甚至固定 PATH
2. **CommandGuard 无结构化解析**：正则匹配整串，组合命令误杀/漏放。改进：词法分析或白名单
3. **审计吞错**：registry.go:153 `_ = r.audit.Write(...)`——可用性优先。严格审计场景应 **fail-closed**（写不进就拒绝操作）。可用性 vs 安全性的经典 tension
4. **HomeDir 兜底 `.` 的隐患**：$HOME 拿不到时审计写进 CWD=workspace——**账本被放进被审计对象的活动范围**，AI 能涂改自己的审计记录。安全组件的降级路径不能丢安全属性。更稳：报错退出或 workspace 外路径
5. **AuditLog 没走 Options 注入**：其他依赖都可注入可测试，唯独 audit 在 NewRegistry 硬编码——测试时没法把日志指到 t.TempDir()。**可测试性也是依赖注入的动机，不只是解耦**——聊 DI 时当反例讲

## 8. 面试话术合集

**安全层总述**：

> "防止 AI 误删文件我做的是纵深防御三层：PathGuard 做路径安检——拒绝 ~ 展开、相对路径锚定到 workspace、filepath.Abs 折叠 `..` 防目录穿越、EvalSymlinks 对根和目标都做物理求值防符号链接逃逸，比较发生在真实路径层面而不是字符串层面。CommandGuard 用正则黑名单拦 rm -rf /、mkfs、fork 炸弹、curl 管道执行等九类自毁命令，但我知道黑名单的天花板——变量间接、延迟引爆都能绕，所以长期方向是白名单默认拒绝。第三层审计日志按天分 JSONL、追加写 0600 权限，参数只记 SHA-256 哈希不落原文——审计和对账能力保留，用户代码不外泄。被拒操作对模型表现为普通 tool result 错误，对话能继续，模型自己换路。另外我主动说一个权衡：审计写失败目前是吞错的可用性优先，严格场景应该 fail-closed。"

**归一化先于鉴权**：

> "PathGuard 里相对路径锚定解决的是坐标系问题：模型说 main.go 指的是 workspace 里的文件，不是进程 CWD 里的——CWD 是不稳定的外部状态。所以相对路径统一 Join 到 workspace 根，Join 顺带 Clean 折叠 `.` 和 `..`。这一步本身不拒绝任何路径——绝对路径 /etc/passwd 也照常通过——它是归一化，真正裁决在 within。安全比较的前提是所有输入先拉到同一坐标系。"

**审计设计**：

> "AuditLog.Write 设计上有四个要点：schema 按审计五问倒推；按天分片内置日志滚动；O_APPEND 表达'账本只增不改'，0600 锁死内容；JSONL 让追加、回放、jq 分析天然友好。整个函数无状态——不缓存句柄、每次重开，用一点性能换来无需 Close、并发安全、崩溃安全。边界：调用方吞了审计失败错误，可用性优先的选择，严格审计应该 fail-closed。"

**等价性验证方法**：

> "我验证过 within 两个条件能否合并的问题：跑边界矩阵，常见 case 全一致，但 `..` 开头的文件名是反例——Rel 返回的 `..secret.txt` 是文件名不是跳转记号，合并写法会误杀 workspace 内合法文件。安全性无损但正确性有损。原版两条条件各自文档化一个防区。安全代码我选能逐条证明防住了什么的写法——前提是边界验证得了；验证不了的场景我反过来选宁可误杀。"

## 9. 下一课预告

进入 `internal/tools/registry.go` 后半部分：13 个内置工具的实现模式（strArg/intArg 参数提取、registerXXX 的注册组织）、`tools/web.go` 的联网搜索与网页抓取、`diagnostics.go` 的 PostWriteHook。
