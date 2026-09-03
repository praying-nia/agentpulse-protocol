# AgentPulse Native Transport v3 / Native 传输协议 v3

## Status and scope / 状态与范围

This document is the canonical contract between the Rust Native Channel and an AgentPulse native client. Native Transport v3 provides authenticated, resumable Session/Event synchronization, atomic form responses, and bounded remote commands.

本文档是 Rust Native Channel 与 AgentPulse 原生客户端之间的权威契约。Native Transport v3 提供带认证、可续传的 Session/Event 同步、原子表单响应与有界远程指令。

Native Transport is separate from the channel-neutral [JSON Wire Protocol v2](wire-v2.md). Control messages and delivery metadata use the envelope defined here; every `domain` value is an unchanged, independently valid JSON Wire v2 envelope.

Native Transport 与 Channel-neutral 的 [JSON 线协议 v2](wire-v2.md)相互独立。控制消息及投递元数据使用本文定义的 Envelope；每个 `domain` 值都是未经改写、可独立校验的 JSON Wire v2 Envelope。

## WebSocket endpoint / WebSocket 端点

The v3 server has the following fixed transport contract:

v3 服务端使用以下固定传输契约：

| Property | Native Transport v3 |
| --- | --- |
| Bind boundary | explicit IPv4/IPv6 loopback, or an explicit private/link-local LAN address with TLS and bearer authorization |
| HTTP path | exactly `/agentpulse/native/v3`, without a query |
| WebSocket subprotocol | required `agentpulse.native.v3` |
| Application messages | UTF-8 WebSocket text messages only |
| Default complete-message limit | 1 MiB (`1048576` bytes) |
| Default outbound queue | 256 complete application messages |
| Default handshake timeout | 5 seconds |
| Default server Ping interval | 15 seconds |
| Default idle timeout | 45 seconds |
| Compression | not negotiated |
| Active clients | exactly one per Native Channel instance |

The operating system assigns the port when port zero is configured. Callers must obtain the actual address from the running Channel and provide it to the client. A wildcard or public address is always invalid. Loopback mode remains available for same-machine clients. LAN mode requires a leaf-and-CA TLS identity plus a live bearer authorizer and accepts only RFC 1918, IPv4 link-local, IPv6 unique-local, or IPv6 link-local bind addresses. A second authorized connection is upgraded only to receive a terminal `connection_busy` error and a policy close; it cannot displace the active connection.

配置端口为零时由操作系统分配实际端口；调用方必须从运行中的 Channel 获取实际地址并交给客户端。Wildcard 与公网地址始终非法。Loopback 模式继续服务同机客户端；LAN 模式必须同时配置 Leaf/CA TLS 身份与实时 Bearer Authorizer，并且只能绑定 RFC 1918、IPv4 Link-local、IPv6 Unique-local 或 IPv6 Link-local 地址。第二条已认证连接只能在 Upgrade 后接收终止性的 `connection_busy` 错误与 Policy Close，不能抢占当前连接。

An authenticated LAN upgrade must use the exact path and subprotocol above and include both `Authorization: Bearer <per-device-token>` and `X-AgentPulse-Client-Id: <UUIDv7>`. The authenticated client ID must equal the subsequent `client_hello.client_id`. Authorization is rechecked during the live connection, so Host-side revocation terminates an already connected device. TLS clients validate the stable Host DNS name against the app-scoped CA learned through [Pairing v1](pairing-v1.md); pairing itself initially pins the advertised leaf SHA-256.

通过 LAN 建立连接时，Upgrade 必须使用上述精确 Path 与 Subprotocol，并同时携带 `Authorization: Bearer <per-device-token>` 和 `X-AgentPulse-Client-Id: <UUIDv7>`。已认证 Client ID 必须与后续 `client_hello.client_id` 完全一致。活动连接期间会持续复核授权，因此 Host 端撤销设备后，已连接设备也会断开。TLS Client 使用 [Pairing v1](pairing-v1.md) 获取的应用内 CA 校验稳定 Host DNS Name；配对连接首次使用公布的 Leaf SHA-256 Pin。

Binary application messages are protocol errors. WebSocket Ping, Pong, Close and fragmentation are transport details: the application codec receives only complete text messages. A message over the configured limit terminates that connection. Queue overflow also terminates the slow client instead of silently dropping, reordering, or retaining an unbounded backlog.

Binary 应用消息属于协议错误。WebSocket Ping、Pong、Close 与分片属于 Transport 细节：应用编解码器只接收完整 Text Message。超过配置上限的消息会终止该连接；输出队列溢出也会终止慢客户端，不会静默丢弃、乱序或无限积压。

## Envelope and strict compatibility / Envelope 与严格兼容

Every Native control or delivery message has this outer envelope:

每条 Native 控制或投递消息均使用以下外层 Envelope：

```json
{
  "native_transport_version": 3,
  "message": {
    "type": "discover_sessions",
    "request_id": "01890f47-7c00-7000-8000-000000000005"
  }
}
```

- `native_transport_version` is the JSON unsigned integer `3`; all other values are rejected before message decoding.
- Every Native object rejects unknown fields, message types, enum values, and malformed scalar types.
- Client, connection, and request identities are canonical lowercase UUIDv7 strings.
- Event cursors are canonical unsigned decimal strings and must satisfy the domain `EventSequence` invariant.
- Optional values are omitted by canonical encoders when absent; decoders also accept JSON `null` for optional fields.
- Native v3 selects only AgentPulse JSON Wire protocol version `2`.

- `native_transport_version` 是 JSON 无符号整数 `3`；其他值在解码消息前即被拒绝。
- 所有 Native Object 都拒绝未知字段、消息类型、枚举值及错误 Scalar 类型。
- Client、Connection 与 Request ID 均为规范小写 UUIDv7 字符串。
- Event Cursor 使用规范无符号十进制字符串，并且必须满足领域 `EventSequence` 不变量。
- Optional 值为空时由规范编码器省略；解码器也接受 JSON `null`。
- Native v3 只选择 AgentPulse JSON Wire 协议版本 `2`。

Any field, message, or behavior added to this strict contract requires a new Native Transport version. Supporting a future domain protocol version does not implicitly change the Native Transport version.

严格契约中的任何字段、消息或行为变更都必须升级 Native Transport 版本。未来支持新的领域协议版本并不隐式改变 Native Transport 版本。

## Client messages / 客户端消息

### `client_hello`

The first application message must be exactly one `client_hello`:

第一条应用消息必须是且只能是一次 `client_hello`：

```json
{
  "type": "client_hello",
  "client_id": "01890f47-7c00-7000-8000-000000000004",
  "display_name": "Fixture Native Client",
  "version": "1.0.0",
  "supported_protocol_versions": [2],
  "host_run_id": "01890f47-7c00-7000-8000-000000000011",
  "session_cursors": [
    { "session_id": "01890f47-7c00-7000-8000-000000000003", "last_sequence": "7" }
  ]
}
```

`display_name` and an included `version` must be nonblank. The protocol-version array must be nonempty and unique and must contain `2`. `host_run_id` identifies the Host run represented by the client's in-memory cache; its Session cursors are positive, unique, contiguous last-applied sequences. Missing cached state omits both fields. A missing, repeated, late, or incompatible Hello is fatal.

`display_name` 与存在的 `version` 必须非空白；协议版本数组必须非空且不重复，并包含 `2`。`host_run_id` 标识客户端内存缓存所属的 Host 运行周期，Session Cursor 是正数、唯一且连续应用的最后序号；没有缓存时两者均省略。缺失、重复、迟到或不兼容的 Hello 都会终止连接。

### `discover_sessions`

```json
{
  "type": "discover_sessions",
  "request_id": "01890f47-7c00-7000-8000-000000000005"
}
```

Discovery is allowed after Hello whenever no subscription baseline is currently being synchronized. Active subscriptions remain live during a refresh. A completed discovery replaces that connection's eligible Session set; clients subscribe only newly discovered Sessions and preserve the cursor, Events, and pending interactions of existing subscriptions.

Discovery 可在 Hello 完成后、且当前没有 Subscription Baseline 正在同步时发起。刷新期间已有订阅保持实时投递。完成的新 Discovery 会替换本连接可订阅的 Session 集合；客户端只订阅新发现的 Session，并保留已有订阅的 Cursor、Event 与 Pending Interaction。

### `subscribe_session`

```json
{
  "type": "subscribe_session",
  "request_id": "01890f47-7c00-7000-8000-000000000005",
  "session_id": "01890f47-7c00-7000-8000-000000000003"
}
```

The Session must be present in the latest completed discovery snapshot for this connection. Subscriptions are idempotent. Only one subscription may be in its baseline synchronization phase at a time; already-active subscriptions remain live.

目标 Session 必须存在于本连接最近一次完成的 Discovery Snapshot 中。订阅操作幂等；同一时刻只能有一个订阅处于 Baseline 同步阶段，其他已建立订阅仍保持实时投递。

### `unsubscribe_session`

```json
{
  "type": "unsubscribe_session",
  "request_id": "01890f47-7c00-7000-8000-000000000005",
  "session_id": "01890f47-7c00-7000-8000-000000000003"
}
```

Unsubscription is idempotent. Once its result is queued, later Events for that Session are not eligible for this client.

取消订阅是幂等操作。结果进入发送队列后，该 Session 的后续 Event 不再属于本客户端的投递目标。

### `submit_interaction_response`

An actively subscribed client may submit one nested JSON Wire v2 `interaction_response`:

```json
{
  "type": "submit_interaction_response",
  "request_id": "01890f47-7c00-7000-8000-000000000005",
  "response": {
    "protocol_version": 2,
    "message": {
      "type": "interaction_response",
      "payload": {
        "request_id": "01890f47-7c00-7000-8000-000000000009",
        "session_id": "01890f47-7c00-7000-8000-000000000003",
        "channel_id": "01890f47-7c00-7000-8000-000000000002",
        "responded_at": "2026-08-29T00:04:00Z",
        "payload": { "type": "approval", "option_id": "01890f47-7c00-7000-8000-00000000000a" }
      }
    }
  }
}
```

The nested Channel ID must identify this Native Channel, the Session must be actively subscribed, and the interaction must still be pending. Approval and choice IDs must belong to the request. Forms submit every field atomically; secret values are forwarded once and then discarded by the Provider runtime.

已活动订阅的客户端可以提交一条嵌套 JSON Wire v2 `interaction_response`。其中 Channel ID 必须是当前 Native Channel，Session 必须已订阅，Interaction 仍须 pending；审批和选择 ID 必须来自该请求。表单一次提交全部字段；敏感值只向 Provider 转发一次，随后即从运行时丢弃。

### `submit_command`

An actively subscribed client may submit one nested JSON Wire v2 `agent_command` whose Channel and Session identify the current Native route:

```json
{
  "type": "submit_command",
  "request_id": "01890f47-7c00-7000-8000-000000000005",
  "command": {
    "protocol_version": 2,
    "message": {
      "type": "agent_command",
      "payload": {
        "id": "01890f47-7c00-7000-8000-00000000000b",
        "session_id": "01890f47-7c00-7000-8000-000000000003",
        "channel_id": "01890f47-7c00-7000-8000-000000000002",
        "issued_at": "2026-09-03T00:00:00Z",
        "payload": { "type": "status" }
      }
    }
  }
}
```

Commands are accepted only for an active subscription and a complete Provider/Channel capability route. `command_result` confirms bounded in-memory handoff, not completion of the underlying App Server request.

## Server messages / 服务端消息

### `server_hello`

The server replies to a valid Client Hello with its UUIDv7 `connection_id`, a nested JSON v2 `channel_descriptor`, selected domain protocol, and effective transport limits. The descriptor kind is exactly `native`; its capabilities are exactly `notification`, `session_view`, `approval`, `text_input`, `form_input`, `realtime_sync`, and `remote_command`.

服务端用 `server_hello` 响应有效 Client Hello，其中包含 UUIDv7 `connection_id`、嵌套的 JSON v2 `channel_descriptor`、选定的领域协议及实际 Transport 限制。Descriptor Kind 固定为 `native`，Capability 精确包含 `notification`、`session_view`、`approval`、`text_input`、`form_input`、`realtime_sync` 与 `remote_command`。

The Hello additionally returns the UUIDv7 `host_run_id` and `resume_accepted`. One run ID is created for the in-memory Host lifetime and survives WebSocket, Relay, and RuntimeHost source restarts. A new Host process creates a new ID. When the supplied ID differs, all supplied cursors are ignored and `resume_accepted` is false.

Hello 还返回 UUIDv7 `host_run_id` 与 `resume_accepted`。运行 ID 对应一次内存 Host 生命周期，WebSocket、Relay 与 RuntimeHost Source 重启不会改变它；新 Host 进程会创建新 ID。客户端提交的 ID 不同时，全部 Cursor 均被忽略，并返回 `resume_accepted = false`。

Native v3 intentionally does not declare the standalone `choice_input` capability; Codex Plan-mode questions use atomic `form_input` instead.

Native v3 有意不声明独立的 `choice_input` Capability；Codex Plan 模式问题统一使用原子 `form_input`。

### Discovery batch

A successful discovery is one contiguous, ordered batch:

成功的 Discovery 是一个连续、有序的 Batch：

```text
sync_started(request_id, provider_count, session_count)
domain_message(discovery_provider, provider_descriptor) × provider_count
domain_message(discovery_session, agent_session, last_sequence) × session_count
sync_completed(request_id)
```

Providers and Sessions are ordered by their strongly typed IDs. Counts describe only the frames between the matching start and completion. The client must stage the batch and commit it only after the matching `sync_completed`; disconnection or an error before completion leaves the previous client view intact.

Provider 与 Session 按强类型 ID 排序。Count 只描述对应 Start 与 Completion 之间的 Frame。客户端必须暂存整个 Batch，并仅在收到匹配的 `sync_completed` 后提交；若在完成前断线或失败，应保留此前的客户端视图。

`last_sequence` is the exact Event cursor represented by the included Session Aggregate at snapshot time. Discovery does not subscribe, replay history, or reserve that cursor.

`last_sequence` 是 Snapshot 时所含 Session Aggregate 已表示的精确 Event Cursor。Discovery 不建立订阅、不重放历史，也不锁定该 Cursor。

### Paged catch-up and live baseline

New and resumed subscriptions replay retained Events in pages of at most 128 Events. A non-final page is:

新订阅与恢复订阅都以最多 128 条 Event 的页面重放内存历史。非最终页顺序为：

```text
subscription_result(status = catching_up, baseline_sequence = N, event_count = K)
domain_message(live_event, agent_event) × K
sync_completed(request_id)
```

The client stages each page and advances its cursor only after the matching `sync_completed`, then repeats `subscribe_session`. The final page uses `status = subscribed`, includes retained Events followed by the current Session and pending-interaction baseline, and atomically establishes live delivery. Events racing with final setup are buffered after `sync_completed`, so no Event after `N` can overtake it. `reset = true` requires discarding prior state for that Session before applying the page.

客户端暂存每一页，只在收到匹配的 `sync_completed` 后推进 Cursor，然后再次发送 `subscribe_session`。最终页使用 `status = subscribed`，先携带保留 Event，再携带当前 Session 与 Pending Interaction Baseline，并原子建立实时投递。与最终建立过程并发的 Event 会缓存在 `sync_completed` 之后，因此 `N` 之后的 Event 不会越过它。`reset = true` 要求应用该页前丢弃对应 Session 的旧状态。

Catch-up Events restore in-app history but do not replay historical user notifications. When the client first enters live mode, it may notify once for each interaction that is still pending; later live Events follow the normal notification policy.

Catch-up Event 用于恢复 App 内历史，不重放历史系统通知。客户端首次进入 Live 时，可对仍然 Pending 的每个 Interaction 各通知一次；此后的 Live Event 使用正常通知策略。

A duplicate active subscription returns `already_subscribed` with the current cursor and does not resend the baseline. `unsubscription_result` reports either `unsubscribed` or `not_subscribed`.

重复订阅活动 Session 时返回 `already_subscribed` 及当前 Cursor，不重复发送 Baseline。`unsubscription_result` 返回 `unsubscribed` 或 `not_subscribed`。

### Domain delivery contexts

`domain_message` contains `context` plus one unchanged JSON v2 envelope in `domain`. Context and domain type must match:

`domain_message` 包含 `context`，以及 `domain` 中一条未经改写的 JSON v2 Envelope。Context 与领域消息类型必须匹配：

| Context type | Required domain type | Meaning |
| --- | --- | --- |
| `discovery_provider` | `provider_descriptor` | Provider in a discovery batch |
| `discovery_session` | `agent_session` | Session in a discovery batch, with `last_sequence` |
| `subscription_session` | `agent_session` | Baseline for the matching subscription request |
| `subscription_interaction` | `interaction_request` | One pending request in that same baseline, with its route |
| `live_event` | `agent_event` | Event after the subscription cursor |
| `live_session` | `agent_session` | Updated view after a state-changing live Event |

A live Event or subscription-interaction context carries `route: observe_only | interaction_read_only | interaction_interactive`. Controls may be exposed only for `interaction_interactive`; the client must submit exactly one Provider-issued opaque option ID. For state-changing Events, the Event is delivered before its updated Session view.

Live Event 或 Subscription Interaction Context 携带 `route: observe_only | interaction_read_only | interaction_interactive`。只有 `interaction_interactive` 可展示控件，客户端必须提交 Provider 给出的一个不透明 Option ID。对于改变状态的 Event，Event 必须先于更新后的 Session View 投递。

After the Bridge and Provider accept a submission, the server returns `interaction_response_result(request_id, session_id, interaction_id)`. This confirms handoff only. The pending request closes solely through a later domain `InteractionResponded`, `InteractionClosed`, or owning lifecycle event.

Bridge 与 Provider 接受提交后，服务端返回 `interaction_response_result(request_id, session_id, interaction_id)`；它只确认交接成功。Pending 请求只能由后续领域 `InteractionResponded`、`InteractionClosed` 或所属生命周期事件关闭。

After a command is accepted into the Provider's bounded process-memory queue, the server returns `command_result(request_id, session_id, command_id)`. User, assistant, and system messages then arrive through ordinary live domain Events; a command-specific failure is reported as a system message.

指令进入 Provider 的有界进程内队列后，服务端返回 `command_result(request_id, session_id, command_id)`。后续用户、助手和系统消息仍通过普通实时领域 Event 到达；具体指令失败会以系统消息报告。

## Errors / 错误

An error contains an optional request ID, a stable code, a nonblank diagnostic, and `recoverable`:

Error 包含可选 Request ID、稳定 Code、非空 Diagnostic 与 `recoverable`：

| Code | Typical meaning | Recoverable |
| --- | --- | --- |
| `connection_busy` | another client owns the endpoint | no |
| `invalid_handshake` | Hello is missing, repeated, timed out, or incompatible | no |
| `invalid_request` | invalid frame or state-machine operation | depends on violation |
| `session_not_discovered` | subscribe target absent from latest snapshot | yes |
| `session_not_found` | target disappeared before Bridge subscription | yes |
| `internal` | runtime handoff or encoding could not complete | normally yes |
| `capability_unavailable` | the complete Provider/Channel approval route is unavailable | yes |
| `interaction_not_pending` | the request was already resolved or closed | yes |
| `session_not_subscribed` | this client does not own the target subscription | yes |
| `provider_rejected` | the Provider rejected the handoff | yes |

`recoverable: true` means the same connection may issue a later valid request; it does not mean the failed request is automatically retried. A nonrecoverable error is followed by a WebSocket Close. Transport framing, size, or queue failures can close the connection without a request-scoped result if a complete error frame cannot be sent safely.

`recoverable: true` 表示同一连接可以继续发起新的合法请求，并不表示失败请求会自动重试。不可恢复 Error 后紧跟 WebSocket Close。若无法安全发送完整 Error Frame，Transport 分帧、大小或 Queue 错误可以直接关闭连接而不返回 Request-scoped Result。

## Heartbeat, disconnect, and reconnect / 心跳、断线与重连

After Hello, the server sends periodic WebSocket Ping frames. Any valid client frame or Pong refreshes activity. The server closes a connection that remains idle for the advertised timeout. Clients should answer Ping according to RFC 6455 and may reconnect only by creating a new WebSocket and repeating Hello.

Hello 完成后服务端定期发送 WebSocket Ping。任何有效 Client Frame 或 Pong 都会刷新 Activity；超过公布的 Idle Timeout 后服务端关闭连接。客户端应依 RFC 6455 响应 Ping，并且只能通过新建 WebSocket、重新执行 Hello 来重连。

Disconnect immediately clears connection-owned subscriptions, pending pages, and queued frames, but not the Host's in-memory Event streams. Reconnect repeats Hello and discovery. An accepted run ID resumes each Session after the submitted cursor; a different run ID clears client history and replays the new run from sequence one. Neither side persists Session/Event history across process death.

断线会立即清除本连接拥有的 Subscription、Pending Page 与 Queue Frame，但不会清除 Host 的本次运行内存事件流。重连重新执行 Hello 与 Discovery；运行 ID 一致时各 Session 从客户端 Cursor 增量补齐，不同时清除客户端历史并从 Sequence 1 重放新周期。任一端都不跨进程持久化 Session/Event 历史。

Stopping or dropping the Channel Source revokes its RuntimeHost generation before cleanup and removes every generation-scoped subscription. Restart creates a fresh listener address and fresh ingress generation; stale handles and old clients cannot regain access.

停止或释放 Channel Source 时，会先撤销当前 RuntimeHost Generation，再清除全部 Generation-scoped Subscription。重启会创建新的 Listener Address 与 Ingress Generation；旧 Handle 与旧 Client 无法重新取得访问权。

## Security and exclusions / 安全边界与不包含内容

Native v3 has two explicit security boundaries: operating-system loopback isolation for same-machine clients, and authenticated TLS for private-LAN clients. LAN credentials are per device, stored only as hashes on the Host, revocable while active, and provisioned only through [Pairing v1](pairing-v1.md). Native v3 still defines no wildcard/public-network exposure, browser Origin authorization, or direct Internet endpoint. Relay remains an opaque outer tunnel. Implementations must fail closed when the credential store, certificate, client binding, or private-address validation fails.

Native v3 具有两种显式安全边界：同机客户端依赖操作系统 Loopback 隔离，私有 LAN 客户端使用带认证 TLS。LAN 凭证按设备独立签发，Host 仅保存 Hash，可在活动连接期间撤销，并且只能通过 [Pairing v1](pairing-v1.md)下发。Native v3 不定义 Wildcard/直接公网暴露或浏览器 Origin 授权；Relay 继续只是外层不透明隧道。凭证库、证书、Client 绑定或私网地址校验失败时，实现必须 Fail Closed。

This contract excludes persistence, pre-run Provider history, automatic approval retry, multiple concurrent clients, binary input, and database dependencies. Complete current-run in-memory retention, paged replay, discovery, cursor-safe live handoff, atomic form correlation, bounded FIFO prompt queues, typed remote commands, cleanup, bounded failure, and explicit reconnection are required v3 behavior.

本契约不包含持久化、Host 启动前的 Provider 历史、审批自动重试、多并发客户端、二进制输入或数据库依赖。本次运行完整内存保留、分页重放、Discovery、Cursor-safe 实时交接、原子表单关联、有界 FIFO Prompt 队列、类型化远程指令、清理、有界失败与显式重连均是 v3 必须行为。

## Canonical fixtures / 权威 Fixtures

[`fixtures/native-v3`](fixtures/native-v3) contains deterministic examples of all six client messages and every current server message family, including resumable Hello, approval/form baseline, command submission, and response frames. They are the cross-language Golden Fixtures. Whitespace and object-key order are not semantic; values, field presence, array order, and JSON types are semantic.

[`fixtures/native-v3`](fixtures/native-v3) 包含全部六种 Client Message 及当前所有 Server Message Family 的确定性示例，包括可恢复 Hello、审批/表单 Baseline、指令提交与响应 Frame。它们是跨语言 Golden Fixtures。空白与 Object Key 顺序不属于语义；值、字段存在性、Array 顺序与 JSON 类型属于语义。
