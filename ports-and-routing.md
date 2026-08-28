# AgentPulse Ports and Capability Routing / 端口与能力路由

## Status and boundary / 状态与边界

This document defines the runtime-neutral boundaries between Provider adapters, the Core/Bridge, and Channel adapters, plus the centralized policy that decides whether a complete route can execute an operation. It uses the semantic types from the [Unified Domain Model](domain-model.md) and does not add a JSON wire message.

本文定义 Provider Adapter、Core/Bridge 与 Channel Adapter 之间运行时中立的边界，以及判断完整链路能否执行操作的集中策略。它使用[统一领域模型](domain-model.md)中的语义类型，不新增 JSON 线消息。

Ports are synchronous handoff contracts. Success means that the receiver accepted the value, possibly by enqueuing it; it does not mean external Provider, network, or platform I/O has completed. The contract does not select an async runtime.

端口是同步交接契约。成功表示接收方已经接收该值，也可以只是完成入队；它不表示外部 Provider、网络或平台 I/O 已完成。本契约不选择异步运行时。

## Independent ports / 独立端口

Provider-side boundaries:

- A Provider publishes normalized `AgentEvent` values through a Provider Event Sink and identifies the configured Provider instance that emitted each Event.
- A Provider Port exposes its `ProviderDescriptor` and accepts centrally validated `InteractionResponse` and `AgentCommand` values.

Provider 侧边界：

- Provider 通过 Provider Event Sink 发布标准化 `AgentEvent`，并标识每个 Event 所属的已配置 Provider 实例。
- Provider Port 暴露 `ProviderDescriptor`，并接收经过集中校验的 `InteractionResponse` 与 `AgentCommand`。

Channel-side boundaries:

- A Channel Port exposes its `ChannelDescriptor`, consumes normalized `AgentEvent` values with their centralized route decision, and may consume a complete `AgentSession` view.
- A Channel submits `InteractionResponse` and `AgentCommand` values through a Channel Action Sink and identifies their configured source Channel.

Channel 侧边界：

- Channel Port 暴露 `ChannelDescriptor`，消费带集中路由结果的标准化 `AgentEvent`，并可以消费完整 `AgentSession` 视图。
- Channel 通过 Channel Action Sink 提交 `InteractionResponse` 与 `AgentCommand`，并标识其已配置的来源 Channel。

Provider and Channel ports never depend on one another. Port signatures contain only shared semantic values and implementation-local handoff errors. Adapter-specific payloads, rendering rules, transport frames, and provider-native commands cannot cross these boundaries.

Provider Port 与 Channel Port 彼此不依赖。端口签名只包含共享语义值与实现本地的交接错误。Adapter 专属 Payload、渲染规则、Transport Frame 及 Provider 原生命令不得跨越这些边界。

## Provider Event capability mapping / Provider Event 能力映射

The Core validates every Provider Event against its selected Provider and current Session before Channel delivery. The required Provider capability is determined only by the normalized Event payload:

Core 在向 Channel 交付前，依据所选 Provider 与当前 Session 校验每个 Provider Event。所需 Provider Capability 只由标准化 Event Payload 决定：

| Event payload | Required Provider capability |
| --- | --- |
| `SessionStarted`, `StateChanged`, `ConnectionChanged`, `SessionEnded` | `SESSION_STATE` |
| `Message` | none / 无额外能力 |
| `ToolActivity` | `TOOL_EVENTS` |
| `PlanUpdated` | `PLAN` |
| `ProgressUpdated` | `PROGRESS` |
| Approval `InteractionRequested` | `APPROVAL_REQUEST` |
| Choice or Text `InteractionRequested` | `USER_INPUT_REQUEST` |
| Approval `InteractionResponded` | `APPROVAL_RESPONSE` |
| Choice or Text `InteractionResponded` | `USER_INPUT_RESPONSE` |
| Submit Prompt `CommandIssued` | `PROMPT_SUBMIT` |
| Cancel Session `CommandIssued` | `CANCEL` |

The Event Session ID must match the selected Session, and that Session must belong to the selected Provider. A `SessionStarted` payload must also name that Provider. Missing publication capability is an invalid Provider Event, not a read-only downgrade.

Event Session ID 必须匹配所选 Session，且该 Session 必须属于所选 Provider；`SessionStarted` Payload 也必须标识该 Provider。缺少发布能力属于非法 Provider Event，不进行只读降级。

## Interaction routing / Interaction 路由

An Interaction Request is `Interactive` only when all of the following are true:

- the Provider declared the capability required to publish that request;
- the Provider declared the corresponding response capability;
- the Channel declared the corresponding input capability.

Interaction Request 只有在以下条件全部成立时才是 `Interactive`：

- Provider 声明了发布该 Request 所需的能力；
- Provider 声明了对应的 Response 能力；
- Channel 声明了对应的输入能力。

The response combinations are:

| Interaction | Provider response capability | Channel input capability |
| --- | --- | --- |
| Approval | `APPROVAL_RESPONSE` | `APPROVAL` |
| Choice | `USER_INPUT_RESPONSE` | `CHOICE_INPUT` |
| Text | `USER_INPUT_RESPONSE` | `TEXT_INPUT` |

If request publication is valid but either response side is missing, the request is delivered as `ReadOnly`. The Channel consumes this decision and must not reconstruct the capability combination. It also stops exposing input after `expires_at`. The Bridge revalidates every submitted Response against endpoint IDs, Session and Request correlation, time limits, payload semantics, and the current declared capabilities before Provider handoff.

如果 Request 发布合法，但任一响应侧能力缺失，则以 `ReadOnly` 交付。Channel 消费该判断，不能自行重新组合 Capability；到达 `expires_at` 后也必须停止提供输入。Bridge 在交给 Provider 前，必须依据端点 ID、Session 与 Request 关联、时间限制、Payload 语义及当前声明能力重新校验每个 Response。

## Command routing / Command 路由

Commands never degrade to a different executable operation. Submit Prompt requires Provider `PROMPT_SUBMIT` plus Channel `REMOTE_COMMAND` and `TEXT_INPUT`. Cancel Session requires Provider `CANCEL` plus Channel `REMOTE_COMMAND`. The command Session and source Channel must match the selected route before Provider handoff.

Command 不会降级成另一种可执行操作。Submit Prompt 需要 Provider `PROMPT_SUBMIT` 以及 Channel `REMOTE_COMMAND` 与 `TEXT_INPUT`；Cancel Session 需要 Provider `CANCEL` 以及 Channel `REMOTE_COMMAND`。交给 Provider 前，Command Session 与来源 Channel 必须匹配所选链路。

## Exclusions / 不包含内容

These contracts do not define adapter discovery, registration, lifecycle, fan-out, buffering, retries, acknowledgements, backpressure, transport, authentication, Relay behavior, persistence, or a concrete Bridge event loop. Those concerns remain later milestones.

本契约不定义 Adapter 发现、注册、生命周期、扇出、缓冲、重试、ACK、背压、Transport、鉴权、Relay 行为、持久化或具体 Bridge Event Loop；这些内容属于后续里程碑。
