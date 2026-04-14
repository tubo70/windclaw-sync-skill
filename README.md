# windclaw-session-sync

这是一个 OpenClaw 共享技能，用于把 WindClaw 端的 `.windclaw-aigw-session` 同步到另一个 OpenClaw。

技能只有一个 agent id：`windclaw-session-sync`。用户在使用技能时指定当前机器角色：

- `publisher`：发送端，只创建发送端 agent
- `subscriber`：接收端，只创建接收端 agent

技能通过用户提供的飞书群中转 sessionId。发送端定时读取 WindClaw 的 session 文件，以人类身份发消息并 `@` 接收端 agent；接收端收到后将 sessionId 覆盖写入 `~/.openclaw-windclaw/.windclaw-aigw-session`。

安装方式：将本仓库中的 [windclaw-session-sync](windclaw-session-sync) 目录安装到 OpenClaw 的共享技能目录中。
