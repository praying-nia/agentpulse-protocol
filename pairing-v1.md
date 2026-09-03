# AgentPulse Pairing v1 / 配对协议 v1

## Scope / 范围

Pairing v1 is the complete QR-only public bootstrap from an untrusted Android installation to one authenticated Native Transport device identity. It transfers no Session, Event, or approval data. A pairing session is one-shot, expires after two minutes, permits at most five requests, and requires explicit confirmation on the Host terminal. USB, ADB, Bluetooth, and a shared LAN are not pairing dependencies.

Pairing v1 是从未受信任 Android 安装到已认证 Native Transport 设备身份的完整纯二维码公网 Bootstrap。它不传输 Session、Event 或审批数据。每个配对会话仅可成功一次、两分钟后过期、最多接收五次请求，并且必须由用户在 Host 终端明确确认。配对不依赖 USB、ADB、蓝牙或共享局域网。

## Discovery bundle / 发现包

The Host prints exactly one terminal QR code containing this URI after its ephemeral route is waiting on the configured public Relay:

Host 仅在临时路由已于配置的公网 Relay 上等待后，打印一个包含以下 URI 的终端二维码：

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
| `address`, `port` | private inner pairing target; normally Host loopback and never dialed directly by Android |
| `leaf_sha256` | exactly 64 lowercase hexadecimal characters |
| `bootstrap_token` | random single-use secret |
| `relay_endpoint` | canonical lowercase ASCII public Relay DNS authority `host:port` |
| `expires_at_unix_seconds` | positive UTC Unix seconds |

Unknown fields, missing fields, malformed JSON/scalars, unsupported versions, invalid UUIDs/fingerprints/Relay authorities, and expired bundles are rejected. There is no BLE advertisement, nearby pairing, deep-link pairing, or printed manual URI. Possession of the QR is possession of the short-lived bootstrap capability, but credentials are issued only after Host-terminal approval.

未知字段、缺失字段、错误 JSON/Scalar、未知版本、非法 UUID/指纹/Relay Authority 及过期 Bundle 均被拒绝。协议不提供 BLE 广播、附近配对、Deep Link 配对或打印的手工 URI。持有二维码即持有短时 Bootstrap Capability，但 Host 仅在终端确认后签发设备凭证。

## WebSocket / WebSocket

| Property | Pairing v1 |
| --- | --- |
| Host bind boundary | loopback-only ephemeral listener |
| Public path | authenticated Relay v1 outer TLS tunnel derived from the QR bootstrap Token |
| TLS trust | exact leaf DER SHA-256 from the discovery bundle |
| HTTP path | exactly `/agentpulse/pair/v1`, without a query |
| WebSocket subprotocol | required `agentpulse.pair.v1` |
| Messages | UTF-8 text only, maximum 16 KiB |
| Attempts | at most five per session |
| Lifetime | two minutes, one successful issuance |

For the pairing route, both Host and Android compute `pairing_root = SHA-256(bootstrap_token UTF-8)` and apply the endpoint-bound Relay v1 `route_id` and `client_auth_key` derivations. The Host authenticates the ephemeral registration with its existing Relay enrollment key. Relay sees only the derived route and opaque inner TLS bytes; it never receives the QR bootstrap Token, pairing request, issued bearer Token, or Host CA. Stable Native routes and one ephemeral pairing route may wait concurrently as disjoint authenticated Host registrations.

对于配对路由，Host 与 Android 都计算 `pairing_root = SHA-256(bootstrap_token UTF-8)`，随后使用 Relay v1 的端点绑定 `route_id` 与 `client_auth_key` 派生。Host 使用既有 Relay Enrollment Key 认证临时注册。Relay 只能看到派生路由和不透明的内层 TLS 字节，无法得到二维码 Bootstrap Token、配对请求、签发的 Bearer Token 或 Host CA。稳定 Native 路由与一个临时配对路由可作为互不重叠的认证 Host 注册同时等待。

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
