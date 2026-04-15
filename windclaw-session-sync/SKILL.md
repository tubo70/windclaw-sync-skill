---
name: windclaw-session-sync
description: 在 WindClaw 环境中，通过飞书群把 `.windclaw-aigw-session` 同步给另一个 OpenClaw。用于在群聊中初始化、确认当前环境是 WindClaw、记录目标 openid、即时同步和定时同步。仅当与 `openclaw.json` 同级目录存在 `.windclaw-aigw-session` 时使用。
---

# WindClaw Session Sync

## Hard Rules

- 这是共享技能，不要作为插件安装
- 只在飞书群聊中初始化
- 只在 WindClaw 环境使用这个技能
- 初始化前必须确认与 `openclaw.json` 同级目录存在 `.windclaw-aigw-session`
- 如果不存在 `.windclaw-aigw-session`，则停止，并提示当前环境不是 WindClaw
- 接收端不安装、不初始化这个技能

## Initialize In Group Chat

初始化必须在飞书群聊中执行。根据消息元数据判断：

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

## Verify WindClaw Environment

按下面流程确认当前环境就是 WindClaw：

1. 执行 `openclaw config file`
2. 获取返回的 `openclaw.json` 绝对路径
3. 检查其同级目录中是否存在 `.windclaw-aigw-session`

规则：

- 文件存在：允许继续
- 文件不存在：停止，并提示当前环境不是 WindClaw

## Remember Peer OpenId

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

如果历史消息中没有出现过对端 agent 的 mention，则提示用户先在群里手工 `@` 一次目标 OpenClaw，再重新执行初始化。

## Read SessionId

读取 WindClaw session 文件：

1. 执行 `openclaw config file`
2. 获取 `openclaw.json` 的绝对路径
3. 用该绝对路径推导同级 `.windclaw-aigw-session` 的绝对路径
4. 用文件工具读取这个绝对路径，优先使用 `path` 参数
5. 文件内容即为 `sessionId`

规则：

- 不要依赖 `~/` 或 `~\` 这种 home 简写路径
- 不要把 `~` 作为字面量路径交给文件工具
- 在 Windows 定时任务上下文中，始终重新计算绝对路径，不要复用带 `~` 的字符串
- 如果该文件在后续运行中丢失，则停止发送，并提示当前环境不再满足 WindClaw 前提

## Send Message

必须以人类身份发送飞书 `post` 消息，并 `@` 目标 OpenClaw。

消息格式固定为：

```text
@ windclaw-session-sync，sessionId：<plain-text-session-id>，请覆盖写入用户 home 目录下的 '.openclaw-windclaw/.windclaw-aigw-session' 文件中。
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
          text: "，sessionId：<plain-text-session-id>，请覆盖写入用户 home 目录下的 '.openclaw-windclaw/.windclaw-aigw-session' 文件中。"
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
- 在 Windows 上先展开 home 目录为绝对路径，再用 `path` 参数读写目标文件

## Immediate Sync

如果用户要求即时同步，则立即执行一次发送流程：

1. 读取 `MEMORY.md`
2. 必要时重新发现目标 openid
3. 读取 `.windclaw-aigw-session`
4. 发送固定格式消息

## Scheduled Sync

如果用户要求定时同步，则创建或更新一个定时任务。

规则：

- 定时任务名固定为 `WindClaw Session Sync`
- 默认周期为 1 小时
- 用户指定周期时，按用户输入生成 cron 表达式
- 若已存在同名任务，则更新，不重复创建
- 定时任务执行时，也必须先通过 `openclaw config file` 重新解析 `.windclaw-aigw-session` 的绝对路径，再读取文件

示例：

```bash
openclaw cron add \
  --name "WindClaw Session Sync" \
  --cron "0 * * * *" \
  --agent windclaw-session-sync \
  --message "使用 windclaw-session-sync 技能同步 session"
```