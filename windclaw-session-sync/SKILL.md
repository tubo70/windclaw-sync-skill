---
name: windclaw-session-sync
description: 在飞书群中同步 WindClaw 的 `.windclaw-aigw-session` 到另一个 OpenClaw。技能只创建当前端需要的一个 agent，用户在使用技能时指定当前角色是发送端还是接收端。
---

# WindClaw Session Sync

## Overview

这个技能用于在两个 OpenClaw 端点之间同步 WindClaw 的 sessionId。

## 初始化

### 检查agent是否已经存在

使用以下命令检查 agent 是否已经存在：
``` bash
openclaw agents list | grep "windclaw-session-sync"
```
如果已经存在，则停止。如果不存在，则继续初始化。

### 初始化环境

初始化必须要飞书群聊中。
在收到使用此技能的请求后，根据收到的元数据判断是否在群聊中，已及获取群id。
``` json
  "is_group_chat": true,          // ← 这里：true=群聊, false=私聊
  "conversation_label": "oc_xxxxxxxxx",  // 群ID
  "group_subject": "oc_yyyyyyyyy",      // 群主题
  "was_mentioned": true,           // 是否被@
  "sender_id": "ou_097d373f70d1d2a05ee2ba0e52cccc48"  
```
如果不是在群聊中，则告诉用户并停止后续操作。

根据 [获取 `.windclaw-aigw-session`](#获取-windclaw-aigw-session-文件内容) 文件内容, 如果文件不存在则跳过[openid获取](#获取群聊中用户的openid), 继续。

#### 获取群聊中用户的openid
然后通过 `feishu_im_user_get_messages` 获取这条消息，从 mentions 中获取其他agent的openid。
将其他agent的openid写入到当前workspace的 `MEMORY.md` , 记录是用于 `WindClaw Sesseion Sync` 技能发送sessionid时@的对象。

可选：

- 轮询周期

如果用户未指定轮询周期，默认使用 1 小时。


### Resolve OpenClaw Root

通过主 agent 的 workspace 推导 OpenClaw 根目录：

```bash
main_workspace=$(openclaw agents list | grep -A 6 "^- main" | grep "Workspace:" | awk '{print $2}')
openclaw_root="${main_workspace%/workspace}"
```

### Create Workspace

为当前 agent 创建独立 workspace。

目录结构建议：

```text
${openclaw_root}/workspace-windclaw-session-sync/
```

### Create Agent

只创建当前端需要的一个 agent，只绑定 channel，不在创建时写入具体 group peer。

创建命令：

```bash
openclaw agents add windclaw-session-sync \
  --workspace "${openclaw_root}/workspace-windclaw-session-sync" \
  --description "Sync WindClaw sessionId through a Feishu group according to the local role." \
  --bind "feishu"
```

如果运行环境要求指定模型，则补充 `--model` 参数，并沿用当前环境默认模型。

### Add Route Bindings

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

### 定时任务
如果存在`.windclaw-aigw-session` 文件，则根据轮询周期创建定时任务。
定时获取 `.windclaw-aigw-session` 文件内容，然后以人类身份发送消息并 `@` 接收端 agent。

## 即时同步
用户使用技能时要求即时同步，那么先获取`.windclaw-aigw-session` 文件内容，然后以人类身份发送消息并 `@` 接收端 agent。
接收端的agent的openid先从当前workspace的 `MEMORY.md` 中获取。如果 `MEMORY.md` 中不存在。则重新[获取群聊中用户的openid](#获取群聊中用户的openid)。然后发送消息并 `@` 获取到的openid。

## 获取 `.windclaw-aigw-session` 文件内容
1. 执行 `openclaw config file`
2. 获取返回的 `openclaw.json` 路径
3. 读取同级目录中的 `.windclaw-aigw-session`
4. 如果 `.windclaw-aigw-session` 文件不存在，则告诉用户并停止。
5. `.windclaw-aigw-session` 文件内容即为 sessionId.


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

### 获取群聊中用户的openid

发送前需要得到被@ 的agent的openid，可用下面方法获取：

1. 使用 feishu_im_user_get_messages 获取群的历史消息
2. 从返回的消息列表中找到 @了目标 agent 的消息
3. 提取该消息的 mentions 数组中目标 agent 的 id 字段，这就是目标 agent 在该群中的 openid。
4. 将其他agent的openid写入到当前workspace的 `MEMORY.md` , 记录是用于 `WindClaw Sesseion Sync` 技能发送sessionid时@的对象。




