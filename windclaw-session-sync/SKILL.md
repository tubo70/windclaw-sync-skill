---
name: windclaw-session-sync
description: 在一个飞书群内创建并驱动两个 OpenClaw agent，把 WindClaw 端的 `.windclaw-aigw-session` 同步到另一个 OpenClaw 的 `~/.openclaw-windclaw/.windclaw-aigw-session`。发送端通过人类身份发消息并 @ 接收端 agent，群 id 由用户提供，轮询周期可选，默认 1 小时。
---

# WindClaw Session Sync

## Overview

这个技能用于在两个 OpenClaw 端点之间同步 WindClaw 的 sessionId。

执行结果如下：

1. 在 WindClaw 所在机器创建一个发布端 agent。
2. 在目标 OpenClaw 所在机器创建一个接收端 agent。
3. 将两个 agent 都路由到用户指定的飞书群。
4. 发布端定时读取 `.windclaw-aigw-session`。
5. 当 sessionId 变化时，发布端以人类身份向群里发送消息，并 `@` 接收端 agent。
6. 接收端收到消息后，将 sessionId 写入 `~/.openclaw-windclaw/.windclaw-aigw-session`。

这是单技能方案，不额外创建独立代码工程。

## Defaults

- 技能 id：`windclaw-session-sync`
- 发布端 agent：`windclaw-session-publisher`
- 接收端 agent：`windclaw-session-subscriber`
- 默认轮询周期：`3600000` 毫秒
- 群消息内容：明文 sessionId
- 当前不做来源限制

## Required Input

至少需要：

- 飞书群 id

可选：

- 轮询周期
- 发布端 agent 名称
- 接收端 agent 名称
- 技能 id

如果用户未指定轮询周期，默认使用 1 小时。

## Roles

同一个技能支持两个角色：

- `publisher`
- `subscriber`

两个 agent 共用同一个技能，通过角色区分行为。

## Agent Creation And Routing

创建 agent 和路由到指定飞书群时，使用下面的固定流程。

前置约束：

- 需要创建两个 agent
- 两个 agent 都使用 `windclaw-session-sync`
- 两个 agent 都要被路由到用户指定的同一个飞书群
- 发布端 agent 角色为 `publisher`
- 接收端 agent 角色为 `subscriber`

### Validate Group Id

在开始创建前，先校验用户提供的飞书群 id 可访问：

```javascript
feishu_chat({
  action: "get",
  chat_id: "<group_id>"
})
```

如果群不存在，或者当前账号无权限访问，直接报错并停止。

### Resolve OpenClaw Root

通过主 agent 的 workspace 推导 OpenClaw 根目录：

```bash
main_workspace=$(openclaw agents list | grep -A 6 "^- main" | grep "Workspace:" | awk '{print $2}')
openclaw_root="${main_workspace%/workspace}"
```

### Create Workspaces

为两个 agent 创建独立 workspace，不共享目录。

目录结构建议：

```text
${openclaw_root}/workspace-windclaw-session-sync/
  windclaw-session-publisher/
  windclaw-session-subscriber/
```

每个 agent 的 workspace 都独立，避免文件互相污染。

### Create Agents

先创建 agent，只绑定 channel，不在创建时写入具体 group peer。

创建命令：

```bash
openclaw agents add windclaw-session-publisher \
  --workspace "${openclaw_root}/workspace-windclaw-session-sync/windclaw-session-publisher" \
  --description "Publish WindClaw sessionId to a Feishu group and mention the subscriber agent." \
  --bind "feishu"
```

```bash
openclaw agents add windclaw-session-subscriber \
  --workspace "${openclaw_root}/workspace-windclaw-session-sync/windclaw-session-subscriber" \
  --description "Receive WindClaw sessionId from a Feishu group and write it to the local session file." \
  --bind "feishu"
```

如果运行环境要求指定模型，则补充 `--model` 参数，并沿用当前环境默认模型。

### Add Route Bindings

`openclaw agents add` 只绑定 channel。要把 agent 路由到具体群，需要补充 bindings。

先读取现有 bindings：

```bash
existing_bindings=$(openclaw config get bindings --json)
```

然后为两个 agent 追加 route 规则。目标效果如下：

```json
[
  {
    "type": "route",
    "agentId": "windclaw-session-publisher",
    "match": {
      "channel": "feishu",
      "peer": {
        "kind": "group",
        "id": "<group_id>"
      }
    }
  },
  {
    "type": "route",
    "agentId": "windclaw-session-subscriber",
    "match": {
      "channel": "feishu",
      "peer": {
        "kind": "group",
        "id": "<group_id>"
      }
    }
  }
]
```

把这些 route 规则和已有 bindings 合并后写回：

```bash
openclaw config set bindings "<merged_bindings_json>"
```

写回后再校验一次：

```bash
openclaw config get bindings --json
```

确认两个 agent 都已经绑定到同一个 `feishu/group/<group_id>`。

## Session File Rules

### Publisher Side

发布端 session 文件路径的确定方式：

1. 执行 `openclaw config file`
2. 获取返回的 `openclaw.json` 路径
3. 取其同级目录
4. 读取同级目录中的 `.windclaw-aigw-session`

即：

```text
dirname($(openclaw config file))/.windclaw-aigw-session
```

### Subscriber Side

接收端写入固定路径：

```text
~/.openclaw-windclaw/.windclaw-aigw-session
```

写入前必须确保目录存在。

## Message Protocol

群消息正文使用自然语言固定格式，不使用 JSON。

消息内容如下：

```text
@ <subscriber_agent_name>，sessionId：<plain-text-session-id>，请覆盖写入 '~/.openclaw-windclaw/.windclaw-aigw-session' 文件中。
```

规则：

- 仅在 sessionId 变化时发送
- 消息正文尽量保持稳定，不随意改写措辞
- sessionId 直接明文放在消息中

## Feishu Rules

### Human Send With Mention

发送端必须以人类身份发送飞书 `post` 消息，并 `@` 接收端 agent。

发送格式：

```javascript
feishu_im_user_message({
  action: "send",
  msg_type: "post",
  receive_id_type: "chat_id",
  receive_id: "<group_id>",
  content: JSON.stringify({
    zh_cn: {
      content: [[
        {
          tag: "at",
          user_id: "<subscriber_agent_openid>",
          user_name: "<subscriber_agent_name>"
        },
        {
          tag: "text",
          text: "，sessionId：<plain-text-session-id>，请覆盖写入 '~/.openclaw-windclaw/.windclaw-aigw-session' 文件中。"
        }
      ]]
    }
  })
})
```

要求：

- `msg_type` 必须为 `post`
- `receive_id_type` 必须为 `chat_id`
- 发送身份必须是人类
- `@` 的对象必须是接收端 agent 在该群中的 openid

### OpenId Discovery

发送前需要先获取：

- 人类在该群中的 openid
- 接收端 agent 在该群中的 openid

直接使用 `feishu_chat_members` 读取群成员：

```javascript
feishu_chat_members({
  chat_id: "<group_id>",
  page_size: 50
})
```

从返回的成员列表中识别：

- 人类账号对应的 openid
- 接收端 agent 对应 bot 的 openid

如果一页不够，则继续翻页，直到拿到目标成员。

## Workflow

### Step 1: Confirm Inputs

确认以下信息：

- 飞书群 id
- 轮询周期，未指定则设为 1 小时
- 发布端所在机器
- 接收端所在机器

### Step 2: Create Two Agents

创建两个 agent：

1. `windclaw-session-publisher`
2. `windclaw-session-subscriber`

两者都使用技能：

```text
windclaw-session-sync
```

角色配置：

- 发布端：`publisher`
- 接收端：`subscriber`

创建时使用独立 workspace，只绑定 `feishu` channel，不直接写 group peer。

### Step 3: Route Both Agents To The Group

将两个 agent 都路由到用户指定的飞书群。

目标是：

- 群消息能够触达接收端 agent
- 群内可以 `@` 到接收端 agent

路由时通过 `openclaw config get bindings --json` 读取现有配置，再追加 `type=route`、`channel=feishu`、`peer.kind=group`、`peer.id=<group_id>` 的规则，最后用 `openclaw config set bindings` 写回。

### Step 4: Publisher Logic

发布端按固定周期执行：

1. 执行 `openclaw config file`
2. 找到同级 `.windclaw-aigw-session`
3. 读取文件内容作为 sessionId
4. 如果内容为空，跳过
5. 如果和上次发送值相同，跳过
6. 如果不同，则以人类身份向飞书群发送 `post` 消息，并 `@` 接收端 agent

### Step 5: Subscriber Logic

接收端收到群消息后：

1. 判断消息是否 `@` 当前 agent
2. 从消息文本中提取 `sessionId：` 后面的 sessionId
3. 确保目录 `~/.openclaw-windclaw` 存在
4. 覆盖写入 `~/.openclaw-windclaw/.windclaw-aigw-session`

### Step 6: Idempotency

为了避免重复写入和重复发送：

- 发布端记录上次成功发送的 sessionId
- 接收端记录上次成功写入的 sessionId

如果新值与旧值一致，直接跳过。

## Operating Rules

- 不要把这个技能实现成独立 Node 或 Python 项目，除非用户后续明确要求
- 当前阶段只保留技能说明、执行步骤和接口占位
- 发送端和接收端共用一个技能，通过角色区分
- 群聊内当前只有一个人类和两个 agent bot，因此暂不增加额外来源校验

## Expected Outcome

完成后将得到：

1. 一个技能：`windclaw-session-sync`
2. 两个 agent：
   - `windclaw-session-publisher`
   - `windclaw-session-subscriber`
3. 一个飞书群作为中转信道
4. 一条稳定同步链路：
   - WindClaw 端检测 sessionId 变化
   - 以人类身份发群消息并 `@` 接收端
   - 接收端写入本地 session 文件

## When Information Is Missing

如果具体接口方法还未给全，先保留流程和占位，不额外扩展工程。

后续只需要补充这些方法：

- 以人类身份发送 `@` bot 的消息

拿到这些方法后，再将本技能补全为可直接执行的版本。
