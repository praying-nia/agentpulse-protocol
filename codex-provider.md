# Codex App Server Provider Contract

This document defines the canonical AgentPulse mapping for Codex observation, approvals, structured user input, and bounded remote control. Provider-private Codex decision payloads are not exposed on the public wire.

本文定义 Codex 观察、审批、结构化用户输入与有界远程控制的权威映射。Provider 私有的 Codex Decision Payload 不会暴露到公共线协议。

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
| `item/tool/requestUserInput` | One atomic form preserving field order, choices, Other/free-text support, blocking state, and secret display policy |
| `serverRequest/resolved` | `InteractionResponded` after an AgentPulse answer was written, otherwise `InteractionClosed(ResolvedElsewhere)` |
| Completed nonblank `agentMessage` item | Baseline informational `Message`; `final_answer` is preferred over commentary for the outcome summary |
| Completed turn | `SessionEnded(Completed)`, `Cancelled`, or `Failed` using the native terminal status and error |
| Other valid items/notifications or unconfigured threads | `ValidatedUnmapped`, with no AgentPulse sequence consumed |

Event sequences are contiguous per Session. A sequence advances after Bridge state accepts the event, including the case where later Channel fan-out partially fails. Exact repeated item/turn notifications are bounded and deduplicated. Notifications for conflicting simultaneous turn IDs are terminal semantic failures rather than guessed reordering.

## Interaction, command, and history policy / 交互、指令与历史策略

- The descriptor declares session state, approval request/response, user-input request/response, prompt submission, cancellation, and control capabilities.
- Supported App Server interactions are `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, and `item/tool/requestUserInput`. An unsupported request on the Provider observer receives JSON-RPC `-32601`; a request on a proxied desktop route remains transparent for that desktop client to handle.
- Public approval options use freshly generated opaque IDs. A private runtime-only map binds each ID to one exact Codex decision: `accept`, `acceptForSession`, `decline`, `cancel`, or a supplied exec/network policy amendment. A Channel cannot synthesize or alter that decision object.
- An approval request is still forwarded to the desktop client as a fallback. AgentPulse records which proxy route owns it and writes a phone selection only to that route's upstream WebSocket; request IDs from different clients never alias.
- An approval has no AgentPulse deadline. It remains pending until Codex confirms `serverRequest/resolved`, the desktop client resolves it, its item/turn/thread ends, its owning proxy disconnects, or the Provider stops. AgentPulse never guesses approval or rejection.
- Command, cwd, reason, network host/protocol, paths, change kinds, and exact diffs are preserved. Requests missing safe presentation details or exceeding the 256 KiB presentation bound are explicitly read-only with a reason and no response options.
- A user-input response answers every form field exactly once. Unknown choices, missing fields, duplicate fields, and text for fields without Other/free-text support are rejected. Secret answers exist only in the outbound response until it is written to Codex; pending/resolved Provider state does not retain them.
- Common typed commands cover queued prompts and explicit steer, stop, model selection/catalog, Plan mode, thread list/resume/new, compact, review, rename, fork, status, permission profiles, and queue pause/resume/clear. They map only to generated-schema-validated App Server calls.
- Prompt queues are process-memory FIFO, limited to 32 items per Session, 64 KiB per item, and 1 MiB in total. `stop` preserves and pauses the queue. A rejected `turn/start` preserves the head item and pauses draining until an explicit queue resume.
- At most 64 interactions/responses and 64 control commands exist per Provider runtime. They are memory-only and disappear when that runtime stops.
- Remote `/resume` first establishes the current Session snapshot, then calls `thread/items/list` in ascending pages until `nextCursor` is absent. User and assistant message history enters the current Host run once; reopening an already tracked thread does not duplicate it. `/resume` listings preserve newest-first order within two groups, placing the current working directory first.
- Automatic thread discovery may follow runtime-opened threads, but neither configured thread IDs nor pending approvals are persisted. Network Transport, Channel behavior, Relay, database, and offline history remain outside this Provider contract.
