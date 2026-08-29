# AgentPulse JSON Wire Protocol v1 / JSON 线协议 v1

## Status and scope / 状态与范围

This document is the canonical wire contract for AgentPulse protocol version 1. It maps the channel-neutral semantics in [Unified Domain Model](domain-model.md) to JSON without exposing Rust implementation details.

本文档是 AgentPulse 协议版本 1 的权威线格式约定。它将[统一领域模型](domain-model.md)中的 Channel-neutral 语义映射为 JSON，不暴露 Rust 实现细节。

Version 1 carries six top-level message types:

版本 1 包含六种顶层消息：

```text
provider_descriptor
channel_descriptor
agent_session
agent_event
interaction_response
agent_command
```

Handshake, version negotiation, acknowledgements, delivery metadata, Session lists, Aggregate synchronization, framing, compression, authentication, and transport size limits are outside this domain protocol. A Transport may impose its own envelope and size limit around an entire JSON v1 envelope. The first such contract is [Native Transport v1](native-transport-v1.md), which nests these domain envelopes unchanged.

握手、版本协商、ACK、投递元数据、Session 列表、Aggregate 同步、分帧、压缩、鉴权及 Transport 大小限制不属于本领域协议。Transport 可以在完整 JSON v1 Envelope 外增加自己的 Envelope 与大小限制；首个此类契约是 [Native Transport v1](native-transport-v1.md)，其中的领域 Envelope 保持不变。

## Envelope and compatibility / Envelope 与兼容性

Every message uses this envelope:

每条消息均使用以下 Envelope：

```json
{
  "protocol_version": 1,
  "message": {
    "type": "agent_event",
    "payload": {}
  }
}
```

- `protocol_version` is the JSON integer `1`. Other values are rejected before message decoding.
- Every object rejects unknown fields. Every message type, tagged Variant, enum value, and Capability rejects unknown values.
- Any schema addition or behavioral change requires a new protocol version; v1 has no same-version additive extension mechanism.
- Optional fields are omitted by canonical encoders when absent. Decoders accept either omission or JSON `null`.
- Field names and wire enum values use lowercase `snake_case`.

- `protocol_version` 是 JSON 整数 `1`；其他值会在消息解码前被拒绝。
- 所有对象拒绝未知字段；消息类型、Tagged Variant、枚举值与 Capability 均拒绝未知值。
- 任何 Schema 增补或行为变化都必须升级协议版本；v1 不提供同版本扩展机制。
- 可选字段为空时由规范编码器省略；解码器同时接受省略或 JSON `null`。
- 字段名和线协议枚举值使用小写 `snake_case`。

## Scalar values / 标量值

| Domain value | JSON v1 representation |
| --- | --- |
| Protocol version | JSON unsigned integer |
| AgentPulse typed ID | UUIDv7 string |
| External ID, Kind, and text | JSON string; Core validation still applies |
| Timestamp | RFC 3339 string; encoder normalizes to UTC with `Z` |
| Revision, Event Sequence, Progress completed/total | Canonical unsigned decimal string |
| Boolean | JSON boolean |

Unsigned decimal strings contain only ASCII digits, contain no sign or whitespace, and have no leading zero unless the value is exactly `"0"`. Values must fit in `u64`. Domain rules subsequently reject zero Revision, zero Event Sequence, zero Progress total, and completed Progress greater than total.

无符号十进制字符串只能包含 ASCII 数字，不包含符号或空白；除 `"0"` 外不能有前导零，并且必须能表示为 `u64`。领域规则会继续拒绝零 Revision、零 Event Sequence、零 Progress Total，以及 Completed 大于 Total 的进度。

Decoders accept valid RFC 3339 offsets and normalize them to UTC. Canonical encoding always emits `Z`; fractional seconds are retained when present.

解码器接受合法的 RFC 3339 Offset 并归一化为 UTC；规范编码始终输出 `Z`，存在小数秒时予以保留。

## Capabilities / 能力

Capabilities are JSON arrays of unique strings. Encoders emit the order below; decoders accept any order but reject duplicates.

Capability 使用不重复的字符串数组。编码器按下表顺序输出；解码器接受任意顺序，但拒绝重复项。

Provider Capability mapping:

```text
SESSION_STATE       → session_state
TOOL_EVENTS         → tool_events
PLAN                → plan
PROGRESS            → progress
APPROVAL_REQUEST    → approval_request
APPROVAL_RESPONSE   → approval_response
USER_INPUT_REQUEST  → user_input_request
USER_INPUT_RESPONSE → user_input_response
PROMPT_SUBMIT       → prompt_submit
CANCEL              → cancel
```

Channel Capability mapping:

```text
NOTIFICATION   → notification
SESSION_VIEW   → session_view
TOOL_VIEW      → tool_view
PLAN_VIEW      → plan_view
PROGRESS_VIEW  → progress_view
RICH_MESSAGE   → rich_message
APPROVAL       → approval
CHOICE_INPUT   → choice_input
TEXT_INPUT     → text_input
FORM_INPUT     → form_input
REALTIME_SYNC  → realtime_sync
REMOTE_COMMAND → remote_command
```

## Top-level message payloads / 顶层消息 Payload

### `provider_descriptor`

```text
id: UUIDv7
kind: Provider Kind
display_name: non-empty text
version?: non-empty text
capabilities: Provider Capability[]
```

### `channel_descriptor`

```text
id: UUIDv7
kind: Channel Kind
display_name: non-empty text
version?: non-empty text
capabilities: Channel Capability[]
```

### `agent_session`

```text
id: UUIDv7
provider_id: UUIDv7
external_id?: non-empty opaque string
title?: non-empty text
workspace?: { path: non-empty text, display_name?: non-empty text }
state: initializing | idle | running | waiting_for_interaction | completed | failed | cancelled
connection_state: connected | reconnecting | disconnected
revision: decimal u64 string greater than zero
created_at: RFC 3339
updated_at: RFC 3339 not earlier than created_at
```

### `agent_event`

```text
id: UUIDv7
session_id: UUIDv7
sequence: decimal u64 string greater than zero
occurred_at: RFC 3339
payload: Agent Event Payload
```

### `interaction_response`

```text
request_id: UUIDv7
session_id: UUIDv7
channel_id: UUIDv7
responded_at: RFC 3339
payload: Interaction Response Payload
```

An isolated response cannot be fully validated without its request. Request correlation, expiration, allowed Approval Scope, and Choice membership are checked later by Core/Reducer when the request context is available.

独立的 Response 无法在缺少 Request 时完成全部验证。Request 关联、过期、允许的 Approval Scope 与 Choice 成员关系由 Core/Reducer 在取得 Request 上下文后校验。

### `agent_command`

```text
id: UUIDv7
session_id: UUIDv7
channel_id: UUIDv7
issued_at: RFC 3339
payload: Agent Command Payload
```

## Tagged payloads / Tagged Payload

Every payload below is a JSON object with a required `type` field and exactly the fields listed for that type.

以下 Payload 均为带必填 `type` 字段的 JSON 对象，并且只能包含该类型列出的字段。

### Agent Event Payload

```text
session_started       { session: Agent Session }
state_changed         { state: Agent State }
connection_changed    { connection_state: Connection State }
message               { message: { level, content } }
tool_activity         { activity: Tool Activity }
plan_updated          { plan: Plan Snapshot }
progress_updated      { progress: Progress Snapshot }
interaction_requested { request: Interaction Request }
interaction_responded { response: Interaction Response }
command_issued        { command: Agent Command }
session_ended         { outcome: Session Outcome }
```

`message.level` is `info`, `warning`, or `error`.

Tool Activity:

```text
started  { call_id, name, summary? }
finished { call_id, outcome, summary? }
```

Tool Outcome is `succeeded`, `failed`, or `cancelled`.

Session Outcome:

```text
completed { summary? }
failed    { error }
cancelled { reason? }
```

### Plan and Progress

Plan Snapshot:

```text
revision: decimal u64 string greater than zero
explanation?: non-empty text
items: [{ id, content, status }]
```

Plan Item IDs must be unique. Status is `pending`, `in_progress`, `completed`, `blocked`, or `skipped`.

Progress Snapshot:

```text
revision: decimal u64 string greater than zero
value: Progress Value
message?: non-empty text
```

Progress Value:

```text
indeterminate {}
determinate   { completed: decimal u64 string, total: decimal u64 string }
```

### Interaction Request Payload

Interaction Request fields:

```text
id: UUIDv7
session_id: UUIDv7
requested_at: RFC 3339
expires_at?: RFC 3339 later than requested_at
prompt: non-empty text
payload: Interaction Request Payload
```

Payload Variants:

```text
approval { allowed_scopes: [once | session] }
choice   { options: [{ id, label, description? }], multiple: boolean }
text     { placeholder?: non-empty text, multiline: boolean }
```

Approval Scope values and Choice Option IDs must be unique. Approval and Choice arrays must not be empty.

### Interaction Response Payload

```text
approval { decision: Approval Decision }
choice   { option_ids: UUIDv7[] }
text     { text: non-empty text }
```

Approval Decision:

```text
approved { scope: once | session }
rejected { reason?: non-empty text }
```

`option_ids` must be non-empty and unique. Matching those IDs to a Request is contextual validation performed later.

### Agent Command Payload

```text
submit_prompt  { text: non-empty text }
cancel_session { reason?: non-empty text }
```

## Validation and fixtures / 校验与 Fixtures

JSON structure is validated before conversion. Decoded DTOs are then rebuilt through the canonical domain constructors, so UUID version, non-empty text, Kind format, timestamp ordering, collection uniqueness, Progress bounds, and embedded Session correlation cannot bypass Core invariants.

JSON 结构会先被校验，随后 DTO 必须通过权威领域构造器重新建立，因此 UUID 版本、非空文本、Kind 格式、时间顺序、集合唯一性、Progress 边界与内嵌 Session 关联都无法绕过 Core 不变量。

The deterministic examples in [`fixtures/v1`](fixtures/v1) cover every top-level message type and are the canonical cross-language Golden Fixtures. Whitespace and object-key order are not semantic; values, field presence, array order, and types are semantic.

[`fixtures/v1`](fixtures/v1) 中的确定性示例覆盖全部顶层消息类型，是跨语言 Golden Fixtures 的权威副本。空白和 Object Key 顺序不属于语义；值、字段存在性、数组顺序与 JSON 类型属于语义。
