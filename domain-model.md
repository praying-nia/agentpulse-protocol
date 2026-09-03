# AgentPulse Unified Domain Model / 统一领域模型

## Status and boundary / 状态与边界

This document is the canonical semantic contract shared by AgentPulse Providers, Core services, and Channels. It defines domain meaning and local invariants only. The Rust `agentpulse-core` crate implements this contract but is not its source of truth.

本文档是 AgentPulse Provider、Core 服务与 Channel 共享的权威语义约定。它只定义领域含义和局部不变量。Rust `agentpulse-core` crate 负责实现本约定，但不是规范本身的权威来源。

This semantic document does not define a wire format, field casing, enum discriminants, Capability bit positions, persistence schema, or protocol version. Those transport concerns are specified separately by [JSON Wire Protocol v2](wire-v2.md). Provider-private payloads cannot pass through the model as custom variants.

本语义文档不定义线格式、字段大小写、枚举判别值、Capability 位值、持久化 Schema 或协议版本；这些传输约定由 [JSON 线协议 v2](wire-v2.md)独立规定。Provider 私有 Payload 不得以 Custom Variant 形式穿透本模型。

## Shared values / 通用值对象

- AgentPulse-owned entities use distinct UUIDv7 identifiers: Provider, Channel, Session, Event, Interaction, Command, Plan Item, Form Field, Choice Option, and Tool Call IDs / AgentPulse 自有实体分别使用强类型 UUIDv7 标识，禁止混用。
- Provider- or Channel-owned identifiers use non-blank opaque external IDs / Provider 或 Channel 原生标识使用非空 opaque External ID，不解释其格式。
- Provider and Channel kinds are extensible lowercase ASCII slugs such as `codex` and `feishu` / Provider 与 Channel Kind 是可扩展的小写 ASCII slug。
- Text preserves its original Unicode form but must contain at least one non-whitespace character / 文本保留原始 Unicode 内容，但必须至少包含一个非空白字符。
- Timestamps represent UTC instants. Revisions and per-Session event sequences start at one / 时间戳表示 UTC 时间点；Revision 与每 Session Event Sequence 从 1 开始。
- Workspace paths are opaque strings with an optional display name; Core never applies local filesystem semantics / Workspace Path 是带可选显示名称的 opaque string，Core 不应用本地文件系统语义。

## Providers, Channels, and capabilities / Provider、Channel 与能力

A Provider or Channel descriptor contains its AgentPulse instance ID, implementation kind, display name, optional version, and declared capabilities. Capabilities describe current behavior, not implementation preference, and are independently declared by both sides.

Provider 或 Channel Descriptor 包含 AgentPulse 实例 ID、实现 Kind、显示名称、可选版本及所声明能力。Capability 描述当前行为，而非实现偏好，并由双方独立声明。

Provider capabilities:

```text
SESSION_STATE
TOOL_EVENTS
PLAN
PROGRESS
APPROVAL_REQUEST
APPROVAL_RESPONSE
USER_INPUT_REQUEST
USER_INPUT_RESPONSE
PROMPT_SUBMIT
CANCEL
```

Channel capabilities:

```text
NOTIFICATION
SESSION_VIEW
TOOL_VIEW
PLAN_VIEW
PROGRESS_VIEW
RICH_MESSAGE
APPROVAL
CHOICE_INPUT
TEXT_INPUT
FORM_INPUT
REALTIME_SYNC
REMOTE_COMMAND
```

Capability bit positions in any implementation are private and must never be reused as a wire representation / 任一实现中的 Capability 位值均为私有细节，不得直接作为线格式。

Complete-route executability, read-only degradation, endpoint correlation, and adapter handoff boundaries are defined in [Ports and Capability Routing](ports-and-routing.md). Providers and Channels declare capabilities; they do not independently redefine the routing combinations.

完整链路可执行性、只读降级、端点关联及 Adapter 交接边界由[端口与能力路由](ports-and-routing.md)定义。Provider 与 Channel 只声明能力，不能各自重新定义能力组合规则。

## Session model / Session 模型

`AgentSession` is a complete observed snapshot containing its Session and Provider IDs, optional Provider-native Session ID, optional title and workspace, revision, creation/update times, execution state, and Provider connectivity state.

`AgentSession` 是完整观察快照，包含 Session 与 Provider ID、可选 Provider 原生 Session ID、可选标题与 Workspace、Revision、创建/更新时间、执行状态及 Provider 连接状态。

Execution state is independent from connectivity:

- `Initializing`, `Idle`, `Running`, `WaitingForInteraction`, `Completed`, `Failed`, `Cancelled`.
- `Connected`, `Reconnecting`, `Disconnected`.

Disconnecting preserves the last known execution state. Completed, failed, and cancelled describe the latest run and do not make a multi-turn Session permanently terminal. The initial model does not enforce state-transition legality.

断线不会覆盖最后已知执行状态。Completed、Failed 与 Cancelled 描述最近一次运行，不会使多轮 Session 永久终止。首版模型不强制状态迁移规则。

## Plan and progress snapshots / Plan 与 Progress 快照

`PlanSnapshot` is a complete replacement identified by a positive revision. Its ordered items have unique IDs, non-blank content, and one of Pending, InProgress, Completed, Blocked, or Skipped. An empty Plan is valid and clears the previous Plan. Multiple items may be InProgress because Provider behavior differs.

`PlanSnapshot` 是带正 Revision 的完整替换快照。顺序 Item 具有唯一 ID、非空内容，以及 Pending、InProgress、Completed、Blocked 或 Skipped 状态。空 Plan 合法并用于清除旧 Plan。由于 Provider 行为不同，允许多个 Item 同时 InProgress。

`ProgressSnapshot` is also a complete replacement identified by a positive revision. Progress is either indeterminate or determinate. Determinate total must be greater than zero and completed units cannot exceed total units.

`ProgressSnapshot` 同样是带正 Revision 的完整替换快照。Progress 可以是 Indeterminate 或 Determinate；Determinate Total 必须大于零，Completed 不得超过 Total。

## Interactions and commands / 交互与命令

An `InteractionRequest` contains its ID, Session ID, request time, optional expiration, non-blank prompt, and one payload:

`InteractionRequest` 包含自身 ID、Session ID、请求时间、可选过期时间、非空 Prompt，以及以下一种 Payload：

- Approval with structured command/network or file-change content plus Provider-issued opaque options. An unavailable request has no options and an explicit reason / Approval 包含结构化命令/网络或文件修改内容，以及 Provider 签发的不透明 Option；不可操作请求不含 Option，并明确给出原因。
- Single- or multiple-choice with one or more uniquely identified options / 单选或多选至少提供一个 ID 唯一的 Option。
- Non-sensitive single- or multiline text input / 非敏感的单行或多行 Text Input。
- An atomic form with ordered uniquely identified fields, choices, optional Other/free text, blocking state, and a sensitive-display flag / 原子表单包含有序且 ID 唯一的字段、选项、可选 Other/自由文本、阻塞状态与敏感显示标记。

An `InteractionResponse` identifies its request, Session, source Channel, response time, and corresponding response payload. Approval responses select exactly one opaque option. Form responses atomically answer every field exactly once. Validation rejects mismatched request or Session IDs, responses preceding requests, expired responses, mismatched payload kinds, read-only approvals, unknown options, duplicate choices/fields, incomplete forms, and text where Other/free text is unavailable.

`InteractionResponse` 标识对应 Request、Session、来源 Channel、响应时间与匹配的响应 Payload；审批响应只能选择一个不透明 Option，表单响应必须一次完整回答全部字段。验证会拒绝关联或类型不匹配、过期、只读审批、未知 Option、重复 Choice/字段、不完整表单，以及不允许 Other/自由文本时提交文本。

`InteractionClosed` removes a pending request without attributing a response to the Channel. Its reason distinguishes resolution by Codex/another client from cancellation by the owning Provider lifecycle. / `InteractionClosed` 在不把响应归因于 Channel 的情况下移除 Pending Request，并区分 Codex/其他客户端已处理与 Provider 生命周期取消。

`AgentCommand` contains its ID, target Session, source Channel, issue time, and one typed prompt, cancellation, model, Plan-mode, thread, review, status, permission, or queue operation. Prompt submission requires Provider `PROMPT_SUBMIT`; cancellation requires `CANCEL`; the remaining controls require `CONTROL`. Every command requires Channel `REMOTE_COMMAND`, and commands carrying text additionally require `TEXT_INPUT`.

`AgentCommand` 包含自身 ID、目标 Session、来源 Channel、发出时间，以及一种类型化 Prompt、取消、模型、Plan 模式、Thread、Review、状态、权限或 Queue 操作。Prompt、取消与其他控制分别要求 Provider `PROMPT_SUBMIT`、`CANCEL` 与 `CONTROL`；全部指令要求 Channel `REMOTE_COMMAND`，携带文本的指令还要求 `TEXT_INPUT`。

## Events / 事件

`AgentEvent` is a normalized envelope containing a global Event ID, Session ID, positive sequence, UTC occurrence time, and one payload. Sequence values are strictly increasing within each Session; a later stream store or reducer enforces cross-event monotonicity.

`AgentEvent` 是标准化 Envelope，包含全局 Event ID、Session ID、正 Sequence、UTC 发生时间及一种 Payload。Sequence 在每个 Session 内严格递增；跨事件单调性由后续 Stream Store 或 Reducer 强制。

Payloads cover:

- Session start with its initial snapshot / 携带初始快照的 Session Start。
- Execution-state and independent connectivity changes / 执行状态与独立连接状态变化。
- User-, assistant-, or system-authored Info, Warning, and Error messages / 用户、助手或系统来源的 Info、Warning 与 Error 消息。
- Sanitized Tool Started and Tool Finished activity / 已清理的 Tool Started 与 Tool Finished Activity。
- Complete Plan and Progress snapshots / 完整 Plan 与 Progress 快照。
- Interaction requests, responses, and explicit closure / Interaction Request、Response 与明确关闭。
- Remote commands / 远程命令。
- Completed, Failed, or Cancelled run outcomes / Completed、Failed 或 Cancelled 运行结果。

Event construction rejects an embedded Session, Interaction, or Command that refers to another Session. Tool Activity contains only its normalized call ID, tool name, outcome, and optional sanitized summary. Raw arguments, full output, secrets, and Provider-private payloads are outside this initial model.

Event 构造会拒绝引用其他 Session 的内嵌 Session、Interaction 或 Command。Tool Activity 只包含标准化 Call ID、工具名称、结果与可选的清理后摘要。原始参数、完整输出、秘密及 Provider 私有 Payload 不属于首版模型。

## Session reduction / Session 状态归约

`SessionAggregate` is the deterministic current-state projection of one Session event stream. Construction requires a sequence-one `SessionStarted` event. Every later event belongs to the same Session and uses the exact next sequence. Retrying the complete last event is idempotent; a different event reusing that sequence, an older sequence, or a sequence gap is rejected without changing aggregate state.

`SessionAggregate` 是单个 Session 事件流的确定性当前状态投影。构造必须从 Sequence 为 1 的 `SessionStarted` 开始。后续事件必须属于同一 Session，并使用严格连续的下一个 Sequence。完整重试最后一个事件具有幂等性；不同事件复用该 Sequence、旧 Sequence 或 Sequence 跳号都会被拒绝，且不会改变 Aggregate 状态。

The Aggregate retains the current Session snapshot, latest Plan and Progress, active Tool Calls, unresolved Interactions, latest run Outcome, and the last applied event cursor. Plan and Progress replacements must strictly advance their independent revisions. Session state, connection, and outcome observations advance the Session revision; timestamps never regress even if event occurrence times do.

Aggregate 保存当前 Session 快照、最新 Plan 与 Progress、活跃 Tool Call、尚未解决的 Interaction、最近一次运行 Outcome，以及最后应用的事件游标。Plan 与 Progress 替换必须严格推进各自独立的 Revision。Session 状态、连接和 Outcome 观察会推进 Session Revision；即使事件发生时间倒退，Session 更新时间也不会倒退。

Tool Started and Finished events are paired by Tool Call ID. Interaction Responses must match a currently unresolved Request and pass the Request's semantic validation. Ending a run maps its Outcome to the corresponding execution state and clears active Tools and unresolved Interactions while preserving final Plan, Progress, and connectivity.

Tool Started 与 Finished 事件按 Tool Call ID 配对。Interaction Response 必须匹配当前尚未解决的 Request，并通过该 Request 的语义校验。运行结束时，Outcome 会映射为对应执行状态，同时清除活跃 Tool 与未解决 Interaction，并保留最终 Plan、Progress 和连接状态。

An optional bounded recent-event window supports lightweight inspection. Its capacity is an in-memory policy, may be zero, and never affects current-state correctness. The reducer performs no I/O and does not prescribe whether replay events come from memory, a file, a database, or a network source. An unresolved request remains in the Aggregate after its expiration time; Channels must use `expires_at` to prevent late input, and the reducer rejects any late response.

可选的有界近期事件窗口用于轻量查看。其容量属于内存策略，可以设为零，且不会影响当前状态的正确性。Reducer 不执行 I/O，也不规定重放事件来自内存、文件、数据库还是网络。未解决 Request 到达过期时间后仍保留在 Aggregate 中；Channel 必须依据 `expires_at` 禁止迟到输入，Reducer 也会拒绝迟到 Response。
