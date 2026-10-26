---
name: feishu-bot-to-bot-chat
description: 飞书多 Agent（bot-to-bot）互通实战指南。Use when 配置多个飞书机器人互相聊天、bot 被 @ 不唤醒、群消息静默丢失、open_id 分域混乱、多 agent 群协作防回环。涵盖 FEISHU_ALLOW_BOTS 配置、原生 <at> 唤醒、两套准入闸门、10 步接入 SOP 与 11 种症状排查表。生产验证于 Hermes Agent 多 profile 团队（2026-06 ~ 2026-10）。
category: devops
version: 1.0.0
---


# 飞书 Bot-to-Bot 聊天（多 Agent 互通完整指南）

> **一句话结论**：飞书平台允许 bot 与 bot 互聊，但必须同时满足四个条件 —— ① 每个 bot 配置 `FEISHU_ALLOW_BOTS`；② bot 必须被**拉进同一个群**；③ 唤醒对方必须用**原生 `<at user_id="...">` 标签**（纯文本 @ 无效）；④ 开通 `im:message.group_at_msg.include_bot:readonly` scope。**Bot↔Bot 私聊（DM）被飞书协议禁止，共享群是唯一通道。**

---

## 一、背景概念（先搞清四个 ID）

| 概念 | 说明 | 例子 |
|---|---|---|
| **Profile** | Hermes 的完整 agent 实例：独立 .env（飞书凭据）、SOUL.md（人设）、state.db（会话库） | `abuilder`、`aplanner`、`ascanner` |
| **App ID** | 飞书自建应用的身份，**一个 app = 一个 bot**，多 agent 必须多 app | `cli_xxxxxxxxxxxx` |
| **Open ID** | bot/用户在**单个 app 视角下**的 ID（分域！见第六节） | `ou_53c4fb1aa0b7b5f7ae2b848f870be941` |
| **Chat ID** | 群聊 ID，发群消息用 | `oc_d7b256d3b0de98b8f47e2fd0d664d8ed` |

同一个 bot 在不同 app 的事件里 open_id **不同** —— 这是 bot-to-bot 最阴的坑。

---

## 二、核心配置（每个参与的 bot 都要设）

```bash
# ~/.hermes/profiles/<name>/.env
FEISHU_ALLOW_BOTS=mentions        # none | mentions | all（见下表）
FEISHU_BOT_OPEN_ID=ou_xxx         # 本 bot 自己的 open_id（不设则自己发的消息都认不出）
FEISHU_GROUP_POLICY=open          # 或 allowlist + FEISHU_GROUP_ALLOWED=oc_xxx
FEISHU_DM_POLICY=allowlist        # DM 准入与群准入是两套独立闸门
FEISHU_ALLOWED_USERS=ou_xxx       # 只填真人 open_id，逗号分隔
```

**`FEISHU_ALLOW_BOTS` 三档**：

| 值 | 行为 | 适用 |
|---|---|---|
| `none`（默认） | 忽略一切 bot 来信 | 单兵 bot |
| `mentions` | 只接收 **@ 了自己**的 bot 消息 | **多 bot 团队首选**（防误触防回环） |
| `all` | 接收一切 bot 消息 | 开放闲聊群（必须配防回环协议） |

> ⚠️ 改完 `.env` 必须**重启 gateway** 才生效（`systemctl restart hermes-gateway-<name>`；gateway 内部被拦截就 `kill <pid>` 让 systemd 自动拉起）。改了不重启 = 改了白改。

### 飞书准入是「两套独立闸门」（amedia 案例教训）

| 闸门 | 环境变量 | 管什么 | 坏了什么症状 |
|---|---|---|---|
| DM 消息准入 | `FEISHU_DM_POLICY` + `FEISHU_ALLOWED_USERS` | 私聊消息 | seen 文件在涨但日志**零 inbound**（静默丢弃） |
| 批准卡授权 | `FEISHU_GROUP_POLICY` + admins | 卡片按钮点击 | `Unauthorized approval click by ou_xxx` WARNING |

**修一个 ≠ 修完**。动配置前把所有 `FEISHU_*_POLICY` / `FEISHU_ALLOWED_*` 字段**全部列出来一起看**，改完重启并验证 `/proc/<pid>/environ`。

> ☠️ 经典翻车：`FEISHU_ALLOWED_USERS=open` —— 解析器按逗号切分，白名单变成 `{"open"}` 这个假用户，**所有真人 DM 被静默丢弃**，且只有 debug 级日志，默认根本看不见。`open`/`all` 这种词**只能**出现在 POLICY 字段，**绝不能**出现在 USERS 字段。

---

## 三、为什么必须用群聊

飞书 IM 协议**禁止 bot 给 bot 发私聊**（即使互为好友/同租户）。实测结论：

- ❌ bot↔bot DM：发送 API 返回成功，对方永远收不到
- ✅ bot↔bot 群聊：双方都在群里就能收到
- ❌ 租户隔离：不同企业租户的 bot 互相 DM 更是不可能

**结论：建一个共享群，把所有 bot 拉进去，一切互通走群。**

---

## 四、拉进群（重启 ≠ 拉群，最高频翻车点）

`hermes gateway restart` 只重连 WebSocket，**不会把 bot 加进任何群**。日志显示 `✓ feishu connected` 不代表 bot 是群成员。

**正确操作**（群主在飞书客户端手动做）：
> 群设置 → 群机器人 → 添加机器人 → 选择对应应用

**症状指纹**：日志 WebSocket 已连接，但 `Inbound group message received` 为 **0** —— 就是「连上了但不是群成员」。注意群成员 API 只返回真人，**确认 bot 是否在群要看飞书客户端的群机器人列表或接收方日志**（add_members API 返回 code:0 可能是假成功）。

---

## 五、唤醒机制：原生 `<at>` 标签（纯文本 @ 无效）

Hermes 的 `send()` **不会**把文本里的 `@名字` 转成真实 mention；接收方 `_mentions_self()` 只认 `mentions[]` 元数据。所以：

```text
# ❌ 无效 —— 只是普通文字，对方收不到唤醒
@策划师 请出方案

# ✅ 有效 —— 原生 mention 标签，飞书渲染成真 @ 并携带元数据
<at user_id="ou_8f589f291e26abbcbcf6e769680c6a12">策划师</at> 请出方案
```

三条硬规则：
1. **带 `<at>` 的消息必须是纯文本（text）**。Hermes 会把 markdown 体转成 post/卡片，mention 元数据在转换中丢失（渲染成字面文本或被吞）。
2. **@ 的 user_id 必须是对方的 open_id**（见第六节分域规则）。
3. **一次只 @ 一个执行人**，不要同时 @ 多个（会抢答刷屏）。

---

## 六、open_id 分域陷阱（最阴的坑）

open_id 是**按 app 分域**的：同一个 bot，在它自己 app 的 `bot/v3/info` 里查到的是「self-scope id」，而在**别的 app 收到的 inbound 事件里是另一个 `ou_...`**。

**规则（实测验证）**：
- **唤醒**对方 → 用对方**花名册（self-scope）**的 open_id
- **回复时 @ 发送者** → 用**自己 inbound 消息里 `[Mentioned: 名字 (open_id=…)]` 占位符**里的 id
- 用错维度 → 消息 API 返回成功但**静默未投递**

每个 agent 的 SOUL.md 里要维护一张**花名册表**（名字 | open_id | 专长 | 状态），并且协议里写明「回 @ 用 inbound 占位符的 id」。

---

## 七、权限 scope（少一个就整链断）

每个参与 bot 的飞书应用后台都要开（开了还要**发布版本**才生效）：

| Scope | 作用 |
|---|---|
| `im:message` | 收发消息基础 |
| `im:message.group_at_msg` | 群 @ 消息 |
| `im:message.group_at_msg.include_bot:readonly` | **接收「bot @ bot」事件**（缺了它，被别的 bot @ 也不产生事件，Hermes 侧无解） |
| `im:message.p2p_msg:readonly` | 私聊消息 |
| `application:bot.basic_info:read` | 显示对方 bot 名字而非裸 open_id（可选，装饰性） |

**bot 加入群聊不需要拉人权限** —— 由群主在客户端手动添加。

---

## 八、防回环设计（否则俩 bot 能聊到天荒地老）

`FEISHU_ALLOW_BOTS=all` 的 Dispatcher 模式极易死循环：A 回复触发 B，B 的回复又触发 A，直到限频。

**多层防线（全部用上）**：
1. **配置层**：默认全员 `mentions`，只有被 @ 才说话（未被 @ 的 bot 平台 0 投递，天然静默）。
2. **协议层**（写进每个 SOUL.md）：
   - 只有被 `<at>` 点名才发言；未被点名回复 `NO_REPLY`
   - 同一话题 bot 间接力 **≤4 轮**，每人同一话题最多发言 3 次
   - 话题终结者输出 `【结论】…` 收尾，且**收尾那条不许 @ 任何人**
3. **平台层**：Hermes 的 `gateway/bot_loop_guard.py` 对 bot 来源消息风暴有滑动窗口熔断（如 12 条/300s → 冷却 900s）。
4. **限频**：共享 GLM key 的团队并发即限流，群接力要**错峰**，降级模型会乱语，宁可等。

> ⚠️ `NO_REPLY` 只在**自主回合**被网关静默吞掉；在被直接 @ 的回合里，裸 `NO_REPLY` 会变成「⚠️ silence marker」警告气泡发到群里（实测踩坑）。所以沉默机制要靠「平台不投递」设计，不要靠回复 NO_REPLY。

---

## 九、新 Bot 接入群协作 Checklist（10 步）

1. 飞书后台建自建应用，开机器人能力，记下 App ID / Secret
2. 开 scope（第七节表格），**发布版本**
3. 事件订阅选「**长连接**」模式（WebSocket，免公网），订阅 `im.message.receive_v1`
4. `hermes profile create <name> --clone` 建 profile（不要手工 mkdir）
5. 写 profile `.env`：`FEISHU_APP_ID/SECRET` + `FEISHU_ALLOW_BOTS=mentions` + `FEISHU_BOT_OPEN_ID`（用 `GET /open-apis/bot/v3/info` 查）+ 四件套 POLICY
6. SOUL.md 写人设 + **花名册表** + 群协作协议（防回环三条）
7. 群主把 bot 拉进共享群（客户端手动）
8. 重启 gateway，验证 `WebSocket connected` + `/proc/<pid>/environ` 里 env 正确
9. **A→B→A 测试**：A 在群里 `<at>` B → B 应答并回 @ A → A 收到
10. 双方日志各 grep 一遍 open_id，确认双向 inbound

---

## 十、故障排查速查（按症状对号入座）

| 症状 | 先查 |
|---|---|
| 日志 WebSocket 连接正常，群消息 0 inbound | bot 没进群（第四节）；不是配置问题 |
| outbound code:0 但对方收不到 | open_id 分域用错（第六节）；或对方 scope 缺 `include_bot:readonly` |
| 对方收到消息但没被唤醒 | @ 用了纯文本而不是 `<at>` 标签；或消息被转成 markdown/卡片丢 mention |
| 日志出现 `dropping message from bot` | `FEISHU_ALLOW_BOTS` 没设/没重启 |
| 日志出现 `my own open_id unknown` | `FEISHU_BOT_OPEN_ID` 没设 |
| 日志出现 `Unauthorized approval click` | 卡片闸门（GROUP_POLICY/admins），不是 DM 白名单 |
| 真人 DM 全部静默丢 | `FEISHU_ALLOWED_USERS` 填了 `"open"` 之类的假值 |
| 只有一个方向通（A→B 通 B→A 不通） | B 没进群；或 B 的 app 版本没重新发布 |
| 改了配置不生效 | 没重启 gateway；env 改动不热加载 |
| 群里提示「操作太频繁」 | 飞书侧限频，等 1 分钟，别连点别重启 |

---

## 十一、备选互通路线（按场景选）

| 路线 | 场景 | 关键点 |
|---|---|---|
| **飞书共享群**（本篇） | 要人在环旁听、移动端可见 | 全部靠 `<at>` + 群 |
| **Hermes A2A 平台** | 任务委派/编排，无需人看 | 每 profile 一个 loopback server，自带防循环（ping-pong 上限）+ 审计 jsonl |
| **hermes peer** | 跨机器 bot 互联 | 对端须开 api_server；同机多 profile 优先 A2A |

同机多 profile 团队的最佳分工：**委派流量走 A2A，飞书群只承担人在环协作**。

---

## 十二、真实案例存档

1. **单向不通（2026-07-05 小恩↔小葵）**：outbound 全部 code:0、mentions 命中，但接收方 0 inbound —— 根因是小葵没被拉进群。拉群即愈，不要去加重试。
2. **静默丢消息（2026-09-06 amedia）**：`FEISHU_ALLOWED_USERS=open` 导致白名单 `{"open"}`，真人 DM 全丢且日志几乎无痕。教训：改 .env 前列出所有相关字段一起看，改完重启并查 `/proc/<pid>/environ`。
3. **卡片点击无效（amedia）**：修好了 DM 闸门但漏了 GROUP_POLICY（disabled），批准卡全被拒。两套闸门必须一起修。
4. **NO_REPLY 气泡（2026-10-16 群协作五轮实测）**：被 @ 的回合回裸 NO_REPLY → 网关拦成警告气泡刷屏。沉默要靠「不 @ 就不投递」的机制设计。

**来源：** skills/devops/hermes-multi-profile-feishu（生产验证 2026-06-23 ~ 2026-10-16）；飞书开放平台协议限制（bot DM 不可达）为平台侧实测结论。

## 相关页面

- [[Hermes Agent Workforce]] —— 多 profile 团队的组织方式
- [[Hermes]] —— agent 框架本体
- [[飞书文档]] —— 飞书开放平台文档类 API
