# Codex App Server Provider Contract

This document defines the canonical AgentPulse mapping for Codex observation and command/file approval write-back. It is a Provider-private integration contract; it does not expose Codex decision payloads on the public wire.

本文定义 Codex 观察以及命令/文件审批回写的权威映射。该协议属于 Provider 私有集成边界，不会把 Codex Decision Payload 暴露到公共线协议。

## Integration boundary / 接入边界

- The Provider uses the official [Codex App Server](https://developers.openai.com/codex/app-server) instead of scraping PTY/TUI output.
- Verified CLI releases are `0.150.1`, `0.152.0`, and `0.152.1`; the bundled `0.152.0`/`0.152.1` schemas are byte-identical. A valid newer SemVer starts best-effort with a visible warning and remains subject to strict schema failure. Older unknown or malformed versions are rejected before runtime creation.
- The complete generated JSON Schema is bundled and validated offline. Every client request/notification and every server request/notification/response/error must pass its corresponding schema.
- Responses on the Provider-owned observer connection must match a pending request ID and the expected method-specific result schema. Responses belonging to a proxied desktop connection are generically schema-validated and passed through because that desktop client owns their correlation. Ambiguous frames, binary frames, and schema-invalid data remain failures of the connection that carried them.
- Schema-valid messages outside the normalized mapping are counted as `ValidatedUnmapped`; they are not silently treated as mapped AgentPulse events.

Provider 使用官方 Codex App Server 而不是抓取 PTY/TUI。已验证版本为 `0.150.1`、`0.152.0` 与 `0.152.1`；更高的合法 SemVer 会带明确警告尽力启动，较旧未知版本或非法版本会在创建运行目录前拒绝。全部双向 JSON 均按内置 Schema 离线校验，响应还必须匹配请求 ID 与方法结果类型。合法但当前领域不承载的消息记为 `ValidatedUnmapped`。

## Topology and lifecycle / 拓扑与生命周期

1. One Provider instance owns one App Server process and one private `0700` runtime directory.
2. The App Server listens upstream on `unix://<owned-directory>/app-server.sock`; the Provider observer connects directly to it.
3. `codex --remote <uri>` clients connect to `unix://<owned-directory>/client.sock`. Each accepted client receives a dedicated transparent upstream WebSocket, and AgentPulse observes server frames on that same request-owning route.
4. Startup performs exact version verification, process readiness, WebSocket upgrade, `initialize`, `initialized`, and `thread/resume` for every explicitly configured thread before exposing the client proxy.
5. Startup succeeds only after every configured thread is resumed. All resume attempts are reported; a partial failure remains non-transactional under the RuntimeHost contract and is cleaned by `stop`.
6. `stop` closes proxy clients and the Provider observer, stops bounded workers, terminates and waits for the owned process, and removes only the owned socket directory. Occupied directories are never overwritten or recursively deleted.
7. A Provider-observer process, transport, correlation, or schema failure publishes `Disconnected` for every tracked Session while ingress remains valid, records terminal Provider health, and requires an explicit RuntimeHost stop/start to recover. A single proxied desktop connection failure closes only approvals owned by that route and leaves the desktop free to reconnect.

首版完整支持 Linux 与 macOS。其他平台返回明确的 `UnsupportedPlatform`。Provider 不自动重启受管进程，避免在调用方不知情时断开共享的 `codex --remote` 客户端。

## Identity and event mapping / 身份与事件映射

Each configured Codex thread ID must be a UUIDv7 and is reused as the AgentPulse `SessionId`. One Codex thread maps to one AgentPulse Session; Codex `sessionId` tree membership does not merge distinct threads.

每个配置的 Codex Thread ID 必须是 UUIDv7，并直接复用为 AgentPulse `SessionId`。一个 Codex Thread 对应一个 AgentPulse Session；Codex `sessionId` 所表示的线程树不会合并不同 Thread。

| Codex input | AgentPulse mapping |
| --- | --- |
| First successful `thread/resume` | Sequence 1 `SessionStarted`; `name` then `preview` supplies title, `cwd` supplies workspace, native timestamps are preserved |
| Resume of an existing mapping | Reconcile connection and current execution state from the returned snapshot; never a second `SessionStarted` |
| `thread/status/changed: idle` | `StateChanged(Idle)` |
| `thread/status/changed: active` | `Running`, or `WaitingForInteraction` when an active waiting flag is present |
| `thread/status/changed: systemError` | `StateChanged(Failed)` |
| `thread/status/changed: notLoaded`, `thread/closed` | `ConnectionChanged(Disconnected)` |
| `turn/started` | `StateChanged(Running)` and reset per-turn summary selection |
| `item/started`, `item/fileChange/patchUpdated` | Runtime-only item detail cache used to present the exact command or latest complete file diff |
| `item/commandExecution/requestApproval` | Structured command/network approval with all decisions actually accepted by the pinned Codex schema |
| `item/fileChange/requestApproval` | Structured file changes, exact diffs, requested grant root, and all accepted file decisions |
| `serverRequest/resolved` | `InteractionResponded` after an AgentPulse answer was written, otherwise `InteractionClosed(ResolvedElsewhere)` |
| Completed nonblank `agentMessage` item | Baseline informational `Message`; `final_answer` is preferred over commentary for the outcome summary |
| Completed turn | `SessionEnded(Completed)`, `Cancelled`, or `Failed` using the native terminal status and error |
| Other valid items/notifications or unconfigured threads | `ValidatedUnmapped`, with no AgentPulse sequence consumed |

Event sequences are contiguous per Session. A sequence advances after Bridge state accepts the event, including the case where later Channel fan-out partially fails. Exact repeated item/turn notifications are bounded and deduplicated. Notifications for conflicting simultaneous turn IDs are terminal semantic failures rather than guessed reordering.

## Approval and history policy / 审批与历史策略

- The descriptor declares `SESSION_STATE`, `APPROVAL_REQUEST`, and `APPROVAL_RESPONSE`. `AgentCommand` remains unsupported.
- Supported App Server requests are exactly `item/commandExecution/requestApproval` and `item/fileChange/requestApproval`. An unsupported request on the Provider observer receives JSON-RPC `-32601`; a request on a proxied desktop route remains transparent for that desktop client to handle.
- Public approval options use freshly generated opaque IDs. A private runtime-only map binds each ID to one exact Codex decision: `accept`, `acceptForSession`, `decline`, `cancel`, or a supplied exec/network policy amendment. A Channel cannot synthesize or alter that decision object.
- An approval request is still forwarded to the desktop client as a fallback. AgentPulse records which proxy route owns it and writes a phone selection only to that route's upstream WebSocket; request IDs from different clients never alias.
- An approval has no AgentPulse deadline. It remains pending until Codex confirms `serverRequest/resolved`, the desktop client resolves it, its item/turn/thread ends, its owning proxy disconnects, or the Provider stops. AgentPulse never guesses approval or rejection.
- Command, cwd, reason, network host/protocol, paths, change kinds, and exact diffs are preserved. Requests missing safe presentation details or exceeding the 256 KiB presentation bound are explicitly read-only with a reason and no response options.
- At most 64 approvals and 64 queued responses exist per Provider runtime. They are memory-only and disappear when that runtime stops.
- Turns included inside `thread/resume` are not replayed. The resume response establishes the current Session snapshot; only subsequent notifications enter the AgentPulse event sequence.
- Automatic thread discovery may follow runtime-opened threads, but neither configured thread IDs nor pending approvals are persisted. Network Transport, Channel behavior, Relay, database, and offline history remain outside this Provider contract.
