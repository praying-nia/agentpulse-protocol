# AgentPulse Native Transport v1 / Native 传输协议 v1

## Status and scope / 状态与范围

This document is the canonical contract between the Rust Native Channel and a local AgentPulse native client. Native Transport v1 is a complete **local, read-only** synchronization path: one client can identify itself, discover current Providers and Sessions, subscribe to selected Sessions, and receive their current view and subsequent normalized Events.

本文档是 Rust Native Channel 与本地 AgentPulse 原生客户端之间的权威契约。Native Transport v1 是一条完整的**本地只读**同步链路：一个客户端可以标识自身、发现当前 Provider 与 Session、订阅指定 Session，并接收其当前视图及后续标准 Event。

Native Transport is separate from the channel-neutral [JSON Wire Protocol v1](wire-v1.md). Control messages and delivery metadata use the envelope defined here; every `domain` value is an unchanged, independently valid JSON Wire v1 envelope.

Native Transport 与 Channel-neutral 的 [JSON 线协议 v1](wire-v1.md)相互独立。控制消息及投递元数据使用本文定义的 Envelope；每个 `domain` 值都是未经改写、可独立校验的 JSON Wire v1 Envelope。

## WebSocket endpoint / WebSocket 端点

The v1 server has the following fixed transport contract:

首版服务端使用以下固定传输契约：

| Property | Native Transport v1 |
| --- | --- |
| Bind boundary | explicit IPv4/IPv6 loopback, or an explicit private/link-local LAN address with TLS and bearer authorization |
| HTTP path | exactly `/agentpulse/native/v1`, without a query |
| WebSocket subprotocol | required `agentpulse.native.v1` |
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
  "native_transport_version": 1,
  "message": {
    "type": "discover_sessions",
    "request_id": "01890f47-7c00-7000-8000-000000000005"
  }
}
```

- `native_transport_version` is the JSON unsigned integer `1`; all other values are rejected before message decoding.
- Every Native object rejects unknown fields, message types, enum values, and malformed scalar types.
- Client, connection, and request identities are canonical lowercase UUIDv7 strings.
- Event cursors are canonical unsigned decimal strings and must satisfy the domain `EventSequence` invariant.
- Optional values are omitted by canonical encoders when absent; decoders also accept JSON `null` for optional fields.
- Native v1 selects only AgentPulse JSON Wire protocol version `1`.

- `native_transport_version` 是 JSON 无符号整数 `1`；其他值在解码消息前即被拒绝。
- 所有 Native Object 都拒绝未知字段、消息类型、枚举值及错误 Scalar 类型。
- Client、Connection 与 Request ID 均为规范小写 UUIDv7 字符串。
- Event Cursor 使用规范无符号十进制字符串，并且必须满足领域 `EventSequence` 不变量。
- Optional 值为空时由规范编码器省略；解码器也接受 JSON `null`。
- Native v1 只选择 AgentPulse JSON Wire 协议版本 `1`。

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
  "supported_protocol_versions": [1]
}
```

`display_name` and an included `version` must be nonblank. The protocol-version array must be nonempty and unique and must contain `1`. A missing, repeated, late, or incompatible Hello is fatal.

`display_name` 与存在的 `version` 必须非空白；协议版本数组必须非空且不重复，并包含 `1`。缺失、重复、迟到或不兼容的 Hello 都会终止连接。

### `discover_sessions`

```json
{
  "type": "discover_sessions",
  "request_id": "01890f47-7c00-7000-8000-000000000005"
}
```

Discovery is allowed only after Hello and while the client has no active or pending subscriptions. A new discovery replaces that connection's eligible Session set. Clients must unsubscribe from every Session before refreshing discovery.

Discovery 只能在 Hello 完成后、且客户端没有活动或同步中的订阅时发起。新的 Discovery 会替换本连接可订阅的 Session 集合；刷新 Discovery 前必须先取消全部 Session 订阅。

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

There is deliberately no Client Action message in v1. Approval, text/choice input, prompt submission, cancellation, or any other write-back cannot be sent over this protocol.

v1 有意不定义任何 Client Action 消息；审批、文本/选择输入、Prompt 提交、取消及其他写回操作都不能通过本协议发送。

## Server messages / 服务端消息

### `server_hello`

The server replies to a valid Client Hello with its UUIDv7 `connection_id`, a nested JSON v1 `channel_descriptor`, selected domain protocol, and effective transport limits. The descriptor kind is exactly `native`; its capabilities are exactly `notification`, `session_view`, and `realtime_sync`.

服务端用 `server_hello` 响应有效 Client Hello，其中包含 UUIDv7 `connection_id`、嵌套的 JSON v1 `channel_descriptor`、选定的领域协议及实际 Transport 限制。Descriptor Kind 固定为 `native`，Capability 只包含 `notification`、`session_view` 与 `realtime_sync`。

The three capabilities are an exact read-only promise, not a placeholder. In particular, Native v1 does not declare `approval`, `choice_input`, `text_input`, `form_input`, or `remote_command`.

这三个 Capability 是精确的只读承诺，而非占位声明；Native v1 不声明 `approval`、`choice_input`、`text_input`、`form_input` 或 `remote_command`。

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

### Subscription result and baseline

For a new subscription, delivery order is:

新订阅的投递顺序固定为：

```text
subscription_result(status = subscribed, baseline_sequence = N)
domain_message(subscription_session, agent_session representing N)
live domain messages with sequence > N
```

The Bridge establishes the subscription and captures the current Aggregate cursor atomically with baseline delivery. Events that race with setup are buffered behind the result and baseline, so no Event after `N` can overtake them. Historical Events at or below `N` are not replayed.

Bridge 在交付 Baseline 时建立订阅并捕获当前 Aggregate Cursor。与建立过程并发的 Event 会缓存在 Result 与 Baseline 后方，因此 `N` 之后的 Event 不会越过它们；`N` 及以前的历史 Event 不会重放。

A duplicate active subscription returns `already_subscribed` with the current cursor and does not resend the baseline. `unsubscription_result` reports either `unsubscribed` or `not_subscribed`.

重复订阅活动 Session 时返回 `already_subscribed` 及当前 Cursor，不重复发送 Baseline。`unsubscription_result` 返回 `unsubscribed` 或 `not_subscribed`。

### Domain delivery contexts

`domain_message` contains `context` plus one unchanged JSON v1 envelope in `domain`. Context and domain type must match:

`domain_message` 包含 `context`，以及 `domain` 中一条未经改写的 JSON v1 Envelope。Context 与领域消息类型必须匹配：

| Context type | Required domain type | Meaning |
| --- | --- | --- |
| `discovery_provider` | `provider_descriptor` | Provider in a discovery batch |
| `discovery_session` | `agent_session` | Session in a discovery batch, with `last_sequence` |
| `subscription_session` | `agent_session` | Baseline for the matching subscription request |
| `live_event` | `agent_event` | Event after the subscription cursor |
| `live_session` | `agent_session` | Updated view after a state-changing live Event |

A live Event context carries `route: observe_only | interaction_read_only`. An Interaction Request may be shown as information but must never expose an input control. For state-changing Events, the Event is delivered before its updated Session view.

Live Event Context 携带 `route: observe_only | interaction_read_only`。Interaction Request 可以只读展示，但绝不能暴露输入控件。对于改变状态的 Event，Event 必须先于更新后的 Session View 投递。

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
| `read_only` | an unsupported write operation was attempted | yes |
| `internal` | runtime handoff or encoding could not complete | normally yes |

`recoverable: true` means the same connection may issue a later valid request; it does not mean the failed request is automatically retried. A nonrecoverable error is followed by a WebSocket Close. Transport framing, size, or queue failures can close the connection without a request-scoped result if a complete error frame cannot be sent safely.

`recoverable: true` 表示同一连接可以继续发起新的合法请求，并不表示失败请求会自动重试。不可恢复 Error 后紧跟 WebSocket Close。若无法安全发送完整 Error Frame，Transport 分帧、大小或 Queue 错误可以直接关闭连接而不返回 Request-scoped Result。

## Heartbeat, disconnect, and reconnect / 心跳、断线与重连

After Hello, the server sends periodic WebSocket Ping frames. Any valid client frame or Pong refreshes activity. The server closes a connection that remains idle for the advertised timeout. Clients should answer Ping according to RFC 6455 and may reconnect only by creating a new WebSocket and repeating Hello.

Hello 完成后服务端定期发送 WebSocket Ping。任何有效 Client Frame 或 Pong 都会刷新 Activity；超过公布的 Idle Timeout 后服务端关闭连接。客户端应依 RFC 6455 响应 Ping，并且只能通过新建 WebSocket、重新执行 Hello 来重连。

Disconnect immediately clears all subscriptions owned by that connection, pending baselines, and queued frames. Reconnect does not resume implicit state: the client must perform a fresh discovery and explicitly subscribe again. The new baseline cursor is authoritative, so the client replaces its Session view and then accepts only later live Events.

断线会立即清除本连接拥有的全部订阅、Pending Baseline 与 Queue Frame。重连不会隐式恢复状态：客户端必须重新 Discovery 并显式 Subscribe。新的 Baseline Cursor 是权威边界，客户端应替换 Session View，此后只接收更晚的 Live Event。

Stopping or dropping the Channel Source revokes its RuntimeHost generation before cleanup and removes every generation-scoped subscription. Restart creates a fresh listener address and fresh ingress generation; stale handles and old clients cannot regain access.

停止或释放 Channel Source 时，会先撤销当前 RuntimeHost Generation，再清除全部 Generation-scoped Subscription。重启会创建新的 Listener Address 与 Ingress Generation；旧 Handle 与旧 Client 无法重新取得访问权。

## Security and exclusions / 安全边界与不包含内容

Native v1 has two explicit security boundaries: operating-system loopback isolation for same-machine clients, and authenticated TLS for private-LAN clients. LAN credentials are per device, stored only as hashes on the Host, revocable while active, and provisioned only through [Pairing v1](pairing-v1.md). Native v1 still defines no wildcard/public-network exposure, browser Origin authorization, Internet endpoint, or Relay tunneling. Implementations must fail closed when the credential store, certificate, client binding, or private-address validation fails.

Native v1 具有两种显式安全边界：同机客户端依赖操作系统 Loopback 隔离，私有 LAN 客户端使用带认证 TLS。LAN 凭证按设备独立签发，Host 仅保存 Hash，可在活动连接期间撤销，并且只能通过 [Pairing v1](pairing-v1.md)下发。Native v1 仍不定义 Wildcard/公网暴露、浏览器 Origin 授权、Internet Endpoint 或 Relay Tunnel。凭证库、证书、Client 绑定或私网地址校验失败时，实现必须 Fail Closed。

This contract also excludes persistence, historical Event replay, offline queues, acknowledgements, automatic retry, multiple concurrent clients, Native UI behavior, Provider write-back, and database dependencies. Those exclusions do not weaken the local read-only path: discovery, cursor-safe baseline synchronization, live delivery, cleanup, bounded failure, and explicit reconnection are all required v1 behavior.

本契约也不包含持久化、历史 Event 重放、离线 Queue、ACK、自动重试、多并发客户端、Native UI 行为、Provider 写回及数据库依赖。这些排除项不削弱本地只读链路：Discovery、Cursor-safe Baseline 同步、Live Delivery、清理、有界失败与显式重连都是 v1 必须行为。

## Canonical fixtures / 权威 Fixtures

[`fixtures/native-v1`](fixtures/native-v1) contains deterministic examples of all four client messages and every current server message family, including a nested discovery Session. They are the cross-language Golden Fixtures. Whitespace and object-key order are not semantic; values, field presence, array order, and JSON types are semantic.

[`fixtures/native-v1`](fixtures/native-v1) 包含全部四种 Client Message 及当前所有 Server Message Family 的确定性示例，包括嵌套 Discovery Session。它们是跨语言 Golden Fixtures。空白与 Object Key 顺序不属于语义；值、字段存在性、Array 顺序与 JSON 类型属于语义。
