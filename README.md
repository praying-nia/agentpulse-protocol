# agentpulse-protocol

Canonical cross-platform protocol specification for AgentPulse.

AgentPulse 的跨平台协议规范仓库。

This repository defines the implementation-independent contract between local bridges, the self-hosted relay, mobile clients, and supported agent adapters. It is the source of truth for interoperable event and interaction semantics across AgentPulse implementations.

本仓库定义本地 Bridge、自托管 Relay、移动客户端和各 Agent Adapter 之间与具体实现无关的通信约定，是 AgentPulse 各端事件语义与交互语义的统一来源。

The protocol is expected to cover:

- Agent identity, capabilities, sessions, and task lifecycle events.
- Plans, todo items, progress updates, and tool activity.
- Waiting-for-input and permission-request flows.
- Completion, failure, cancellation, and connectivity states.
- Capability-gated responses such as approve, reject, answer, and short text submission.

协议计划覆盖：

- Agent 身份、能力、会话与任务生命周期事件。
- Plan、Todo、进度更新与工具活动。
- 等待输入与权限请求流程。
- 完成、失败、取消与连接状态。
- 基于能力协商的批准、拒绝、回答与简短文本提交。

The `agentpulse-protocol` crate in `agentpulse-rs` will provide the Rust implementation of this specification; it is not the canonical specification itself.

`agentpulse-rs` 内的同名 crate 将负责本规范的 Rust 实现，但不取代本仓库的规范权威性。

## Status / 状态

Repository scaffold only; no wire format or protocol version has been selected yet.

当前仅完成仓库占位，尚未确定线格式或协议版本。

