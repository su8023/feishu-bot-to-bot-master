# 飞书 Bot-to-Bot 聊天（多 Agent 互通完整指南）

> 让多个飞书机器人互相聊天的完整实战手册。生产验证于 Hermes Agent 多 profile Agent 团队（2026-06 ~ 2026-10）。

## 一句话结论

飞书允许 bot 与 bot 互聊，但必须同时满足四个条件：

1. 每个 bot 配置 `FEISHU_ALLOW_BOTS`（推荐 `mentions`：只响应被 @ 的消息）
2. 所有 bot 被拉进**同一个群**（重启网关 ≠ 拉进群！）
3. 唤醒对方必须用**原生 `<at user_id="ou_xxx">` 标签**，纯文本 `@名字` 无效
4. 每个应用开通 `im:message.group_at_msg.include_bot:readonly` scope 并发布版本

**Bot↔Bot 私聊被飞书协议禁止 —— 共享群是唯一通道。**

## 你会在这里找到什么

完整内容见 [SKILL.md](./SKILL.md)，十二章：

| # | 章节 | 解决什么问题 |
|---|------|------------|
| 一 | 背景概念 | Profile / App ID / open_id / chat_id 四个 ID 是什么 |
| 二 | 核心配置 | `FEISHU_ALLOW_BOTS` 三档、DM/卡片两套独立准入闸门 |
| 三 | 为什么必须群聊 | bot DM 协议级禁止的实测结论 |
| 四 | 拉进群 | 「WebSocket 连了但收不到群消息」的头号原因 |
| 五 | 唤醒机制 | 原生 `<at>` 标签三条硬规则 |
| 六 | open_id 分域陷阱 | 唤醒用花名册 id、回 @ 用 inbound 占位符 id |
| 七 | 权限 scope | 少一个整链断，缺 `include_bot` 时 Hermes 无解 |
| 八 | 防回环设计 | 四层防线，防止俩 bot 聊到天荒地老 |
| 九 | 接入 Checklist | 新 bot 上群 10 步 SOP |
| 十 | 排查速查表 | 11 种症状 → 先查什么的对号入座表 |
| 十一 | 备选路线 | 飞书群 vs Hermes A2A vs hermes peer |
| 十二 | 真实案例 | 4 个带日期和根因的翻车存档 |

## 三个最贵的坑（先看这个）

- **`FEISHU_ALLOWED_USERS=open`** —— 白名单会变成一个叫 "open" 的假用户，所有真人私聊被**静默丢弃**且几乎无日志。`open`/`all` 只能出现在 POLICY 字段。
- **改了 .env 不重启** —— env 不热加载，改了白改。验证方法：`/proc/<pid>/environ`。
- **open_id 按应用分域** —— 同一个 bot 在自己 app 和别人 app 事件里的 open_id 不同，用错维度时消息 API 返回成功但静默未投递。

## 适用场景

- Hermes Agent 多 profile 团队的飞书协作
- 任何基于飞书长连接（WebSocket 事件）的多 bot 系统
- 企业内多个自建应用（机器人）需要互相触发的自动化流程

## 来源与验证

- 生产环境：DB13-N 多 profile Hermes Agent 集群（7 个 bot）
- 验证周期：2026-06 ~ 2026-10，含五轮群协作实测与多次故障复盘
- 本文档同时沉淀于 NAS Obsidian 知识库（Hermes-Wiki）

## License

MIT
