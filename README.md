# windclaw-session-sync

这是一个 OpenClaw 共享技能，用于在 WindClaw 环境中读取 `.windclaw-aigw-session`，并通过飞书群把 sessionId 同步到另一个 OpenClaw。

## 安装

必须先安装这个技能，后续才能使用。

本仓库根目录不是技能目录。

真正的技能目录名称是：`windclaw-session-sync/`

从 GitHub 仓库安装时，必须将整个 `windclaw-session-sync/` 目录复制到 OpenClaw 的用户级共享技能目录中（默认为与openclaw.json同级目录下的skills目录），并保持目录名称仍然为 `windclaw-session-sync`。

`openclaw config file` 的输出可以确定openclaw.json文件的目录，即为openclaw根目录。

不要把仓库根目录直接当作技能目录。
不要把仓库内容拆开复制。
不要跳过技能安装步骤。

## 说明

这个技能只做发送端工作，而且只能在 WindClaw 环境使用，也就是与 `openclaw.json` 同级目录存在 `.windclaw-aigw-session` 时才允许使用。目标 OpenClaw 不需要安装这个技能，只需要理解群消息，并把 sessionId 覆盖写入用户 home 目录下的 `.openclaw-windclaw/.windclaw-aigw-session`。