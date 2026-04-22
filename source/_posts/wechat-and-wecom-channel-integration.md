---
title: OpenAgent 接入微信与企业微信消息通道的方案实践
date: 2026-04-22 09:30:00
updated: 2026-04-22 09:30:00
categories:
  - 技术
tags:
  - OpenAgent
  - AI
  - 微信
  - 企业微信
  - Agent
description: 记录 OpenAgent 如何把微信私聊与企业微信 AI Bot 接入统一 Gateway、session binding、去重和 allowlist 体系。
---

OpenAgent 的消息通道设计目标，是让不同 IM 入口都能接入同一套 agent runtime。无论消息来自终端、微信还是企业微信，进入系统后都会被转换成统一的 Gateway envelope，再交给同一个 `Gateway -> runtime -> Gateway egress` 链路处理。

这次接入主要覆盖两个通道：

- 微信私聊通道：面向个人微信私聊文本消息。
- 企业微信通道：面向企业微信 AI Bot 私聊/群聊文本消息。

两者最终都进入 OpenAgent 的统一会话体系，但接入方式不同。差异来自平台协议本身：微信侧使用 iLink Bot API 和 Python SDK；企业微信侧使用 AI Bot WebSocket。

## 总体架构

OpenAgent 没有让模型 runtime 直接感知微信或企业微信，而是把渠道能力收口在 `gateway/channels/*` 下。

整体链路如下：

```text
微信 / 企业微信
-> channel client
-> channel adapter normalize_inbound()
-> Gateway.process_input()
-> OpenAgent runtime
-> channel adapter project_outbound()
-> channel client reply/respond
-> 微信 / 企业微信
```

这里有三个关键分层：

- `client`：负责和外部平台协议通信，例如登录、长连接、收发消息。
- `adapter`：负责把平台消息转换成 OpenAgent 的标准入站/出站结构。
- `host`：负责 glue logic，例如去重、allowlist、session binding、调用 Gateway 和发送回复。

这样做的好处是：平台协议变化时，只需要改对应 channel；OpenAgent 的核心 runtime、工具调用、会话存储和模型调用都保持不变。

## 微信通道方案

微信通道当前面向私聊文本消息，基于 `wechatbot-sdk` 接入 WeChat iLink Bot API。

微信侧的关键复杂度在于登录态和回复上下文：

- 需要扫码登录。
- 收消息走长轮询。
- 服务端会维护 opaque cursor。
- 回复需要正确携带 `context_token`。
- session 过期后需要重新登录。

这些能力更适合交给 SDK 处理。OpenAgent 在微信通道里只做自己应该做的部分：

- 把 SDK 消息对象归一化为 `InboundEnvelope`。
- 用微信用户 id 建立私聊会话绑定。
- 对入站 message id 做去重。
- 支持 `OPENAGENT_WECHAT_ALLOWED_SENDERS` 限制可驱动 agent 的联系人。
- 调用 SDK 的 `reply(msg, text)` 把回答发回原私聊。

微信消息进入 OpenAgent 后，会被标准化成类似结构：

```json
{
  "type": "message",
  "message_type": "text",
  "message_id": "wx_msg_1",
  "conversation_id": "wx_user_1",
  "from_user": "wx_user_1",
  "content": "hello"
}
```

OpenAgent 内部会把它映射成：

```text
channel_type = wechat
conversation_id = wechat:private:<wx_user_id>
session_id = wechat-session:wechat:private:<wx_user_id>
```

启动方式：

```bash
uv sync --extra wechat
export OPENAGENT_WECHAT_ALLOWED_SENDERS=wx_user_1,wx_user_2
uv run openagent-host --channel wechat
```

运行中也可以通过管理命令加载：

```text
/channel-config wechat allowed_senders wx_user_1,wx_user_2
/channel wechat
```

## 企业微信通道方案

企业微信通道当前面向 AI Bot 私聊文本消息，使用 `aiohttp` 直连企业微信 AI Bot WebSocket。

企业微信侧的协议形态更像一个长连接事件流：

- WebSocket 地址为 `wss://openws.work.weixin.qq.com`。
- 建连后发送 `aibot_subscribe` 完成订阅认证。
- 收到用户消息后会下发 frame。
- OpenAgent 处理完成后，通过 `aibot_respond_msg` 回包。

企业微信通道的核心 client 负责：

- 建立 WebSocket 连接。
- 发送订阅帧。
- 发送 ping 保活。
- 接收企业微信 frame。
- 解析 `aibot_msg_callback`。
- 用 `aibot_respond_msg` 回复文本。

订阅成功时，日志会类似：

```text
wecom-host> websocket connected url=wss://openws.work.weixin.qq.com
wecom-host> sent aibot_subscribe frame
wecom-host> subscription acknowledged
```

企业微信消息进入 OpenAgent 后，会被标准化成类似结构：

```json
{
  "type": "message",
  "message_type": "text",
  "message_id": "wecom_msg_1",
  "conversation_id": "chat_1",
  "from_user": "userid_1",
  "content": "hello",
  "reply_context": {
    "req_id": "req_1",
    "msgid": "wecom_msg_1",
    "chatid": "chat_1"
  }
}
```

其中 `reply_context` 只在通道内部使用。Gateway 和 runtime 不需要理解企业微信协议字段。

OpenAgent 内部会把它映射成：

```text
channel_type = wecom
conversation_id = wecom:private:<chat_id>
session_id = wecom-session:wecom:private:<chat_id>
```

启动方式：

```bash
export OPENAGENT_WECOM_BOT_ID=your_bot_id
export OPENAGENT_WECOM_SECRET=your_secret
export OPENAGENT_WECOM_ALLOWED_USERS=userid_1,userid_2
export OPENAGENT_PROVIDER=openai
export OPENAGENT_BASE_URL=http://127.0.0.1:8080
export OPENAGENT_MODEL=Qwen3.5-9B-Q4_K_M.gguf

uv run --extra wecom openagent-host --channel wecom
```

## 两个通道的共同设计

微信和企业微信通道虽然协议不同，但在 OpenAgent 内部保持了相同的抽象。

### 统一 Gateway Envelope

所有平台消息都会被转换为 `InboundEnvelope`：

```text
channel_identity
input_kind
payload
delivery_metadata
```

普通用户消息会进入 `Gateway.process_input()`；管理命令例如 `/channel`、`/channel-config ...` 会进入 host management handler。

### Lazy Session Binding

首次收到某个私聊会话消息时，host 会自动创建 session binding：

```text
一个聊天会话 -> 一个 OpenAgent session
```

这样用户后续在同一个聊天窗口继续对话时，会复用同一个 agent session。

### 入站去重

两个通道都使用 message id 做入站去重，避免长连接重连、轮询重复或平台重投导致同一条消息触发多次 agent。

### Allowlist

两个通道都支持允许列表：

```bash
OPENAGENT_WECHAT_ALLOWED_SENDERS=...
OPENAGENT_WECOM_ALLOWED_USERS=...
```

为空时表示允许所有私聊用户驱动 agent；真实环境建议显式配置。

### Runtime 无渠道感知

模型、工具调用、session/memory、observability 都不需要知道消息来自微信还是企业微信。它们只看到标准化后的用户输入和 runtime event。

这让 OpenAgent 可以继续保持一个核心 runtime，同时扩展多个入口。

## 方案取舍

当前方案的核心选择是：通道层尽量薄，平台协议只停留在 `gateway/channels/*` 内。

微信通道把登录态、长轮询和回复上下文交给 `wechatbot-sdk`，OpenAgent 聚焦消息归一化和 Gateway 接入。

企业微信通道直接实现 AI Bot WebSocket 的最小协议面，OpenAgent 控制连接、订阅、收帧、回复和排障日志。

这两个选择背后的共同原则是：

- 平台协议细节不要进入 runtime。
- 能稳定复用的上下文能力交给对应 client。
- OpenAgent 自己维护 session binding、去重、allowlist 和管理命令。
- 每个通道都通过同一套 Gateway envelope 接入。

最后形成的结构是：

```text
src/openagent/gateway/channels/wechat/
  adapter.py
  client.py
  host.py
  dedupe.py
  assembly.py

src/openagent/gateway/channels/wecom/
  adapter.py
  client.py
  host.py
  dedupe.py
  assembly.py
```

这套结构让新通道的接入边界很清楚：新增一个 channel，只需要实现平台 client、adapter、host 和 assembly，而不用改 OpenAgent 的核心执行链路。
