# agentpulse-protocol

Canonical cross-platform protocol specification for AgentPulse.

AgentPulse 的跨平台协议规范仓库。

This repository defines the implementation-independent contract between Providers, the AgentPulse Core and local Bridge, the optional self-hosted Relay, and Channels. It is the source of truth for interoperable event and interaction semantics across AgentPulse implementations.

本仓库定义 Provider、AgentPulse Core 与本地 Bridge、可选的自托管 Relay、Channel 之间与具体实现无关的通信约定，是 AgentPulse 各端事件语义与交互语义的统一来源。

The protocol is channel-neutral: it describes what happened and what response is requested, never how a mobile screen, bot message, or Webhook payload should render it. Its shared model is expected to cover concepts such as:

协议与展示 Channel 无关：它描述发生了什么以及需要何种响应，不规定移动端界面、Bot 消息或 Webhook Payload 应如何呈现。统一模型计划覆盖以下概念：

- `AgentSession`, `AgentState`, and `AgentEvent` / Agent 会话、状态与事件。
- `Progress`, `Plan`, todo items, and tool activity / 进度、计划、Todo 与工具活动。
- `InteractionRequest` and `InteractionResponse`, including approval, choice, and text input flows / 交互请求与响应，包括审批、选择与文本输入。
- Completion, failure, cancellation, and connectivity states / 完成、失败、取消与连接状态。
- Provider identity and negotiated Provider/Channel capabilities / Provider 身份与协商后的 Provider/Channel 能力。

The semantic contract and its invariants are defined in [Unified Domain Model](domain-model.md). Runtime-neutral adapter boundaries and centralized capability policy are defined in [Ports and Capability Routing](ports-and-routing.md). The versioned JSON representation is defined separately in [JSON Wire Protocol v1](wire-v1.md).

具体语义及其不变量由[统一领域模型](domain-model.md)定义；运行时中立的 Adapter 边界与集中 Capability 策略由[端口与能力路由](ports-and-routing.md)定义；版本化 JSON 表示由 [JSON 线协议 v1](wire-v1.md)独立规定。

For example, an approval remains an `InteractionRequest::Approval` throughout the Core. A Native Channel may render it as buttons, while a Bot Channel may present `/approve <interaction-id>` and `/reject <interaction-id>` commands.

例如，一个审批请求在 Core 中始终是 `InteractionRequest::Approval`。Native Channel 可以将其呈现为按钮，而 Bot Channel 可以将其表现为 `/approve <interaction-id>` 与 `/reject <interaction-id>` 命令。

## Capabilities / 能力

Each Provider declares which agent-side events and write-back operations it supports. Initial Provider Capability names are expected to include:

每个 Provider 声明自身支持的 Agent 事件与回写操作。首批 Provider Capability 名称计划包括：

```text
SESSION_STATE
TOOL_EVENTS
PLAN
PROGRESS
APPROVAL_REQUEST
APPROVAL_RESPONSE
USER_INPUT_REQUEST
USER_INPUT_RESPONSE
PROMPT_SUBMIT
CANCEL
```

Each Channel separately declares which presentation and input forms it supports. Initial Channel Capability names are expected to include:

每个 Channel 独立声明自身支持的展示与输入形式。首批 Channel Capability 名称计划包括：

```text
NOTIFICATION
SESSION_VIEW
TOOL_VIEW
PLAN_VIEW
PROGRESS_VIEW
RICH_MESSAGE
APPROVAL
CHOICE_INPUT
TEXT_INPUT
FORM_INPUT
REALTIME_SYNC
REMOTE_COMMAND
```

Capabilities are negotiated across the complete route. The Core must not assume that every Provider accepts remote input or that every Channel can render every interaction. Unsupported operations degrade to the available notification or read-only behavior instead of being presented as actionable.

能力需要在完整链路上协商。Core 不能假设每个 Provider 都接受远程输入，也不能假设每个 Channel 都能呈现全部交互。不受支持的操作应退化为当前可用的通知或只读行为，不能表现为可执行操作。

The `agentpulse-protocol` crate in `agentpulse-rs` will provide the Rust implementation of this specification; it is not the canonical specification itself.

`agentpulse-rs` 内的同名 crate 将负责本规范的 Rust 实现，但不取代本仓库的规范权威性。

## Status / 状态

The initial channel-neutral domain semantics, strict JSON protocol v1, independent Provider/Channel ports, centralized capability routing, and minimal Bridge orchestration are defined. Canonical cross-language examples are available as [Golden Fixtures](fixtures/v1).

首版与 Channel 无关的领域语义、严格 JSON 协议 v1、Provider/Channel 独立端口、集中 Capability 路由及 Bridge 最小编排已经确定；跨语言规范示例位于 [Golden Fixtures](fixtures/v1)。
