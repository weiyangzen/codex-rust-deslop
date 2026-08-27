# Codex 0.148 的 Rust 重构：把“能跑的代码”变成“有边界的系统”

**报告日期：2026-08-27**

**研究对象：OpenAI Codex 仓库的 `rust-v0.148.0` 发布及其 Rust 代码变更**

**范围：`rust-v0.147.0..rust-v0.148.0`，重点看结构性重构、删除旧路径、所有权、并发、安全和测试**

## 摘要

用户的说法是：0.148 引入了 Rust 高手，把过去 AI 生成的 Rust 屎山清掉了。先把证据边界说清楚：官方发布说明和提交记录**没有**说旧代码由 AI 生成，也没有说由一位“Rust 大神”完成。这个区间有很多作者，合并提交经常由 `copyberry` 机器人完成。旧代码的来源不能从 diff 反推出来，可能是人写的、AI 辅助的，也可能只是多年赶工后自然堆积的结果。

但用户观察到的“味道变化”是有根据的。0.148 的一批 Rust 提交表现出非常一致的工程方向：

> **不再让多个近似的真相并存，而是为每个领域指定一个 owner、一个入口、一个状态机和一条安全路径；把隐含的约定搬进类型、所有权、边界和测试。**

这不是“把代码写得更像 Rust 教科书”，也不是单纯删行。最有价值的动作包括：

- 把重复的技能加载器收敛到 `ext/skills`，再删除旧 crate，而不是继续加转发层。
- 把持久化历史类型从较大的 `codex-protocol` 挪进 `codex-history`，同时保留旧磁盘格式的读取能力。
- 去掉 `Submission` 和 `Op` 的 `Clone`，让操作在队列中被消费，而不是到处复制来绕过借用问题。
- 用 `TurnInputRequest` 和 `Started/Steered/NotSubmitted` 把 start、steer、拒绝做成明确状态机，并规定拒绝时不产生副作用。
- 用 `Cow<str>`、借用 JSON 和原地 merge patch 处理真实的分配成本；但在非法 patch、旧数据和并发时序上保持原有合同。
- 对无法安全判断的沙箱 glob 直接 fail closed；对进程 waiter 不随便 abort，确保子进程真的被回收并记录退出状态。
- 删除 lockfile、fallback、fingerprint 等不再需要或已被替代的旁路机制。

所以，所谓“铲屎”的精髓可以压缩成一句话：

> **把“编译器暂时没报错”的实现，改成“错误的状态难以表达、失败的副作用不会发生、旧数据不会突然失效、资源一定有人负责”的实现。**

## 1. 版本和证据范围

### 1.1 0.148 是什么

本报告以 OpenAI Codex 仓库的正式标签为准。这里的 `0.148` 是 **Codex 产品版本，不是 Rust 编译器 1.48**。`rust-v0.148.0` 于 **2026-08-18 22:26:03 UTC** 发布，发布页名称为 `0.148.0`，标签最终解析到提交 `3ba0f711642a888aec92a611a3f3b2211157ff89`。官方发布说明列出的用户可见内容包括：TUI 对话导出、会话 fork/archive/restore、启动时编辑 prompt、线程用量估算、Bedrock provider、异步 hooks；修复项包括恢复会话时的 cwd/approval、provider/MCP 重连、CRLF 和长 URL、沙箱限制失败关闭等。

这些功能和 bug fix 很重要，因为它们说明 0.148 不是一次“只做清理”的版本。结构重构是在继续加功能的同时进行的，不能把整个版本的删行都归功于“清理 AI 代码”。

### 1.2 变更规模

按本地仓库中两个发布标签之间的 `git diff` 统计：

- 全仓库：**1473 个文件，新增 119220 行，删除 32706 行**。
- `codex-rs`：**1441 个文件，新增 118488 行，删除 30657 行**。
- GitHub compare 显示区间包含 **381 个提交**。文件数采用标签间的本地 git diff；compare 页主要用来核对提交范围和逐项链接。

数字只能说明变化很大，不能说明代码一定变好。真正能证明重构质量的是：边界是否清楚、非法状态是否被挡住、旧数据是否兼容、测试是否覆盖失败和时序。

### 1.3 “AI 屎山”应当怎样理解

“AI 屎山”在这里最好当作一种**代码形态的比喻**，而不是代码 provenance 的断言。常见形态有：

- 为了让 borrow checker 闭嘴，先到处加 `Clone`、`Arc`、`Mutex`。
- 一个概念在多个 crate 各有一份 loader、snapshot 和过滤逻辑。
- 用 `bool`、`Option` 和字符串拼接表达本来应该是枚举的状态机。
- 为了“向后兼容”长期保留没人调用的 fallback 和 re-export。
- 处理 JSON 时复制整棵树，处理文本时每次都分配新字符串。
- 取消异步任务时只 `abort`，却没有想清楚谁负责 wait、reap、记录退出状态。
- 测试只验证 happy path，不验证拒绝、竞态、旧格式和资源回收。

这些形态也可能由人类赶工造成，不能凭外观给作者贴标签。0.148 的价值在于，它对这些形态逐个做了架构判断，而不是简单跑一次 formatter 或 clippy。

## 2. 最能说明问题的重构案例

下面按“旧问题—新做法—获得的保证”来读 diff。提交链接均指向官方仓库，便于复核。

### 2.1 技能加载器：先建立唯一 owner，再删掉旧实现

这是 0.148 最完整的一条重构链，发生在 8 月 7 日前后。

旧结构里，`core-skills` 自己维护 loader、root snapshot、namespace 和 product filtering；插件路径又有一套近似逻辑。这样的结构容易让同一目录在不同入口得到不同的技能列表，修一个 bug 也要找好几处。

新结构不是突然把 4500 行删掉，而是分阶段收敛：

1. [#37439](https://github.com/openai/codex/commit/a4b129eb3e1a6929c09d6e2e1af0638122c56f0d) 在 `codex-skills` 建立共享的 `SkillRootLoader` 接口和请求/结果类型，并让 snapshot cache 用 owner 管理的句柄和身份语义。
2. [#37440](https://github.com/openai/codex/commit/e75a1888d7e91ce56fc90d0edd32a9e6a8974686) 和 [#37444](https://github.com/openai/codex/commit/e58d9ef447785d4e81718dc11c2bcce14782fa8a) 把插件 root 也接到 host skills service，复用 snapshot 和 product policy。
3. [#37452](https://github.com/openai/codex/commit/c5d94319715d2598a2e8b2b7d0e21a3b1e83aec6) 把插件 inventory 和 capability summary 也改走共享 loader，同时保留一个重要的语义区别：legacy plugin 可以递归发现，agent plugin 只允许 skills root 的直接子项。
4. [#37457](https://github.com/openai/codex/commit/33e365b19e4a7023b9b3ed74b57aa11748165a53) 才删除 legacy core loader、root snapshot 和重复的 product filtering。这个提交 37 个文件，**新增 1222 行，删除 4530 行**。
5. [#37466](https://github.com/openai/codex/commit/b3278e96cb6df4b77b8dd93cf6c65d74990a033d) 把 skill config selector、ordered rules 和 layer-stack parsing 放进 `codex-config`，让配置规则不再依赖 `SkillMetadata` 这种不相干的领域类型。
6. [#37503](https://github.com/openai/codex/commit/beac16cccd6702d1f1d26e231b9dca50ec6e25a2) 把 host skill prompt injection 放进 skills extension，保留显式调用 telemetry、不可读提示、插件 prompt 顺序和 provider prompt 覆盖规则。
7. [#37505](https://github.com/openai/codex/commit/45f8cafa4e2ec20f9b189d5aa9409e424b6d3d09) 删除 `codex-core-skills` crate，把仍有用的 `SkillLoadOutcome` 和索引逻辑移到 extension。

这条链的重点不是“删了一个 crate”，而是**先迁移所有调用者，再删掉旧 owner**。旧 loader 删除的约 2089 行测试所覆盖的行为，并没有凭空消失；从 diff 可以看出，它们按 discovery、namespace、filesystem routing、root merge、symlink、frontmatter 等主题被迁移或重写到新 owner 下，并增加了测试支持设施。

有一个细节很能说明维护者的判断：测试中的 `RecordingFileSystem` 会记录 read、metadata、walk，并用 `Atomic`、`Notify`、`Semaphore` 人为控制 I/O 交错。从测试实现可以看出，它不是只测“最后列表对不对”，还在锁定“并发扫描时哪些调用顺序是合同”。

**得到的保证：**技能加载只有一个主要 owner；插件和 host 看到的 snapshot 更一致；递归和 direct-child 的安全语义被明确写出；旧实现删除后，测试仍跟着新边界走。

### 2.2 持久化历史：把协议杂物箱变成领域 crate

[#37871](https://github.com/openai/codex/commit/63002bdb26c939925f3fa59b9575cc0a3564cb45) 新增 `codex-history`，承载 `RolloutItem`、`RolloutLine`、`CompactedItem`、`InitialHistory` 和 `ResumedHistory` 等模型历史与持久化领域类型。这个提交涉及 127 个文件，新增 791 行、删除 673 行；原来塞在 `codex-protocol` 的一大段实现被移出，`codex-rollout` 重新导出类型供旧调用者过渡。

这不是简单的“移动文件”：

- 类型的归属从“所有协议都放这里”改成“谁负责持久化历史，谁拥有它”。
- `CompactedItem` 的旧 JSON 形状仍可读。早期版本把数字 window number 写在 `window_id`，新 decoder 会把这个旧数字解释成 `window_number`，不会让旧 rollout 无法恢复。
- 新 crate 加了 JSON round-trip、压缩历史兼容、history mode 和 multi-agent version 等测试。

同一思路还出现在 [#38399](https://github.com/openai/codex/commit/2bd8727a0c07655fafeae0e4b6fe4c2b4741b399)：Serde 的 flattened/internal-tagged envelope 与 `arbitrary_precision` 浮点表示组合时会出问题。修复不是把特殊判断散落到每个调用者，而是在 JSON 持久化边界先解 envelope，再解 payload，并让 resume 和 thread-history projection 共用 decoder。

**得到的保证：**模块边界更薄，磁盘格式兼容逻辑集中在边界，协议 crate 不再承担所有领域的历史包袱。

### 2.3 Turn input：把隐含竞态改成显式状态机

[#38275](https://github.com/openai/codex/commit/cbb7e82a8bdd2b59b6f25619e1db4b4c74de1b04) 是最能体现“不是让代码编译，而是先定义合同”的提交之一。它对 137 个文件做了大规模迁移，新增 5212 行、删除 6807 行。

新的核心类型大致是：

```text
TurnInputRequest
        |
        v
TurnInputSubmission
        |
        +-- Started { turn_id }
        +-- Steered { turn_id }
        +-- NotSubmitted { reason }
```

调用者不再自己拼 `Op::UserInput`、猜当前有没有 active turn、再决定是否排队。`CodexThread` 提供三个有名字的入口：

- `start_or_steer_turn`：空闲就启动，已有可 steer 的 regular turn 就转向。
- `start_turn_if_idle`：只允许空闲启动。
- `steer_turn`：只允许向指定的 active turn 转向。

拒绝原因也不再是一个模糊的错误字符串，而是 `NotSubmittedReason` 的变体，例如：`NotIdle`、`PendingTriggerTurn`、`PlanMode`、`NoActiveTurn`、`ExpectedTurnMismatch`、`ActiveTurnNotSteerable`、`ActiveTurnOutputSchemaMismatch`、`EmptyInput`。

实现里有几个容易被忽略的细节：

- settings 先做 preview，只有 Core 接受 input 后才 apply；拒绝时不改 settings、不入队。
- start-if-idle 会检查 trigger mailbox，并通过 reservation 防止两个并发请求都以为自己抢到了空闲 turn；出错时清理 reservation。
- steer 使用 `mem::take` 转移输入所有权，不复制大 payload。
- output schema 用 JSON `Value` 的结构相等判断，而不是把 JSON 序列化成字符串比较，因此对象键顺序不会误判，数组和标量差异仍然保留。
- 返回值只表示“Core 已经接受路由决定”，不假装已经等完 prompt hook、context persistence 或 sampling。

测试覆盖并发 start-or-steer、settings 接受/拒绝、schema compatibility、idle rejection、plan mode 和 app-server steering。这里人做的关键工作是列出**所有拒绝分支及其副作用规则**，不是单纯把旧函数换个名字。

配套的 [#38092](https://github.com/openai/codex/commit/da2803c73cd366b5e01ffe8d0e5f7d396247f827) 又把 queue admission 的确认点改为“Core 接受 input”而不是“rollout 已持久化”。因此，消息在 Core 接受后就可以从队列删除，即使之后被 prompt hook 停止；也删除 persistence-specific、hook-specific error 和 task bookkeeping。

**得到的保证：**“接受输入”“开始 turn”“转向 active turn”“持久化完成”“hook 允许执行”不再混成一个含糊的成功。每个时间点都有明确含义。

### 2.4 所有权和分配：不把 Clone 当万能胶

#### `Submission` 和 `Op` 改为 move-only

[#37901](https://github.com/openai/codex/commit/ab3b4d26d4739ad824e31d0b5d96284d84630947) 去掉 `Submission` 的 `Clone`，也去掉 `Op` 的 `Clone` 和 `PartialEq`。提交循环直接消费 operation；测试只 capture 要检查的少数变体和字段，而不是为了断言复制一个大 enum。

这首先是语义改进，其次才是性能改进。一个 operation 在队列里本来就应该只有一个消费者。删掉 `Clone` 后，错误的 ownership 假设会在编译期暴露，调用者必须重新想清楚“谁拥有它、谁消费它、谁只借用它”。

#### `Cow<str>` 让“没有变化”真的没有分配

[#38214](https://github.com/openai/codex/commit/91d6f48992ad8db636b3ca52a3a36c2fb6d75537) 以前无论输入有没有终端控制字符，sanitizer 都新建一个 `String`。现在返回 `Cow<str>`：

- 干净输入直接借用原字符串。
- 需要删除控制字符时才创建 owned 字符串。
- 如果只是删除头尾，尽量复用原来的 owned buffer，而不是再分配一次。

测试覆盖普通文本、控制序列、Unicode 控制字符、不完整 escape、buffer reuse 和多片段输出；从这些断言可以看出，改动没有为了“零分配”牺牲边界行为。

同类的小改动还有 [#38103](https://github.com/openai/codex/commit/eb9dceba1a2e658142a456c5898836774835616b)：TUI 格式化 MCP invocation 时借用 invocation、server name 和 tool name；以及 [#38358](https://github.com/openai/codex/commit/80ceab7aaa25068aee26a52353c0d63039c93b29)：单次遍历收集借用的 call ID，只有真的出现 orphan output 才 compact history。

工具搜索缓存也遵循“先定义失效条件，再优化”的顺序。[#37279](https://github.com/openai/codex/commit/57f42a81131ccf5933e7ec5dc659c381eeb5d72b) 对不可变 deferred runtime 保存 `Weak` 身份，对动态工具比较当前 search metadata；只有 registry、暴露状态、source-listing 或动态描述变化时才重建 handler。测试分别验证不可变 handler 复用、registry/exposure 变化和动态 metadata 刷新。这里减少的不是一次偶然 clone，而是把缓存命中条件写成了可检查的资源身份规则。

#### World state 原地 patch

[#38078](https://github.com/openai/codex/commit/f317dc8a17d30d8feb2c79add1d9d565be0402bf) 先在当时的 `Value` API 上直接从借用的 JSON 反序列化 typed snapshot，不先 clone 整棵 JSON 树；merge patch 按 RFC 7386 在原对象上修改，只为新增 key 创建拥有的值。它的测试验证顶层为 `null`、`true`、数组等无效 patch 时，旧 snapshot 不会被污染。随后 [#38274](https://github.com/openai/codex/commit/4b07886d593546a4aee64a09aab219dd6660497f) 又把持久化的 world-state `state` 和 patch 收窄为 JSON object map，避免调用层处理无法表达 world state 的任意 JSON 形状。

这组连续改动的顺序很重要：先保证原地优化的失败语义，再把不合法的顶层形状从 API 边界上拿掉；对象内部才交给 RFC 7386 合并。

这就是成熟的性能优化：先确定“失败不能污染旧状态”的合同，再决定哪些 clone 可以去掉。不是看见 clone 就机械删除。

### 2.5 环境和模型设置：只有一个实时真相

[#38423](https://github.com/openai/codex/commit/781445f7c6928eda08fe8dc160a6003d2f3a184b) 让 `ThreadEnvironments` 成为 live environment selection 的唯一来源。配置快照、permission profile、每 turn 配置和 MCP refresh 都从它读取；settings preview 不产生副作用，只有 accepted update 才影响后续 turn。`EnvironmentConfig` 也移到更合适的 protocol 边界。

[#38461](https://github.com/openai/codex/commit/535795f7d12495ee6ea18db3201d2f85c3606aef) 进一步把完整的 `TurnEnvironmentSelection` 保存在 resolved `TurnEnvironment` 里，而不是把 environment ID、cwd、workspace roots 拆成几组容易漂移的字段。

[#38785](https://github.com/openai/codex/commit/00f6a8a60e5c5e93d185c7fe67fd596b7e62240f) 处理另一个时间问题：turn 运行中可以修改 thread settings，但当前 sampling 不能半途换模型。实现把 model、reasoning、service tier、approval 和 telemetry 快照放进 `StepContext`；更新只影响下一 turn。集成测试会暂停 active turn，更新 settings，再检查当前 turn 的所有请求仍使用旧值。

这里的设计判断很朴素：**配置是可变的，正在执行的步骤必须是稳定的。**把二者混在一个 live struct 里，代码即使没有 data race，也会有业务 race。

### 2.6 安全和资源生命周期：不能用“继续运行”掩盖未知

#### 沙箱 glob 失败关闭

[#38026](https://github.com/openai/codex/commit/1dac3d9ca04a347632056f752b15ddfa4d7cd757) 处理 Linux deny-read glob。像 `/**/*.env` 这种模式需要从根目录扫描，无法安全切出非 root 的 ripgrep search root。旧做法如果静默跳过，表面上命令能启动，实际上 deny-read 规则没有生效。新做法直接返回 fatal sandbox construction error，并给出修改 pattern 的提示；回归测试明确要求 `/**/*.env` 被拒绝。

这是典型的 fail-closed：安全规则无法可靠实施时宁可不执行，也不假装执行了。

#### 子进程 waiter 不能随便 abort

[#37498](https://github.com/openai/codex/commit/6db53df37f4e87cbf4a01888168c11c4d356f199) 区分了两类后台任务：I/O helper 可以 abort，但 child waiter 在 terminate/drop 时要 detach，让它继续 wait 并 reap 子进程。否则 PTY 子进程可能已经退出却没有被回收，session 也拿不到 exit status。

测试覆盖 pipe terminate、drop、排队中的 PTY waiter、Unix process group，并用 `waitpid` 验证不会留下 zombie。这种改动很难靠局部读代码发现，因为问题发生在取消、线程调度和操作系统进程语义的交叉处。

[#38396](https://github.com/openai/codex/commit/779e9114ae63cad0f6d4f1f792463bd6135b7096) 还把 Linux sandbox 的孤儿进程回收、PID 1、信号转发和 fallback 路径补齐。重点同样不是“多写几个 cleanup”，而是明确谁拥有进程树的最终回收责任。

#### 审批路径收敛

作为相邻背景（该提交已在 0.147 基线，不计入本次 0.147→0.148 差异），[#37128](https://github.com/openai/codex/commit/778b8698299745122f8140922fc87e32e774ecf5) 把 permission hooks、reviewer routing、approval cache 和 user approval request 移到 `Session` 级流程。shell、unified exec、apply-patch 只描述 `ApprovalAction`，由统一流程生成 hook payload、cache key、telemetry 和 retry reason。把它放在这里是为了说明同一设计方向的前后延续，不把它冒充 0.148 的新增提交。

这避免了“同一个危险动作从不同入口走不同审批逻辑”的问题。安全边界越靠近资源 owner，越不容易被某个新入口绕开。

本次范围内的 [#38299](https://github.com/openai/codex/commit/357696c5e7127525a9259d3dcfa0574516b1fe84) 把被策略拦截的网络访问也接入同一条审批管线：请求变成 `ApprovalAction`，沿用 active turn 的 review settings（包括前一 turn 启动的后台终端），最终决定才写入 telemetry；deny amendment 会被持久化，而且当前请求仍保持拒绝。测试覆盖严格自动审核、跨 turn 后台请求、拒绝持久化和不泄露目标地址的 telemetry。这是“统一入口”在安全场景的具体落地。

### 2.7 删除不值得继续维护的旁路

有些改动不是抽象，而是做产品取舍：

- [#38011](https://github.com/openai/codex/commit/279b93242cfef379e65da97e87e44b83c5934fd7) 删除 config lockfile 的 export/replay/validation 及相关 schema 和 helper，新增 1 行、删除 1421 行。
- [#38291](https://github.com/openai/codex/commit/8d637ae3980fdae79044e638c93c7e579de3c62e) 删除没被使用的 apply_patch prompt fallback。
- [#38473](https://github.com/openai/codex/commit/5c6f498b0e35058ddf0d3ba0959a0967c5b6495b) 停止生成 accepted-line fingerprints。
- [#37461](https://github.com/openai/codex/commit/3b366654f1de1b77587ffb026c8f35507f3fe4ef) 删除未使用的 remote skills client。

另一个同类例子是 [#38682](https://github.com/openai/codex/commit/eb147c0db364d1e642439f33e3078f8712ef9aa5)：把上游 streamed/HTTP 400/403 的 `misalignment_policy_violation` 识别成 typed、不可重试的终止错误，并同步到 app-server schema；空消息也有明确 fallback。这里没有让各层猜字符串、各自决定是否重试。

真正难的是判断“这是死代码，还是一个虽少见但必须保留的兼容入口”。AI 很容易把所有旧路径都保留，再加一层 if；资深维护者会查调用关系、发布承诺、遥测和迁移历史，然后承担删除的后果。

## 3. 这些变化背后的设计理念

### 3.1 一个概念只能有一个主要 owner

技能加载由 skills extension 负责，持久化历史由 history crate 负责，环境选择由 `ThreadEnvironments` 负责，审批由 Session 负责。owner 不是目录名，而是一个责任承诺：读取、合并、过滤、缓存、错误和测试都在这里定义。

**检查方法：**看到两个模块都在做同一件事时，不要先问“能否共用一个 helper”，先问“谁应该拥有这个概念，另一个模块能否只调用它”。

### 3.2 一个入口胜过很多近似入口

`start_or_steer_turn`、共享 `SkillRootLoader`、统一 Approval flow 和共享 rollout decoder，都是把分叉入口压成一个决策点。入口少了，日志、telemetry、错误和测试才有机会一致。

### 3.3 让类型表达状态，而不是让调用者猜

`Started`、`Steered`、`NotSubmitted(reason)` 比 `Result<String, String>` 更有用；move-only 比“任何地方都能 clone”更有用；[#38682](https://github.com/openai/codex/commit/eb147c0db364d1e642439f33e3078f8712ef9aa5) 中的 typed misalignment error，比字符串里写一句“可能不对齐”更有用。

Rust 的 enum、私有字段、newtype、`Result` 和所有权系统在这里被当作设计工具，而不只是语法。

### 3.4 失败路径必须先定义副作用

成熟实现会先问：

- schema 不兼容时，settings 是否已写入？
- idle reservation 失败时，谁清理标记？
- patch 非法时，旧 snapshot 是否完整保留？
- deny-read glob 无法安全展开时，沙箱是否应该继续？
- process handle drop 后，谁 wait 子进程？

答案都被放进 API、控制流和测试，而不是留给调用者“应该不会发生”。

### 3.5 优化的是数据流，不是某一行代码

去 clone 之前先画 ownership；去分配之前先区分 borrowed/owned；改 queue 之前先定义 acceptance boundary；改异步任务之前先定义 cancellation 和 reap owner。这样优化才不会变成新的隐患。

### 3.6 兼容性属于边界，不应污染所有领域类型

旧 numeric `window_id` 的解析、Serde arbitrary precision workaround 都集中在 persistence decoder。领域代码看到的是干净的 `CompactedItem` 和 `RolloutLine`，兼容脏活在边界完成。

### 3.7 安全规则不接受“差不多执行了”

无法安全展开的 glob 就拒绝；审批从所有入口共用；插件 direct-child 与 legacy recursive 明确区分；filesystem helper 只给必要范围。安全决策比“用户这次先跑起来”优先级高。

### 3.8 删除是架构动作，不是清洁癖

删掉 loader、crate、re-export、fallback 和 lockfile，只有在调用关系、替代路径、迁移和测试都准备好时才有意义。否则只是把复杂度藏到外面。

### 3.9 测试要锁定合同，而不只是锁定输出

好测试会控制时序、检查不变性和资源状态：

- 让两个 turn input 并发抢 idle，确认只有一个成功。
- 让 active turn 暂停，再更新 settings，确认当前请求不变。
- 传入非法 patch，确认 snapshot 语义不变。
- 人为阻塞 PTY waiter，terminate 后确认最终仍被 reap。
- 给 sanitizer 干净文本，确认返回 borrowed；给边缘控制字符，确认复用 buffer。

这类测试更像“可执行的设计文档”，比一堆只走成功路径的单元测试有价值得多。

### 3.10 迁移要分阶段，删除要最后发生

技能 loader 的提交顺序是很好的模板：建立共享接口 -> 接入新调用者 -> 迁移 policy/prompt -> 搬测试 -> 删除旧实现 -> 删除旧 crate。这样每一步都能编译、能回滚，reviewer 也能看懂。

## 4. 为什么这看起来像“清掉了 AI 味”，但不等于证明旧代码由 AI 写的

所谓 AI 味，通常不是某个语法，而是**局部最优叠加后的结构**：

| 看到的形态 | 眼前的好处 | 长期代价 | 0.148 的对应动作 |
|---|---|---|---|
| 到处 `Clone`/`Arc` | 借用错误马上消失 | ownership 不清、复制成本隐藏 | `Submission` move-only、`Cow`、借用 JSON |
| 每层一个 loader/wrapper | 当前调用点改动小 | 同一规则多份实现，结果会漂移 | shared `SkillRootLoader`，删除 legacy loader |
| `bool` + `Option` 表示流程 | 写起来快 | 非法组合很多，调用者靠猜 | `TurnInputSubmission` 和 typed reason |
| 大而全 protocol crate | import 方便 | 领域边界变薄，依赖方向混乱 | 新建 `codex-history` |
| 任何错误都继续/重试 | 看起来不打断用户 | 安全规则可能失效、状态被污染 | fail closed、typed non-retryable errors |
| `abort` 所有后台任务 | 取消简单 | 子进程不 reap、状态不完整 | 保留 child waiter，单独取消 I/O helper |
| 永远保留 fallback | 暂时不担心旧调用者 | 维护矩阵越来越大 | 删除无调用 fallback、lockfile、fingerprint |
| 只测最后结果 | 测试便宜 | 竞态、旧格式和拒绝分支无人保护 | schedule/compatibility/resource tests |

这张表描述的是一种审查视角，不是对某个作者的指控。人类团队也会写出同样的代码；差别在于是否有人愿意停下来重画边界，并为删除旧路径负责。

## 5. 当前 AI 能做什么，哪些判断仍然需要人来把关

这里的“做不到”应理解为：**在没有明确合同、没有长期上下文、只接到一个局部任务时，当前常见 coding agent 不能稳定、可重复地独立完成**。这不是说模型永远不能写出类似代码；给它完整架构资料、强测试和多轮 review，表现会好很多。

### 5.1 AI 通常做得不错的部分

- 在明确 owner 和 API 后，机械迁移 import、调用签名和 serde derive。
- 按现有测试风格补 round-trip、错误分支和 fixture。
- 根据编译器错误修正生命周期、`Send`/`Sync`、trait bound 和类型转换。
- 把一个已经决定好的 helper 拆成几个文件，统一命名和文档。
- 搜索所有调用点，生成初步依赖清单和变更摘要。
- 对局部热点做有依据的 `Cow`、借用或缓存改写，并运行 benchmark/测试。

这些工作很有价值，但前提是**人先决定要保持什么合同**。

### 5.2 当前 AI 常常不可靠的判断

#### 1. 跨模块找出真正的 owner

局部上下文看不出 `core-skills` 和 `ext/skills` 哪个应该拥有 product filtering，也看不出某个 protocol 类型其实只服务于历史持久化。AI 很容易在两个地方都加一个 wrapper，让当前编译通过，复杂度反而增加。

#### 2. 决定哪些“兼容”必须保留

旧 `window_id` 的数字形状没有写在新函数签名里，只有历史文件、迁移代码和线上恢复行为能说明它重要。模型若没有读完整调用链和数据样本，可能直接改 schema，测试当下通过，旧会话却无法恢复。

#### 3. 设计并发的线性化点

“Core 接受”与“rollout 持久化”哪个算 admission 成功，不是 Rust 语法问题；`start_if_idle` 两次检查和 reservation 是否足够，也不是从一个函数能推出来的。需要知道产品要保证什么，再安排锁、mailbox、oneshot 和取消顺序。

#### 4. 理解取消、进程和操作系统语义

`abort` 一个 waiter 在普通 async 代码里似乎合理，在 PTY/Unix 子进程里可能制造 zombie。模型能生成 `tokio::spawn`，但不一定知道 waitpid、process group、PID 1 和 exit status 的责任链。

#### 5. 进行威胁建模并选择 fail-open 还是 fail-closed

`/**/*.env` 扩展失败时，继续启动还是拒绝启动，取决于“deny-read 必须有效”这个安全目标。没有威胁模型，AI 往往选择最少报错的路径，也就是静默跳过。

#### 6. 判断何时删除功能，而不是再加一层开关

删除 config lockfile、remote client 或 prompt fallback 会影响文档、用户习惯、迁移和支持成本。模型倾向于保守保留，因为删除看起来风险大；但长期维护一条没人使用的路径本身也是风险。

#### 7. 设计真正有区分度的测试

让测试“覆盖一行”很容易；让测试控制两个异步任务的交错、验证拒绝没有副作用、证明子进程已经被 reap，需要对失败模式有具体想象。测试 oracle 不是从实现自动长出来的。

#### 8. 评估性能收益是否值得复杂度

看到 clone，模型可能一律改成借用；但生命周期、跨线程和缓存边界可能要求拥有值。`Cow` 的 borrowed/owned 分支、world-state 原地 patch 和 `Weak::ptr_eq` 缓存都需要真实数据流和基准，不是风格偏好。

#### 9. 处理可观测性和运营后果

把 admission 提前到 Core 接受，会改变 telemetry 的时间点；把错误改成 typed non-retryable，会改变重试系统；把 settings 延后到下一 turn，会改变用户看到的状态。模型若只看函数返回值，容易漏掉日志、指标、回滚和客服诊断。

#### 10. 在冲突目标之间做取舍并承担责任

性能、兼容、安全、代码量和交付速度经常冲突。选择保留旧 decoder、拒绝 root glob、增加 StepContext snapshot，都是明确取舍。人可以查看事故历史、用户承诺和上线风险；模型通常只能根据当前文本猜测。

### 5.3 人类在这些提交中控制得好的地方

从 diff 和测试能看到几种很具体的人类判断：

1. **先画边界，再搬代码。** history 和 skills 都是先确定 owner，再迁移调用者和测试。
2. **把“拒绝”当成一等结果。** `NotSubmittedReason`、fatal sandbox error、invalid patch unchanged 都不是异常漏网，而是设计好的出口。
3. **把时间写进模型。** active turn 的 settings snapshot、Core acceptance boundary、queued message deletion point 都明确了“什么时候生效”。
4. **把资源责任写进生命周期。** child waiter、I/O helper、sandbox process tree 各有不同的取消和回收规则。
5. **只删除有证据的旁路。** 无调用、无文档承诺、可由新路径替代，并且测试覆盖后才删除。
6. **用反例而不是口号验收。** 非法 glob、非 object patch、旧浮点 rollout、并发 start、Unicode 控制字符和 zombie child 都是针对具体失败的测试。

## 6. 给 AI 辅助 Rust 项目的可执行做法

如果想让 AI 参与类似清理，比较可靠的流程不是“让它把屎山重构一下”，而是把人的控制点显式化。

### 6.1 人先写一页合同

在动代码前写清楚：

- 每个领域的唯一 owner 和允许依赖方向。
- 状态有哪些，哪些转移合法，拒绝时哪些字段绝不能改变。
- 当前 turn、下一 turn、持久化完成、hook 完成分别何时生效。
- 旧磁盘格式、协议字段和外部客户端必须保留什么。
- 哪些安全规则必须 fail closed，哪些错误可以重试。
- 每个后台 task、子进程和缓存的 owner 是谁。

### 6.2 让 AI 先做只读盘点

先让它输出调用图、重复实现、`Clone`/`Arc` 热点、公共 re-export、持久化字段和测试缺口；不要一上来改代码。人审核盘点是否漏了外部入口和历史格式。

### 6.3 一次只迁移一个语义边界

推荐顺序：新类型/API -> 适配一批调用者 -> 补失败和兼容测试 -> 删除旧入口 -> 再删 crate 或依赖。每个提交都能编译和回滚，review 才不会被 1000 个无关格式变化淹没。

### 6.4 给 AI 设几条硬门槛

- 不得为了消除 borrow error 自动添加 `Clone`、`Arc` 或 `Mutex`，必须说明 ownership。
- 不得扩大 `pub` 或增加 re-export，除非有明确的跨 crate 合同。
- 不得把错误改成字符串后丢失可匹配的原因。
- 不得在 schema/持久化边界删除旧形状，除非有迁移和兼容测试。
- 每个并发改动必须有拒绝、取消、超时和交错测试。
- 每个安全改动必须说明 fail-open/fail-closed 选择。
- 每个删除必须列出调用者、替代路径和回滚方案。

### 6.5 人的最终验收清单

```text
[ ] 这个概念是否只有一个 owner？
[ ] API 是否表达了 Started/Steered/Rejected 等真实状态？
[ ] 失败时 settings、queue、snapshot、telemetry 是否保持合同要求？
[ ] 是否保留旧 rollout/protocol 的必要形状？
[ ] 是否验证并发交错，而不只是顺序 happy path？
[ ] clone/alloc 的减少是否有生命周期和基准依据？
[ ] 取消后谁 wait/reap/close/flush？
[ ] 安全条件无法判断时是否 fail closed？
[ ] 删除的代码是否真的没有调用者和外部承诺？
[ ] diff、测试、日志和迁移说明是否能让下一位维护者看懂？
```

## 7. 结论

Codex 0.148 的这批 Rust 改进，最值得学习的不是某个高级 lifetime 技巧，而是一种维护大型系统的纪律：**少造一个真相，少保留一条旁路，少复制一次所有权，少吞掉一个失败；同时把剩下的规则写进类型、边界和测试。**

如果把它简单说成“Rust 大神把 AI 写的屎山删了”，会漏掉真正有用的部分。更准确的说法是：一个多人维护、持续加功能的 Rust 系统，在 0.148 前后做了一轮集中架构收敛。维护者把重复 loader、模糊 admission、分散环境字段、无意义 clone、隐式兼容和不安全的静默失败，逐项改成了可检查的合同。

当前 AI 已经能很好地执行局部搬运、机械替换和测试补全；它最不稳定的地方仍是决定“系统应该相信谁、什么时候算成功、失败能否留下副作用、旧数据要不要继续活、资源最终由谁回收”。这些判断需要完整上下文、真实运营经验和愿意为取舍负责的人。

因此，比较现实的分工不是“人写每一行，AI 不能碰架构”，也不是“AI 自主重构，人只点确认”。更好的分工是：

> **人定义边界、合同、威胁模型和验收反例；AI 负责搜索、迁移、实现和重复验证；人对删除、兼容、并发和上线风险做最终决定。**

这正是 0.148 这些提交给出的工程答案。

## 附录 A：精选提交索引

| 主题 | PR / 提交 | 关键变化 |
|---|---|---|
| 共享技能 loader | [#37439](https://github.com/openai/codex/commit/a4b129eb3e1a6929c09d6e2e1af0638122c56f0d) | object-safe `SkillRootLoader`、snapshot identity |
| 插件技能统一 | [#37444](https://github.com/openai/codex/commit/e58d9ef447785d4e81718dc11c2bcce14782fa8a) | host/plugin 共用 loader 和 snapshot |
| 旧 loader 删除 | [#37457](https://github.com/openai/codex/commit/33e365b19e4a7023b9b3ed74b57aa11748165a53) | +1222/-4530，迁移并扩充测试 |
| 技能规则归属 | [#37466](https://github.com/openai/codex/commit/b3278e96cb6df4b77b8dd93cf6c65d74990a033d) | 规则解析移入 `codex-config` |
| 删除旧 skills crate | [#37505](https://github.com/openai/codex/commit/45f8cafa4e2ec20f9b189d5aa9409e424b6d3d09) | 移除 `codex-core-skills` |
| 历史领域边界 | [#37871](https://github.com/openai/codex/commit/63002bdb26c939925f3fa59b9575cc0a3564cb45) | 新建 `codex-history`，保留旧序列化兼容 |
| move-only 操作 | [#37901](https://github.com/openai/codex/commit/ab3b4d26d4739ad824e31d0b5d96284d84630947) | `Submission`/`Op` 去 `Clone` |
| 原地 world-state patch | [#38078](https://github.com/openai/codex/commit/f317dc8a17d30d8feb2c79add1d9d565be0402bf) | 借用 JSON、原地 RFC 7386、非法 patch 不改状态 |
| queue admission | [#38092](https://github.com/openai/codex/commit/da2803c73cd366b5e01ffe8d0e5f7d396247f827) | Core 接受即确认，不等持久化 |
| MCP 借用格式化 | [#38103](https://github.com/openai/codex/commit/eb9dceba1a2e658142a456c5898836774835616b) | 避免复制 invocation/name |
| TUI sanitizer | [#38214](https://github.com/openai/codex/commit/91d6f48992ad8db636b3ca52a3a36c2fb6d75537) | `Cow<str>`、clean path 零分配 |
| turn input 状态机 | [#38275](https://github.com/openai/codex/commit/cbb7e82a8bdd2b59b6f25619e1db4b4c74de1b04) | typed start/steer/reject、原子路由 |
| 沙箱 fail closed | [#38026](https://github.com/openai/codex/commit/1dac3d9ca04a347632056f752b15ddfa4d7cd757) | 不安全 root glob 直接拒绝 |
| child waiter 生命周期 | [#37498](https://github.com/openai/codex/commit/6db53df37f4e87cbf4a01888168c11c4d356f199) | waiter 继续 reap，I/O helper 单独取消 |
| 环境唯一真相 | [#38423](https://github.com/openai/codex/commit/781445f7c6928eda08fe8dc160a6003d2f3a184b) | `ThreadEnvironments` 统一读取 |
| 环境 selection 保留 | [#38461](https://github.com/openai/codex/commit/535795f7d12495ee6ea18db3201d2f85c3606aef) | 不拆散复制 selection 字段 |
| active-turn snapshot | [#38785](https://github.com/openai/codex/commit/00f6a8a60e5c5e93d185c7fe67fd596b7e62240f) | 当前 turn 稳定，更新延后下一 turn |
| rollout decoder | [#38399](https://github.com/openai/codex/commit/2bd8727a0c07655fafeae0e4b6fe4c2b4741b399) | 浮点和 envelope 兼容集中处理 |
| 旧旁路删除 | [#38011](https://github.com/openai/codex/commit/279b93242cfef379e65da97e87e44b83c5934fd7) | 删除 config lockfile 支持 |
| 上游策略错误 | [#38682](https://github.com/openai/codex/commit/eb147c0db364d1e642439f33e3078f8712ef9aa5) | typed、不可重试的 misalignment 错误 |
| world-state 形状收窄 | [#38274](https://github.com/openai/codex/commit/4b07886d593546a4aee64a09aab219dd6660497f) | 持久化 state/patch 限定为 JSON object map |

## 附录 B：官方资料和复核方法

### 官方资料

- [Codex 0.148.0 发布说明](https://github.com/openai/codex/releases/tag/rust-v0.148.0)
- [0.147.0 到 0.148.0 的完整比较页](https://github.com/openai/codex/compare/rust-v0.147.0...rust-v0.148.0)
- [Rust Book：所有权](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
- [Rust Book：引用与借用](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)
- [Rust Book：错误处理总览](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
- [Rust Book：何时 `panic!`、何时返回 `Result`](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html)
- [Rust Book：无畏并发](https://doc.rust-lang.org/book/ch16-00-concurrency.html)
- [Rust Book：消息传递并发](https://doc.rust-lang.org/book/ch16-02-message-passing.html)
- [Rust Book：共享状态并发](https://doc.rust-lang.org/book/ch16-03-shared-state.html)
- [Rust Book：`Send` 与 `Sync`](https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html)
- [Rustonomicon：Send and Sync（unsafe marker trait 约束）](https://doc.rust-lang.org/nomicon/send-and-sync.html)
- [Cargo：SemVer 兼容性指南](https://doc.rust-lang.org/cargo/reference/semver.html)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Rust API Guidelines：为未来演进留空间](https://rust-lang.github.io/api-guidelines/future-proofing.html)
- [RFC 1105：API 演进与稳定性](https://rust-lang.github.io/rfcs/1105-api-evolution.html)
- [RFC 7386：JSON Merge Patch](https://www.rfc-editor.org/rfc/rfc7386)

### 可复核命令

在包含 Codex 仓库的机器上，可以用以下命令复核版本、规模和单个提交。命令只读，不会改工作树：

```bash
cd ~/weiyangzen/codex
git show -s --format=fuller rust-v0.148.0
git diff --shortstat rust-v0.147.0..rust-v0.148.0
git diff --shortstat rust-v0.147.0..rust-v0.148.0 -- codex-rs
git show --stat 33e365b19e4a7023b9b3ed74b57aa11748165a53
git show --stat cbb7e82a8bdd2b59b6f25619e1db4b4c74de1b04
git show --stat 91d6f48992ad8db636b3ca52a3a36c2fb6d75537
```

本报告只使用发布标签、官方 release body、官方 PR/commit 说明和可复现的 git diff 做事实依据；关于“AI 味”“Rust 高手”的部分，是基于这些证据的工程分析，不是官方归因。
