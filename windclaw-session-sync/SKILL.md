---
name: windclaw-session-sync
description: 在 WindClaw 环境中，通过飞书群把 `.windclaw-aigw-session` 同步给另一个 OpenClaw。用于在群聊中初始化、确认当前环境是 WindClaw、记录目标 openid 和 session 文件绝对路径，并在初始化时自动创建或更新定时同步任务。收到触发词“同步 windclaw session”时执行一次同步。仅当与 `openclaw.json` 同级目录存在 `.windclaw-aigw-session` 时使用。
---

# WindClaw Session Sync

## Hard Rules

- 这是共享技能，不要作为插件安装
- 只在飞书群聊中初始化
- 只在 WindClaw 环境使用这个技能
- 初始化前必须确认与 `openclaw.json` 同级目录存在 `.windclaw-aigw-session`
- 如果不存在 `.windclaw-aigw-session`，则停止，并提示当前环境不是 WindClaw
- 接收端不安装、不初始化这个技能
- 初始化时就解析 `.windclaw-aigw-session` 的绝对路径，并写入 `MEMORY.md`
- 初始化时自动创建或更新定时任务，不需要用户额外要求
- 即时触发和定时触发都使用同一个触发词：`同步 windclaw session`
- 即时触发和定时触发优先使用 `MEMORY.md` 中记录的绝对路径

## Initialization

初始化按下面顺序执行。

### 1. Verify Group Context

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

### 2. Verify WindClaw Environment

按下面流程确认当前环境就是 WindClaw，并同时得到 session 文件绝对路径：

1. 执行 `openclaw config file`
2. 获取返回的 `openclaw.json` 绝对路径
3. 用该绝对路径推导同级 `.windclaw-aigw-session` 的绝对路径
4. 检查该绝对路径对应的文件是否存在

规则：

- 文件存在：允许继续，并记录该绝对路径
- 文件不存在：停止，并提示当前环境不是 WindClaw

### 3. Discover Peer OpenId

发送消息前需要知道目标 OpenClaw 在当前群里的 openid。

使用 `feishu_im_user_get_messages` 获取当前群历史消息。

步骤：

1. 找到 `@` 了目标 OpenClaw 的消息
2. 从该消息的 `mentions` 数组中提取目标 agent 的 `id`
3. 将结果记录下来

如果历史消息中没有出现过对端 agent 的 mention，则提示用户先在群里手工 `@` 一次目标 OpenClaw，再重新执行初始化。

### 4. Write MEMORY.md

把同步需要的状态统一写入当前 workspace 的 `MEMORY.md`。

`MEMORY.md` 使用固定结构：

```md
## WindClaw Session Sync
- group_id: <group_id>
- peer_bot_openid: <bot_openid>
- session_file_path: <absolute_path>
- updated_at: <iso_datetime>
```

规则：

- 如果 `group_id` 与当前群不匹配，则重写
- 如果 `peer_bot_openid` 缺失，则重新发现并重写
- 如果 `session_file_path` 缺失，则重新解析并重写
- 如果 `session_file_path` 指向的文件已不存在，则重新解析；仍不存在则停止

### 5. Create Or Update Scheduled Sync

初始化时自动创建或更新定时任务。

规则：

- 定时任务名固定为 `WindClaw Session Sync`
- 默认周期为 1 小时
- 用户指定周期时，按用户输入生成 cron 表达式
- 若已存在同名任务，则更新，不重复创建
- 定时任务触发消息固定为：`同步 windclaw session`
- 定时任务执行时，先读取 `MEMORY.md` 中的 `session_file_path`
- 如果 `session_file_path` 无效，再重新通过 `openclaw config file` 解析绝对路径并回写 `MEMORY.md`
- 不要在定时任务中直接读取相对路径 `.windclaw-aigw-session`

示例：

```bash
openclaw cron add \
  --name "WindClaw Session Sync" \
  --cron "0 * * * *" \
  --agent main \
  --message "请使用 `windclaw-session-sync` 同步 windclaw session"
```

## Read SessionId

读取 WindClaw session 文件时按下面顺序执行：

1. 先读取 `MEMORY.md`
2. 如果存在 `session_file_path`，则优先使用这个绝对路径读取文件
3. 如果 `session_file_path` 不存在或读取失败，则重新执行 `openclaw config file`，重新解析绝对路径
4. 将新解析出的绝对路径回写到 `MEMORY.md`
5. 文件内容即为 `sessionId`

规则：

- 不要直接读取相对路径 `.windclaw-aigw-session`
- 不要依赖 `~/` 或 `~\` 这种 home 简写路径
- 不要把 `~` 作为字面量路径交给文件工具
- 在 Windows 定时任务上下文中，始终使用绝对路径
- 文件工具优先使用 `path` 参数
- 如果重新解析后仍然找不到文件，则停止发送，并提示当前环境不再满足 WindClaw 前提

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

## Sync Execution

同步流程如下：

1. 读取 `MEMORY.md`
2. 必要时重新发现目标 openid
3. 必要时重新解析 `session_file_path`
4. 用绝对路径读取 `.windclaw-aigw-session`
5. 发送固定格式消息