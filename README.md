# windclaw-session-sync

这是一个 OpenClaw 共享技能，用于把 WindClaw 端的 `.windclaw-aigw-session` 同步到另一个 OpenClaw。

技能只有一个 agent id：`windclaw-session-sync`，并且只会在当前端创建这一个 agent。

角色不是手工指定，而是自动判断：

- 如果本机存在 `.windclaw-aigw-session`，当前端是发送端
- 如果本机不存在 `.windclaw-aigw-session`，当前端是接收端

发送端会定时读取 WindClaw 的 session 文件，以人类身份发消息并 `@` 接收端 agent；接收端收到后将 sessionId 覆盖写入 `~/.openclaw-windclaw/.windclaw-aigw-session`。

安装方式：将本仓库中的 [windclaw-session-sync](windclaw-session-sync) 目录安装到 OpenClaw 的共享技能目录中。
