# windclaw-session-sync

这是一个 OpenClaw 共享技能，用于把 WindClaw 端的 `.windclaw-aigw-session` 同步到另一个 OpenClaw。

技能目录是 [windclaw-session-sync](windclaw-session-sync)。安装时只复制这个技能目录到 OpenClaw 的共享技能目录，不要把整个仓库当作技能安装，也不要作为插件安装。

技能只使用一个 agent id：`windclaw-session-sync`。它会根据与 `openclaw.json` 同级目录中是否存在 `.windclaw-aigw-session` 自动判断当前端是发送端还是接收端。发送端负责定时或即时发送 sessionId，接收端负责在收到匹配消息后覆盖写入 `~/.openclaw-windclaw/.windclaw-aigw-session`。