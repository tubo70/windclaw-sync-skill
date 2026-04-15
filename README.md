# windclaw-session-sync

这是一个 OpenClaw 共享技能，用于在 WindClaw 环境中读取 `.windclaw-aigw-session`，并通过飞书群把 sessionId 同步到另一个 OpenClaw。

技能目录是 [windclaw-session-sync](windclaw-session-sync)。安装时只复制这个技能目录到 OpenClaw 的共享技能目录，不要把整个仓库当作技能安装，也不要作为插件安装。

这个技能只做发送端工作，而且只能在 WindClaw 环境初始化，也就是与 `openclaw.json` 同级目录存在 `.windclaw-aigw-session` 时才允许使用。目标 OpenClaw 不需要安装这个技能，只需要理解群消息，并把 sessionId 覆盖写入用户 home 目录下的 `.openclaw-windclaw/.windclaw-aigw-session`。