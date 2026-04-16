---
name: windclaw-session-sync
description: 在 WindClaw 环境中，通过飞书群把 `.windclaw-aigw-session` 同步给另一个 OpenClaw。收到类似触发词“同步 windclaw session”、“同步windclaw”、“windclaw”、“windclaw session” 时执行一次同步。
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
- 初始化时自动开启 `hooks.internal.enabled`
- 初始化时自动在 default agent 的 `workspace` 和 `agentDir` 下创建或更新 `BOOT.md`
- 手动触发词使用：`同步 windclaw session`
- 定时任务发送的消息固定为：`请使用 windclaw-session-sync技能 同步 windclaw session`
- 即时触发和定时触发都优先使用 `MEMORY.md` 中记录的绝对路径

## 术语变量
以下是一些本技能要使用的术语变量
 - `peer_bot_names` : 用户提供需要接收sessionid的bot的名字，如果多个用逗号分隔
 - `peer_bot_openids` : 接收sessionid的bot的openid，如果多个用逗号分隔
 - `openclaw_root` : openclaw根目录, [获取方法](#获取openclaw根目录)
 - `agent_agentDir` : 默认agent的agentDir目录, [获取方法](#获取默认agent目录)
 - `agent_workspace` : 默认agent的workspace目录, [获取方法](#获取默认agent目录)
 - `session_file_path` : `.windclaw-aigw-session` 文件的绝对路径，[获取方法](#获取session文件路径)
 - `group_id` : 当前群 id
 - `session_id` : 需要同步的sessionid，[获取方法](#获取sessionid)
 - `session_key` : 本群聊会话的session key，[获取方法](#获取群聊SessionKey)

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
  "sender_id": "ou_zzzzzzzzz"
}
```

规则：

- 如果 `is_group_chat` 不是 `true`，则提示用户必须在飞书群聊中初始化并停止
- 使用 `conversation_label` 作为当前群 id， 记录为 `group_id`

### 2. Verify WindClaw Environment

1. 获取openclaw根目录
2. 获取session文件目录，如果不存在则停止，提示当前环境不是WindClaw
3. 获取默认agent目录

### 3. Discover Peer OpenId

获取接收sessionid的成员openid

### 4. Write MEMORY.md

把同步需要的状态统一写入当前 workspace 的 `MEMORY.md`。

`MEMORY.md` 使用固定结构：

```md
## WindClaw Session Sync
- group_id: <group_id>
- peer_bot_openids: <peer_bot_openids>
- session_file_path: <session_file_path>
- updated_at: <iso_datetime>
```

规则：

- 如果 `group_id` 与当前群不匹配，则重写
- 如果 `peer_bot_openids` 缺失，则重新发现并重写
- 如果 `session_file_path` 缺失，则重新解析并重写
- 如果 `session_file_path` 指向的文件已不存在，则重新解析；仍不存在则停止

### 5. Create Or Update Scheduled Sync

初始化时自动创建或更新定时任务。

规则：

- 定时任务名固定为 `WindClaw Session Sync`
- 默认周期为 1 小时
- 用户指定周期时，按用户输入生成 cron 表达式
- 若已存在同名任务，则更新，不重复创建
- 定时任务触发消息固定为：`请使用 windclaw-session-sync技能 同步 windclaw session`
- 定时任务执行时，先读取 `MEMORY.md` 中的 `session_file_path`
- 如果 `session_file_path` 无效，再重新[获取session文件路径](#获取session文件路径)


示例：

```bash
openclaw cron add \
  --name "WindClaw Session Sync" \
  --cron "0 * * * *" \
  --session-key <session_key> \
  --message "请使用 windclaw-session-sync技能 同步 windclaw session"
```

### 6. Enable Startup Sync Hook

初始化时自动开启 internal hooks：

```bash
openclaw config set hooks.internal.enabled true
```

在这 `agent_agentDir` 和  `agent_workspace` 两个目录下分别查找或创建：

- `<agent_agentDir>/BOOT.md`
- `<agent_workspace>/BOOT.md`

写入或保持如下内容：

```md
# 启动时自动同步 WindClaw Session

## 使用 sessions_send 工具发送
- sessionkey: <session_key>
- message: "请使用 `windclaw-session-sync` 技能 同步 windclaw session"

```

规则：

- 如果 `BOOT.md` 不存在，则创建
- 如果 `BOOT.md` 已存在但没有上述段落，则追加
- 如果已经存在相同段落，则不要重复写入


## Sync Execution

触发同步时，唯一合法顺序如下：

1. 读取 `MEMORY.md`
2. 从 `MEMORY.md` 中取出 `peer_bot_openid`
3. 从 `MEMORY.md` 中取出 `session_file_path`
4. 必要时重新发现目标 openid
5. 必要时重新解析 `session_file_path`
6. 调用文件工具读取 `session_file_path`，必须使用 `path` 参数，强制从磁盘读取文件内容，不要使用内存中的缓存。
7. 发送固定格式消息

## OpenClaw Tools

### 获取openclaw根目录

通过 `openclaw config file` 的输出解析出openclaw.json文件的目录，即为openclaw根目录，记录成 `openclaw_root` 。

### 获取默认agent目录

通过 `openclaw agents list` 的输出找到 `default` agent 的记录，并从中提取 `agentDir` 和 `workspace` 路径，其中:
- `agentDir` 转换成绝对路径，记录为 `agent_agentDir` 
- `workspace` 转换成绝对路径，记录为 `agent_workspace` 

### 获取session文件路径

同 `openclaw_root` 同级目录的 `.windclaw-aigw-session` 文件的路径，转换成绝对路径，记录成 `session_file_path` 。

### 获取sessionId

直接读取 `session_file_path` 文件的内容即为sessionid，记录成 `session_id` 。

注意：
 - 强制从磁盘读取文件内容，不要使用内存中的缓存

 ### 获取群聊SessionKey

 SessionKey 格式为 `agent:main:feishu:group:<group_id>`，记录为 `session_key` 。
 可以通过 `sessions_list` 工具获取来验证。

## Feishu Tools

### 获取接收sessionid的成员openid

使用 `feishu_im_user_get_messages` 获取当前群历史消息，用用户提供的 `peer_bot_names` 找到对应的id。

步骤：

1. 找到 `@` 了目标的消息
2. 从该消息的 `mentions` 数组中提取目标 agent 的 `id`，记录为 `peer_bot_openids` ，多个用逗号分隔

### 发送sessionid同步消息

如果有多个接收sessionid的openid，先将 `peer_bot_openids` 以逗号解析出 `peer_bot_openid1` 、`peer_bot_openid2` ...
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
          user_id: "<peer_bot_openid1>"
        },
        {
          tag: "at",
          user_id: "<peer_bot_openid2>"
        },
        {
          tag: "text",
          text: "，请将sessionId：<session_id>，覆盖写入 '~/.openclaw-windclaw/.windclaw-aigw-session' 文件中, 只写入sessionid本身。"
        }
      ]]
    }
  })
})
```
