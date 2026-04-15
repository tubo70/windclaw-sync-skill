---
name: windclaw-session-sync
description: 在飞书群中同步 WindClaw 的 `.windclaw-aigw-session` 到另一个 OpenClaw。用于初始化或修复 `windclaw-session-sync` agent、将 agent 路由到指定飞书群、根据本机是否存在与 `openclaw.json` 同级目录的 `.windclaw-aigw-session` 自动判定发送端或接收端，并通过人类身份 @ 对端 agent 传递 sessionId。
---

# WindClaw Session Sync

## Hard Rules

- 只创建一个 agent：`windclaw-session-sync`
- 只在飞书群聊中初始化
- 不要作为插件安装；这是共享技能
- 不要在同一端创建两个 agent
- 角色只由本机是否存在与 `openclaw.json` 同级目录的 `.windclaw-aigw-session` 决定
- 发送端才创建或更新定时任务
- 接收端只在被 `@` 且消息格式匹配时写文件

角色判定：

- 文件存在：当前端是发送端
- 文件不存在：当前端是接收端

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
  --description "Sync WindClaw sessionId through a Feishu group according to the local role." \
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

### Detect Role

按下面流程获取 session 文件并判定角色：

1. 执行 `openclaw config file`
2. 获取返回的 `openclaw.json` 路径
3. 检查其同级目录中是否存在 `.windclaw-aigw-session`

规则：

- 文件存在：发送端
- 文件不存在：接收端

## Sender Side

### Maintain Peer OpenId

发送端需要知道接收端 agent 在当前群里的 openid。

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

1. 找到 `@` 了目标 agent 的消息
2. 从该消息的 `mentions` 数组中提取目标 agent 的 `id`
3. 将结果写入 `MEMORY.md`

如果历史消息中没有出现过对端 agent 的 mention，则提示用户先在群里手工 `@` 一次对端 agent，再重新初始化。

### Read SessionId

读取发送端 session 文件：

1. 执行 `openclaw config file`
2. 获取 `openclaw.json` 路径
3. 读取同级目录中的 `.windclaw-aigw-session`
4. 文件内容即为 `sessionId`

如果文件丢失，则当前端不再视为发送端，应按接收端处理。

### Send Message

必须以人类身份发送飞书 `post` 消息，并 `@` 接收端 agent。

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
- `@` 的对象必须是接收端 agent 在该群中的 openid

### Immediate Sync

如果用户要求即时同步，则立即执行一次发送流程：

1. 读取 `MEMORY.md`
2. 必要时重新发现对端 openid
3. 读取 `.windclaw-aigw-session`
4. 发送固定格式消息

### Scheduled Sync

只有发送端创建或更新定时任务。

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

## Receiver Side

接收端目标路径固定为：

```text
~/.openclaw-windclaw/.windclaw-aigw-session
```

只在下面条件全部满足时处理消息：

1. 消息中 `@` 当前 agent
2. 消息文本包含 `sessionId：`
3. 消息文本包含 `，请覆盖写入`

提取规则：

- 取 `sessionId：` 与 `，请覆盖写入` 之间的内容作为 sessionId
- 如果边界不完整，则忽略，不写文件

写入步骤：

1. 确保目录 `~/.openclaw-windclaw` 存在
2. 覆盖写入 `~/.openclaw-windclaw/.windclaw-aigw-session`

## Repair Behavior

当用户再次使用技能时，不要默认停止。执行修复式检查：

1. 检查是否在群聊中
2. 检查 agent 是否存在
3. 检查 route 是否存在
4. 检查角色是否变化
5. 发送端检查 `MEMORY.md` 是否有效
6. 发送端检查定时任务是否存在并与当前周期一致

缺什么就补什么。