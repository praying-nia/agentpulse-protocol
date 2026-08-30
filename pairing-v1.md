# AgentPulse Pairing v1 / 配对协议 v1

## Scope / 范围

Pairing v1 is the complete local bootstrap from an untrusted Android installation to one authenticated, read-only Native Transport device identity. It transfers no Session or Event data. A pairing session is one-shot, expires after two minutes, permits at most five requests, and requires explicit confirmation on the Host terminal.

Pairing v1 是从未受信任 Android 安装到已认证、只读 Native Transport 设备身份的完整本地 Bootstrap。它不传输 Session 或 Event 数据。每个配对会话仅可成功一次、两分钟后过期、最多接收五次请求，并且必须由用户在 Host 终端明确确认。

## Discovery bundle / 发现包

The Host exposes the same URI through a secure BLE GATT characteristic and a terminal QR fallback:

Host 通过安全 BLE GATT Characteristic 与终端 QR 备用路径暴露同一个 URI：

```text
agentpulse://pair/v1/<base64url-without-padding(JSON)>
```

The decoded JSON object has exactly these fields:

| Field | Contract |
| --- | --- |
| `pairing_version` | JSON integer `1` |
| `pairing_id` | one-time canonical UUIDv7 |
| `host_id` | stable canonical UUIDv7 |
| `host_name` | nonblank user-facing name |
| `server_name` | stable TLS DNS name |
| `address`, `port` | current private LAN pairing endpoint |
| `leaf_sha256` | exactly 64 lowercase hexadecimal characters |
| `bootstrap_token` | random single-use secret |
| `expires_at_unix_seconds` | positive UTC Unix seconds |

Unknown fields, malformed JSON/scalars, unsupported versions, invalid UUIDs/fingerprints, and expired bundles are rejected. BLE advertises service UUID `d22e50f9-015e-53ba-be49-3e4d235f3288`; the URI is read from characteristic `ea63bfc9-87c3-5074-aa37-49b6a617569b`. BLE access requires an authenticated encrypted link and operating-system association. BLE carries only this bootstrap URI; all request and credential delivery uses pinned WSS.

未知字段、错误 JSON/Scalar、未知版本、非法 UUID/指纹及过期 Bundle 均被拒绝。BLE 使用 Service UUID `d22e50f9-015e-53ba-be49-3e4d235f3288`，并从 Characteristic `ea63bfc9-87c3-5074-aa37-49b6a617569b` 读取 URI。BLE 访问要求经过认证的加密链路及操作系统 Association。BLE 只承载 Bootstrap URI；请求与凭证下发全部通过 Pinned WSS 完成。

## WebSocket / WebSocket

| Property | Pairing v1 |
| --- | --- |
| Bind boundary | explicit private/link-local LAN address only |
| TLS trust | exact leaf DER SHA-256 from the discovery bundle |
| HTTP path | exactly `/agentpulse/pair/v1`, without a query |
| WebSocket subprotocol | required `agentpulse.pair.v1` |
| Messages | UTF-8 text only, maximum 16 KiB |
| Attempts | at most five per session |
| Lifetime | two minutes, one successful issuance |

Every message uses `{ "pairing_version": 1, "message": { ... } }` and rejects unknown fields.

每条消息都使用 `{ "pairing_version": 1, "message": { ... } }`，并拒绝未知字段。

The client sends exactly one `pair_request` containing the matching `pairing_id` and `bootstrap_token`, its stable installation UUIDv7 `client_id`, a nonblank `display_name`, and optional nonblank `version`. Bootstrap comparisons are constant-time. A valid request receives `pairing_pending` before local approval.

Client 发送一条 `pair_request`，包含匹配的 `pairing_id` 与 `bootstrap_token`、稳定安装 UUIDv7 `client_id`、非空 `display_name` 及可选非空 `version`。Bootstrap 比较使用 Constant-time。合法请求在本地批准前先收到 `pairing_pending`。

After approval, `pairing_succeeded` returns the stable Host identity/name, Base64 DER app-scoped CA, stable TLS server name, current Native address/port, a random per-device bearer token, Native Transport version `1`, and supported domain protocol versions. The client validates the Host identity against the discovery bundle and persists only this Host credential material in OS-protected storage. It never persists Session/Event data.

批准后，`pairing_succeeded` 返回稳定 Host 身份/名称、Base64 DER 应用内 CA、稳定 TLS Server Name、当前 Native 地址/端口、随机的每设备 Bearer Token、Native Transport 版本 `1` 及支持的领域协议版本。Client 必须将 Host Identity 与 Discovery Bundle 对照，并且只在操作系统保护存储中持久化这组 Host 凭证；不得持久化 Session/Event 数据。

`pairing_error` uses one of `invalid_request`, `invalid_credential`, `expired`, `used`, `denied`, `capacity`, or `internal`, plus a bounded message and `recoverable`. Denial, expiry, use, and successful issuance are terminal. Invalid requests/credentials consume an attempt and never reveal which secret component differed.

`pairing_error` 使用 `invalid_request`、`invalid_credential`、`expired`、`used`、`denied`、`capacity` 或 `internal`，并携带有界 Message 与 `recoverable`。拒绝、过期、已使用及成功签发均为终止状态。非法请求/凭证会消耗一次 Attempt，且不会透露具体哪个 Secret 分量不匹配。

## Credential lifecycle / 凭证生命周期

The Host stores only SHA-256 hashes of bearer tokens, supports at most 16 devices, and reloads authorization state for every check. Device revocation therefore invalidates new upgrades and active connections. Credential rotation replaces the local CA and leaf while retaining the stable Host ID and revokes every device. Leaf certificates last 90 days and are renewed within 14 days of expiry under the same CA.

Host 仅保存 Bearer Token 的 SHA-256 Hash，最多支持 16 台设备，并在每次授权校验时重新加载状态。因此设备撤销会同时阻止新 Upgrade 并终止活动连接。凭证轮换会保留稳定 Host ID、替换本地 CA 与 Leaf，并撤销所有设备。Leaf 有效期为 90 天，进入到期前 14 天时由同一 CA 自动续签。

Canonical examples are in [`fixtures/pairing-v1`](fixtures/pairing-v1). Whitespace and object key order are not semantic; field presence, JSON type, and array order are semantic.

权威示例位于 [`fixtures/pairing-v1`](fixtures/pairing-v1)。空白与 Object Key 顺序不属于语义；字段存在性、JSON 类型及 Array 顺序属于语义。
