# AgentPulse Unified Domain Model / 统一领域模型

## Status and boundary / 状态与边界

This document is the canonical semantic contract shared by AgentPulse Providers, Core services, and Channels. It defines domain meaning and local invariants only. The Rust `agentpulse-core` crate implements this contract but is not its source of truth.

本文档是 AgentPulse Provider、Core 服务与 Channel 共享的权威语义约定。它只定义领域含义和局部不变量。Rust `agentpulse-core` crate 负责实现本约定，但不是规范本身的权威来源。

This version does not define a wire format, field casing, enum discriminants, Capability bit positions, persistence schema, or protocol version. Provider-private payloads cannot pass through the model as custom variants.

本版本不定义线格式、字段大小写、枚举判别值、Capability 位值、持久化 Schema 或协议版本。Provider 私有 Payload 不得以 Custom Variant 形式穿透本模型。

## Shared values / 通用值对象

- AgentPulse-owned entities use distinct UUIDv7 identifiers: Provider, Channel, Session, Event, Interaction, Command, Plan Item, Choice Option, and Tool Call IDs / AgentPulse 自有实体分别使用强类型 UUIDv7 标识，禁止混用。
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

- Approval with one or more unique allowed scopes: Once or Session / Approval 至少提供一个唯一 Scope：Once 或 Session。
- Single- or multiple-choice with one or more uniquely identified options / 单选或多选至少提供一个 ID 唯一的 Option。
- Non-sensitive single- or multiline text input / 非敏感的单行或多行 Text Input。

An `InteractionResponse` identifies its request, Session, source Channel, response time, and corresponding response payload. Validation rejects mismatched request or Session IDs, responses preceding requests, expired responses, mismatched payload kinds, disallowed approval scopes, duplicate choices, choices absent from the request, and multiple values for a single-choice request.

`InteractionResponse` 标识对应 Request、Session、来源 Channel、响应时间与匹配的响应 Payload。验证会拒绝 Request 或 Session 不匹配、响应早于请求、响应过期、Payload 类型不匹配、Approval Scope 不受支持、重复 Choice、未知 Choice，以及单选请求的多值响应。

`AgentCommand` contains its ID, target Session, source Channel, issue time, and either `SubmitPrompt` or `CancelSession`. SubmitPrompt requires Provider `PROMPT_SUBMIT` plus Channel `REMOTE_COMMAND` and `TEXT_INPUT`; CancelSession requires Provider `CANCEL` plus Channel `REMOTE_COMMAND`.

`AgentCommand` 包含自身 ID、目标 Session、来源 Channel、发出时间，以及 `SubmitPrompt` 或 `CancelSession`。SubmitPrompt 需要 Provider `PROMPT_SUBMIT` 与 Channel `REMOTE_COMMAND`、`TEXT_INPUT`；CancelSession 需要 Provider `CANCEL` 与 Channel `REMOTE_COMMAND`。

Form and sensitive-input semantics are reserved for later versions / Form 与敏感输入语义留待后续版本定义。

## Events / 事件

`AgentEvent` is a normalized envelope containing a global Event ID, Session ID, positive sequence, UTC occurrence time, and one payload. Sequence values are strictly increasing within each Session; a later stream store or reducer enforces cross-event monotonicity.

`AgentEvent` 是标准化 Envelope，包含全局 Event ID、Session ID、正 Sequence、UTC 发生时间及一种 Payload。Sequence 在每个 Session 内严格递增；跨事件单调性由后续 Stream Store 或 Reducer 强制。

Payloads cover:

- Session start with its initial snapshot / 携带初始快照的 Session Start。
- Execution-state and independent connectivity changes / 执行状态与独立连接状态变化。
- User-facing Info, Warning, and Error messages / 面向用户的 Info、Warning 与 Error 消息。
- Sanitized Tool Started and Tool Finished activity / 已清理的 Tool Started 与 Tool Finished Activity。
- Complete Plan and Progress snapshots / 完整 Plan 与 Progress 快照。
- Interaction requests and responses / Interaction Request 与 Response。
- Remote commands / 远程命令。
- Completed, Failed, or Cancelled run outcomes / Completed、Failed 或 Cancelled 运行结果。

Event construction rejects an embedded Session, Interaction, or Command that refers to another Session. Tool Activity contains only its normalized call ID, tool name, outcome, and optional sanitized summary. Raw arguments, full output, secrets, and Provider-private payloads are outside this initial model.

Event 构造会拒绝引用其他 Session 的内嵌 Session、Interaction 或 Command。Tool Activity 只包含标准化 Call ID、工具名称、结果与可选的清理后摘要。原始参数、完整输出、秘密及 Provider 私有 Payload 不属于首版模型。
