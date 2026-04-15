---
name: windclaw-session-sync
description: 在 WindClaw 环境中初始化 `windclaw-session-sync` agent，读取与 `openclaw.json` 同级目录中的 `.windclaw-aigw-session`，通过飞书群以人类身份 @ 目标 OpenClaw 并发送 sessionId。仅当与 `openclaw.json` 同级目录存在 `.windclaw-aigw-session` 时使用；用于初始化、修复路由、缓存目标 openid、即时同步和定时同步。
---

# WindClaw Session Sync

## Hard Rules

- 这是共享技能，不要作为插件安装
- 这个技能只能在 WindClaw 环境初始化
- 初始化前必须确认与 `openclaw.json` 同级目录存在 `.windclaw-aigw-session`
- 如果不存在 `.windclaw-aigw-session`，则停止，并提示当前环境不是 WindClaw
- 只创建一个 agent：`windclaw-session-sync`
- 只在飞书群聊中初始化
- 只在当前 WindClaw 端创建或修复 agent、route、cron 和 `MEMORY.md`
- 接收端不安装、不初始化这个技能

## Initialization

### Verify Group Context

只在飞书群聊中执行初始化。根据消息元数据判断：

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

- 如果 `is_group_chat` 不是 `true`，则提示用户必须在飞书群聊中初始化并停止
- 使用 `conversation_label` 作为当前群 id

### Resolve OpenClaw Root

通过主 agent 的 workspace 推导 OpenClaw 根目录：

```bash
main_workspace=$(openclaw agents list | grep -A 6 "^- main" | grep "Workspace:" | awk '{print $2}')
openclaw_root="${main_workspace%/workspace}"
```

当前 skill workspace 使用：

```text
${openclaw_root}/workspace-windclaw-session-sync/
```

### Verify WindClaw Environment

按下面流程确认当前环境就是 WindClaw：

1. 执行 `openclaw config file`
2. 获取返回的 `openclaw.json` 路径
3. 检查其同级目录中是否存在 `.windclaw-aigw-session`

规则：

- 文件存在：允许继续初始化
- 文件不存在：停止，并提示当前环境不是 WindClaw，不能初始化此技能

### Ensure Agent Exists

先检查 agent 是否已经存在：

```bash
openclaw agents list | grep "windclaw-session-sync"
```

规则：

- 如果 agent 已存在，则跳过创建，但继续执行后续校验和补齐
- 如果 agent 不存在，则创建

创建命令：

```bash
openclaw agents add windclaw-session-sync \
  --workspace "${openclaw_root}/workspace-windclaw-session-sync" \
  --description "Read the WindClaw session file and sync sessionId to another OpenClaw through a Feishu group." \
  --bind "feishu"
```

如果环境要求指定模型，则补充 `--model` 参数，并沿用当前默认模型。

### Ensure Group Route Exists

先读取现有 bindings：

```bash
existing_bindings=$(openclaw config get bindings --json)
```

确保存在下面的 route：

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

把 route 和已有 bindings 合并后写回：

```bash
openclaw config set bindings "<merged_bindings_json>"
openclaw config get bindings --json
```

## Peer OpenId

发送消息前需要知道目标 OpenClaw 在当前群里的 openid。

优先从当前 workspace 的 `MEMORY.md` 读取；如果缺失、群 id 不匹配或 openid 为空，则重新发现并写回。

`MEMORY.md` 使用固定结构：

```md
## WindClaw Session Sync
- group_id: <group_id>
- peer_bot_openid: <bot_openid>
- updated_at: <iso_datetime>
```

### Discover Peer OpenId

使用 `feishu_im_user_get_messages` 获取当前群历史消息。

步骤：

1. 找到 `@` 了目标 OpenClaw 的消息
2. 从该消息的 `mentions` 数组中提取目标 agent 的 `id`
3. 将结果写入 `MEMORY.md`

如果历史消息中没有出现过对端 agent 的 mention，则提示用户先在群里手工 `@` 一次目标 OpenClaw，再重新初始化。

## Read SessionId

读取 WindClaw session 文件：

1. 执行 `openclaw config file`
2. 获取 `openclaw.json` 路径
3. 读取同级目录中的 `.windclaw-aigw-session`
4. 文件内容即为 `sessionId`

如果该文件在后续运行中丢失，则停止发送，并提示用户当前环境不再满足 WindClaw 前提。

## Send Message

必须以人类身份发送飞书 `post` 消息，并 `@` 目标 OpenClaw。

消息格式固定为：

```text
@ windclaw-session-sync，sessionId：<plain-text-session-id>，请覆盖写入 '~/.openclaw-windclaw/.windclaw-aigw-session' 文件中。
```

发送调用：

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
- `@` 的对象必须是目标 OpenClaw 在该群中的 openid

## Immediate Sync

如果用户要求即时同步，则立即执行一次发送流程：

1. 读取 `MEMORY.md`
2. 必要时重新发现目标 openid
3. 读取 `.windclaw-aigw-session`
4. 发送固定格式消息

## Scheduled Sync

只有 WindClaw 端创建或更新定时任务。

规则：

- 定时任务名固定为 `WindClaw Session Sync`
- 默认周期为 1 小时
- 用户指定周期时，按用户输入生成 cron 表达式
- 若已存在同名任务，则更新，不重复创建

示例：

```bash
openclaw cron add \
  --name "WindClaw Session Sync" \
  --cron "0 * * * *" \
  --agent windclaw-session-sync \
  --message "使用 windclaw-session-sync 技能同步 session"
```

## Repair Behavior

当用户再次使用技能时，不要默认停止。执行修复式检查：

1. 检查是否在群聊中
2. 检查当前环境是否仍然是 WindClaw
3. 检查 agent 是否存在
4. 检查 route 是否存在
5. 检查 `MEMORY.md` 是否有效
6. 检查定时任务是否存在并与当前周期一致

缺什么就补什么。