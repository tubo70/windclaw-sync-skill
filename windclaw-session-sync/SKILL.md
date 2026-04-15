---
name: windclaw-session-sync
description: 在飞书群中同步 WindClaw 的 `.windclaw-aigw-session` 到另一个 OpenClaw。技能会创建一个agent，并根据本机是否存在 `.windclaw-aigw-session` 自动判断当前端是发送端还是接收端。
---

# WindClaw Session Sync

## Overview

这个技能用于在两个 OpenClaw 端点之间同步 WindClaw 的 sessionId。

技能使用的 agent id：

- `windclaw-session-sync`

角色自动判断规则：

- 如果本机存在与 `openclaw.json` 同级目录的 `.windclaw-aigw-session` 文件 ，当前端是发送端
- 如果本机不存在与 `openclaw.json` 同级目录的 `.windclaw-aigw-session` 文件，当前端是接收端

注意：
 - 这是技能，请不要作为插件安装。

## 初始化

### 检查是否在飞书群聊中

初始化必须在飞书群聊中执行。

在收到使用此技能的请求后，根据消息元数据判断是否处于群聊，并获取群 id。

```json
{
  "is_group_chat": true,
  "conversation_label": "oc_xxxxxxxxx",
  "group_subject": "oc_yyyyyyyyy",
  "was_mentioned": true,
  "sender_id": "ou_097d373f70d1d2a05ee2ba0e52cccc48"
}
```

规则：

- 如果 `is_group_chat` 不是 `true`，则告诉用户必须在飞书群聊中初始化，并停止
- `conversation_label` 作为当前群 id 使用

### 检查 agent 是否已经存在

使用以下命令检查 agent 是否已经存在：

```bash
openclaw agents list | grep "windclaw-session-sync"
```

规则：

- 如果 agent 已存在，则跳过创建步骤，但继续执行后续校验和补齐
- 如果 agent 不存在，则继续创建

### 获取当前端角色

根据 [获取 `.windclaw-aigw-session` 文件内容](#获取-windclaw-aigw-session-文件内容) 的结果判断当前端角色：

- 文件存在：当前端是发送端
- 文件不存在：当前端是接收端

注意：

- 不再依赖用户手工指定 `publisher` 或 `subscriber`
- 角色以本地文件状态为唯一判断依据

### 获取群聊中被@对象的 openid

如果当前端是发送端，则需要知道接收端 agent 在当前群里的 openid。

先从当前 workspace 的 `MEMORY.md` 中读取。如果不存在，再通过 [获取群聊中用户的 openid](#获取群聊中用户的-openid) 的方法重新获取。

将记录写入 `MEMORY.md` 时，使用固定结构：

```md
## WindClaw Session Sync
- group_id: <group_id>
- peer_bot_openid: <bot_openid>
- updated_at: <iso_datetime>
```

如果当前端是接收端，则跳过该步骤。

### 轮询周期

可选输入：

- 轮询周期

如果用户未指定轮询周期，默认使用 1 小时。

## Resolve OpenClaw Root

通过主 agent 的 workspace 推导 OpenClaw 根目录：

```bash
main_workspace=$(openclaw agents list | grep -A 6 "^- main" | grep "Workspace:" | awk '{print $2}')
openclaw_root="${main_workspace%/workspace}"
```

## Create Workspace

为当前 agent 创建独立 workspace。

目录结构：

```text
${openclaw_root}/workspace-windclaw-session-sync/
```

## Create Agent

只创建当前端需要的一个 agent，只绑定 channel，不在创建时写入具体 group peer。

创建命令：

```bash
openclaw agents add windclaw-session-sync \
  --workspace "${openclaw_root}/workspace-windclaw-session-sync" \
  --description "Sync WindClaw sessionId through a Feishu group according to the local role." \
  --bind "feishu"
```

如果运行环境要求指定模型，则补充 `--model` 参数，并沿用当前环境默认模型。

## Add Route Bindings

`openclaw agents add` 只绑定 channel。要把当前 agent 路由到具体群，需要补充 bindings。

先读取现有 bindings：

```bash
existing_bindings=$(openclaw config get bindings --json)
```

然后为当前 agent 追加 route 规则。目标效果如下：

```json
[
  {
    "type": "route",
    "agentId": "windclaw-session-sync",
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

确认当前 agent 已经绑定到 `feishu/group/<group_id>`。

## 定时任务

只有发送端需要定时任务。

规则：

- 如果当前端存在 `.windclaw-aigw-session`，则创建或更新同名定时任务
- 如果当前端不存在 `.windclaw-aigw-session`，则不创建定时任务
- 定时任务名固定为 `WindClaw Session Sync`
- 默认周期为 1 小时，用户指定则覆盖默认值

使用openclaw cron add 创建定时任：
cron 表达式根据定时周期设置

``` bash
openclaw cron add \
  --name "WindClaw Session Sync" \
  --cron "0 * * * *" \ 
  --agent windclaw-session-sync \
  --message "使用 windclaw-session-sync 技能同步 session"
```

请用如下命令创建定时任务：
```bash
openclaw cron add --name windclaw-session-sync --every 1h --agent windclaw-session-sync --message "请使用 `windclaw-session-sync` 执行 windclaw session 同步"
```

## 即时同步

如果用户要求即时同步，则立即执行一次发送流程。

规则：

- 只有发送端可以执行即时同步
- 发送前先从 `MEMORY.md` 读取接收端 openid
- 如果 `MEMORY.md` 不存在，则重新获取并写回 `MEMORY.md`

## 获取 `.windclaw-aigw-session` 文件内容

1. 执行 `openclaw config file`
2. 获取返回的 `openclaw.json` 路径
3. 读取同级目录中的 `.windclaw-aigw-session`
4. 如果文件存在，则文件内容即为 sessionId
5. 如果文件不存在，则将当前端识别为接收端，而不是直接报错停止

## 接收端写入规则

接收端目标路径固定为：

```text
~/.openclaw-windclaw/.windclaw-aigw-session
```

收到消息后：

1. 只有当消息中 `@` 当前 agent 时才处理
2. 只有当消息文本包含 `sessionId：` 时才处理
3. 提取 `sessionId：` 与 `，请覆盖写入` 之间的内容作为 sessionId
4. 确保目录 `~/.openclaw-windclaw` 存在
5. 覆盖写入 `~/.openclaw-windclaw/.windclaw-aigw-session`

## 飞书工具使用

### 以人类身份发送消息并 `@` 消息

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
          user_id: "<bot_openid>"
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

### 获取群聊中用户的 openid

发送前需要得到被 `@` 的 agent 的 openid，可用下面方法获取：

1. 使用 `feishu_im_user_get_messages` 获取当前群的历史消息
2. 从返回的消息列表中找到 `@` 了目标 agent 的消息
3. 提取该消息 `mentions` 数组中目标 agent 的 `id` 字段，这就是目标 agent 在该群中的 openid
4. 将结果写入当前 workspace 的 `MEMORY.md`

如果历史消息中没有出现过对端 agent 的 mention，则提示用户先在群里手工 `@` 一次对端 agent，再重新执行初始化。
