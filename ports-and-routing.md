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

## Multi-endpoint Bridge orchestration / Bridge 多端点编排

The Bridge is one synchronous, in-memory orchestration service containing any number of explicitly registered heterogeneous Provider and Channel Ports. Registration snapshots each Descriptor and rejects another endpoint of the same strongly typed ID. A Session is created only from a sequence-one `SessionStarted` Event and remains owned by the Provider named by that initial Session.

Bridge 是一个同步内存编排服务，可以显式注册任意数量、具体实现不同的 Provider 与 Channel Port。注册时会保存 Descriptor 快照，并拒绝同一强类型 ID 的重复端点。Session 只能由 sequence 为 1 的 `SessionStarted` Event 创建，并始终归属于初始 Session 标识的 Provider。

A Channel receives a Session only through an explicit Session-to-Channel subscription. Both endpoints must already exist. When the Channel declares `SESSION_VIEW`, subscription first hands off the current `AgentSession` view and becomes active only after that handoff succeeds. A Channel without `SESSION_VIEW` subscribes directly to future Events. Repeating either subscribe or unsubscribe is idempotent. Subscription does not replay historical Events, and a newly started Session has no implicit subscribers.

Channel 只有通过显式 Session–Channel 订阅才能接收该 Session。订阅时双方必须已经存在。若 Channel 声明 `SESSION_VIEW`，Bridge 会先交付当前 `AgentSession` 视图，且只有交接成功后订阅才生效；不支持 `SESSION_VIEW` 的 Channel 直接从后续 Event 开始接收。重复订阅与取消订阅均为幂等操作。订阅不会重放历史 Event，新创建的 Session 也没有隐式订阅者。

For a Provider Event, the Bridge:

1. resolves the registered publishing Provider;
2. resolves the current Session or validates a sequence-one `SessionStarted` Event;
3. validates Provider ownership and publication capability;
4. computes a centralized route for every subscribed Channel before state mutation;
5. creates or advances the Session Aggregate exactly once; and
6. fans the applied Event out in stable Channel-ID order, followed by the latest Session view for state-changing Events when each target declares `SESSION_VIEW`.

Provider Event 进入 Bridge 后依次执行：

1. 解析已经注册的发布方 Provider；
2. 解析当前 Session，或校验 sequence 为 1 的 `SessionStarted` Event；
3. 校验 Provider 归属与发布能力；
4. 在修改状态前，为每个订阅 Channel 计算集中 Route；
5. 只创建或推进一次 Session Aggregate；
6. 按 Channel ID 稳定顺序扇出已应用 Event；对会改变 Session 视图的 Event，再向声明 `SESSION_VIEW` 的目标交付最新视图。

Each subscribed Channel has an independent delivery result. An Event handoff failure skips the Session-view handoff for that target but does not prevent later targets from being attempted. A Session-view failure is reported after the Event was accepted. Any partial failure returns the complete ordered target report, including successful targets. Once reduction succeeds, neither a handoff failure nor a partial fan-out rolls back the Aggregate.

每个订阅 Channel 都有独立的交付结果。若某目标的 Event 交接失败，则跳过该目标的 Session 视图，但不阻止后续目标继续交付；若 Session 视图失败，则 Event 已经由该目标接收。发生部分失败时，Bridge 返回包含成功与失败目标的完整有序报告。Reducer 成功后，任何交接失败或部分扇出都不会回滚 Aggregate。

An exact retry of the Aggregate's latest Event is accepted as already applied and produces no delivery attempts. Capability or reduction failures cause neither state mutation nor Channel handoff. This contract does not automatically retry failed targets.

Aggregate 最新 Event 的精确重试按已应用处理，不触发任何交付。Capability 或 Reducer 失败不会修改状态，也不会触发 Channel 交接。本契约不会自动重试失败目标。

For a Channel Action, the Bridge resolves the registered source Channel, requires its active subscription to the target Session, resolves that Session's owning Provider, and revalidates the complete capability route. An Interaction Response additionally must match a currently pending Request. Only then is the Action handed to the owning Provider. A successful handoff does not mutate the Aggregate or synthesize an Event: the Provider remains responsible for publishing normalized `InteractionResponded` or `CommandIssued` confirmation Events.

Channel Action 进入 Bridge 后，Bridge 会解析已注册的来源 Channel，要求它仍订阅目标 Session，再解析该 Session 所属的 Provider，并重新校验完整 Capability 链路。Interaction Response 还必须匹配当前待处理 Request，之后才能交给所属 Provider。交接成功不会直接修改 Aggregate 或合成 Event；Provider 仍负责发布标准化 `InteractionResponded` 或 `CommandIssued` 确认 Event。

## Runtime Host and Adapter lifecycle / Runtime Host 与 Adapter 生命周期

The Runtime Host owns one Bridge plus an explicitly registered execution Source for every Provider and Channel Port. Registration pairs a Provider Port with a Provider Event Source, or a Channel Port with a Channel Action Source, and is accepted only while the Host is stopped. The Bridge owns the Port half and the Host owns the Source half. The two halves may share Adapter-private state, but neither Source receives or owns the Bridge itself.

Runtime Host 拥有一个 Bridge，并为每个 Provider/Channel Port 显式拥有对应的执行 Source。注册时必须成对提供 Provider Port 与 Provider Event Source，或 Channel Port 与 Channel Action Source，而且只有 Host 处于停止态时才允许注册。Bridge 拥有 Port 半部，Host 拥有 Source 半部；两者可以共享 Adapter 私有状态，但 Source 不会直接接收或拥有 Bridge。

Every start cycle supplies each Source with a fresh, identity-bound ingress handle. A Provider Event handle can publish only for its registered Provider. A Channel handle can obtain a stable discovery snapshot, subscribe or unsubscribe its registered Channel, and submit supported Actions only for that Channel. Discovery contains ordered Provider descriptors and current Session views paired with the exact latest Event cursor represented by each Aggregate. Handles hold a weak Host reference, can be cloned and moved to Adapter workers, and retain the synchronous handoff semantics of the independent Sink ports. Bridge access from different threads is serialized. Synchronous re-entry from a Port callback on the thread already processing that Bridge is rejected as a structured ingress error instead of deadlocking. The Host does not add buffering or automatic retry.

每次启动周期都会向各 Source 提供一个全新且绑定端点身份的入口句柄。Provider Event 句柄只能代表其注册 Provider 发布；Channel 句柄可以取得稳定 Discovery Snapshot、为其注册 Channel 建立或取消订阅，并且只能代表该 Channel 提交受支持 Action。Discovery 包含有序 Provider Descriptor，以及与每个 Aggregate 已表示的精确最新 Event Cursor 配对的当前 Session View。句柄弱引用 Host，可以克隆并移动到 Adapter Worker，同时保留独立 Sink Port 的同步交接语义。不同线程对 Bridge 的访问会被串行化；若 Port 回调在当前正在处理该 Bridge 的同一线程同步重入，则以结构化入口错误拒绝，避免死锁。Host 不增加缓冲或自动重试。

Host start and stop are explicit and idempotent:

- Start attempts Sources in registration order. A repeated start reports `AlreadyStarted` without invoking an Adapter again.
- One start failure revokes only that Source's current handle and does not roll back successfully started Sources or any Event already accepted into the Bridge. Because a failed start may have acquired resources, a complete stop is required before another start.
- Stop revokes every current ingress before invoking any Adapter, then attempts Sources in reverse registration order. One stop failure does not prevent the remaining Sources from being attempted. The Host enters `StopFailed`, and another stop retries only the Sources that did not stop successfully.
- A Source declares whether its subscriptions are persistent or scoped to one Source generation. Before stopping a generation-scoped Channel Source, the Host removes every subscription owned by that Channel. A successful stop otherwise retains registered Ports, Session Aggregates, and persistent subscriptions. A later start creates new handles; handles from every earlier generation remain permanently inactive.
- Dropping a Host without a successful explicit stop performs the same reverse-order cleanup on a best-effort basis. Only explicit stop returns lifecycle failures to the caller, and Source objects are released before the Bridge and its Ports.

Host 的启动与停止是显式且幂等的：

- 启动按注册顺序尝试 Source；重复启动返回 `AlreadyStarted`，不会再次调用 Adapter。
- 单个 Source 启动失败只会撤销该 Source 当前周期的句柄，不回滚已经成功启动的 Source，也不回滚已经由 Bridge 接收的 Event。由于失败的启动可能已经获取资源，再次启动前必须先完成一次完整停止。
- 停止会先撤销全部当前入口，再按注册逆序尝试 Source。一个停止失败不会阻止其余 Source；Host 进入 `StopFailed`，再次停止时只重试尚未成功停止的 Source。
- Source 会声明其订阅是持久的，还是只属于一个 Source Generation。停止 Generation-scoped Channel Source 前，Host 会先移除该 Channel 拥有的全部订阅；除此之外，成功停止仍保留已注册 Port、Session Aggregate 与 Persistent Subscription。后续启动创建新句柄，此前所有周期的旧句柄永久保持失效。
- 若 Host 未成功显式停止便被释放，会尽力执行同样的反序清理。只有显式停止会向调用方返回生命周期失败，且 Source 对象会先于 Bridge 及其 Port 释放。

Lifecycle failures identify the endpoint and start/stop phase and preserve the Adapter error as the error source. A Host-level report contains every attempted Adapter in callback order, so partial success and failure remain observable. Lifecycle governs Source execution and Source-to-Bridge ingress only: stopping does not unregister Ports, clear Bridge state, replay Event history, or automatically resynchronize a Channel. Generation-scoped subscription cleanup is the sole explicit exception and prevents a stopped connection-oriented Source from retaining invisible delivery membership.

生命周期失败会标识端点与启动/停止阶段，并把 Adapter 错误保留为错误源。Host 级报告按照回调顺序包含每个尝试过的 Adapter，因此部分成功与失败均可观察。生命周期只管理 Source 执行与 Source 到 Bridge 的入口；停止不会注销 Port、清空 Bridge 状态、重放 Event 历史或自动重新同步 Channel。Generation-scoped Subscription 清理是唯一显式例外，用于防止已经停止的 Connection-oriented Source 继续保留不可见的投递关系。

## Exclusions / 不包含内容

These generic contracts do not define adapter discovery, endpoint removal, subscription persistence across process lifetime or authorization outside the active routing membership, historical replay, buffering, retries, acknowledgements, backpressure, transport, authentication, Relay behavior, or persistence. A concrete Transport may define its own bounded synchronization and reconnect behavior; see [Native Transport v3](native-transport-v3.md).

这些通用契约不定义 Adapter 发现、端点移除、跨进程生命周期的订阅持久化或活动路由关系之外的鉴权、历史重放、缓冲、重试、ACK、背压、Transport、认证、Relay 行为或持久化。具体 Transport 可以定义自身的有界同步与重连行为；参见 [Native Transport v3](native-transport-v3.md)。
