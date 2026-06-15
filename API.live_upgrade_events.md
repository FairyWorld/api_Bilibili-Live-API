# 直播间用户升级 / 等级类推送事件

> 当前状态：2026-06-15 通过 WBI 签名 `getDanmuInfo` + WebSocket（op 7 鉴权，protover 2 zlib）实测抓取验证。本页记录用户荣耀（财富）等级升级、粉丝灯牌升级以及相关的等级 / 身份类推送事件。

## 连接前置

升级类事件都通过弹幕长连接（WebSocket）推送，连接链路与 [`API.getDanmuInfo.md`](./API.getDanmuInfo.md)、[`API.WebSocket.md`](./API.WebSocket.md) 一致：

1. `nav` 取 WBI key → 对 `getDanmuInfo` 签名，拿到 `token` + `host_list`。
2. `wss://{host}:{port}/sub`，发送 op `7` 鉴权包。
3. 实测鉴权包必须字段：`{uid, roomid, protover, platform:"web", type:2, key:<token>, buvid:<buvid3>}`。
   - `uid=0` 匿名实测被拒（close 1006）；需带登录态 `uid`（由 `nav.data.mid` 获取）+ `buvid3`（由 `x/frontend/finger/spi` 的 `b_3` 获取）。
   - `protover=2` 走 zlib 压缩（避免 protover 3 的 brotli 依赖）。
4. 鉴权成功收到 op `8`（AUTH_REPLY），随后开始收到 op `5`（业务消息）。

## 一、核心升级事件

### 1. 用户荣耀 / 财富等级升级

B 站「荣耀等级」即财富等级（wealth level）。等级数据随 `ENTRY_EFFECT`、`SEND_GIFT` 等事件以 `wealthy_info` / `wealth` 字段携带；升级本身另有低频专项推送。

实测 `ENTRY_EFFECT.data.wealthy_info` 结构（room 21622811 抓取）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `uid` | int | 用户 UID。 |
| `level` | int | 当前荣耀（财富）等级，实测值 `30`。 |
| `level_total_score` | int | 等级总积分。 |
| `cur_score` | int | 当前积分。 |
| `upgrade_need_score` | int | 升到下一级还需的积分（升级进度依据）。 |
| `status` | int | 状态。 |
| `dm_icon_key` | string | 弹幕图标 key。 |

同包 `uinfo.wealth` 亦给出 `{level, dm_icon_key}`。

> 升级触发的专项广播 cmd（如 `USER_TOAST_MSG` 续费类、财富等级升级提示）为低频事件，本轮多房间累计抓取窗口内未触发；其等级字段口径与上表 `wealthy_info` 一致。

### 2. 粉丝灯牌（粉丝勋章）升级

粉丝灯牌等级随用户互动事件携带，主要载体为 `INTERACT_WORD` / `INTERACT_WORD_V2` 的 medal 字段、`ENTRY_EFFECT.uinfo.medal`。

实测 `ENTRY_EFFECT.data.uinfo.medal` 结构：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | string | 灯牌名（主播粉丝团名），实测 `"质检团"`。 |
| `level` | int | **灯牌等级**，实测值 `21`。 |
| `score` | int | 当前亲密度积分，实测 `2433`（升级进度依据）。 |
| `id` | int | 勋章 ID。 |
| `ruid` | int | 勋章所属主播 UID。 |
| `guard_level` | int | 大航海等级（`0`=无，`1`=总督，`2`=提督，`3`=舰长）。 |
| `is_light` | int | 灯牌是否点亮（`1`/`0`）。 |
| `color` / `color_start` / `color_end` / `color_border` | int | 旧版颜色（十进制）。 |
| `v2_medal_color_*` | string | 新版颜色（`#RRGGBBAA` 字符串）。 |
| `guard_icon` / `honor_icon` | string | 大航海 / 荣耀图标。 |

实测 `INTERACT_WORD.data.fans_medal`（明文 JSON，仍在用）字段：`anchor_roomid, guard_level, icon_id, is_lighted, medal_color*, medal_level, medal_name, score, special, target_id`。其中 `medal_level` 为灯牌等级，`score` 为亲密度。

> 2026 变化：`INTERACT_WORD_V2` 已改为 **protobuf 编码**，`data` 仅含一个 base64 的 `pb` 字段，需按 protobuf wire-format 解析（无明文 JSON）。

实测 `INTERACT_WORD_V2.pb` 解出的字段（room 545068 抓取）：

| protobuf field | 含义 | 实测值 |
| --- | --- | --- |
| `1` | uid | `168461531` |
| `2` | uname | `深空z` |
| `5` | msg_type | `1`（进入） |
| `6` | roomid | `545068` |
| `7` | timestamp | `1781489107` |
| `9` | 粉丝勋章子结构 | 见下 |
| `9.2` | medal_level（灯牌等级） | `31` |
| `9.3` | medal_name | `德云色` |
| `9.13` | score（亲密度） | `34134` |

`msg_type`：`1`=进入，`2`=关注，`3`=分享（与 `INTERACT_WORD` 一致）。

## 二、相关等级 / 身份类事件

| CMD | 说明 | 升级相关字段 | 抓取状态 |
| --- | --- | --- | --- |
| `ENTRY_EFFECT` | 入场特效（舰长 / 高等级用户）。 | `wealthy_info.level`、`uinfo.medal.level`、`uinfo.guard.level` | ✅ 实测 |
| `INTERACT_WORD` | 进入 / 关注 / 分享（明文 JSON）。 | `fans_medal.medal_level`、`fans_medal.guard_level` | ✅ 实测 |
| `INTERACT_WORD_V2` | 同上，protobuf 编码。 | pb field 9（medal_level / score） | ✅ 实测 |
| `GUARD_BUY` | 开通 / 续费大航海。 | `guard_level`（1 总督 / 2 提督 / 3 舰长） | 文档；低频未抓到 |
| `USER_TOAST_MSG` | 上舰 / 续费 Toast 广播。 | `guard_level`、`role_name` | 文档；低频未抓到 |
| `GUARD_HONOR_THOUSAND` | 千舰荣耀（主播侧荣誉）。 | 千舰达成状态 | 文档；未抓到 |
| `ONLINE_RANK_V3` | 高能用户榜（**protobuf 编码**）。 | 榜内用户等级 / 排名 | ✅ 实测（pb） |
| `NOTICE_MSG` | 全站 / 房间广播（含礼物、上舰等）。 | `msg_common`、`msg_self`、`roomid` | ✅ 实测 |
| `SUPER_CHAT_MESSAGE` | 醒目留言。 | `user_info`、`price` | 文档 |

## 三、抓取实测发现的额外 2026 事件（仓库原文档未记录）

90 秒 × 10 房间累计窗口观测到的、`API.WebSocket.md` 现有列表未覆盖的 cmd：

| CMD | 推测含义 | 备注 |
| --- | --- | --- |
| `UNIVERSAL_ASR_TEXT` | 语音实时字幕（ASR）。 | 高频，语音 / 电台区密集出现。 |
| `UNIVERSAL_EVENT_GIFT_V2` | 通用活动礼物事件 V2。 | 与活动 / 礼物联动。 |
| `INTERACT_WORD_V2` | 互动事件 protobuf 版。 | 已逐步替代明文 `INTERACT_WORD`。 |
| `ONLINE_RANK_V3` | 高能榜 V3，protobuf 编码。 | 替代 `ONLINE_RANK_V2`。 |
| `RANK_CHANGED_V2` | 榜单变动 V2。 | 高频。 |
| `DM_INTERACTION` | 弹幕互动聚合（合并多条互动）。 | — |
| `PK_INFO` | PK 实时信息。 | PK 进行中房间。 |
| `POPULARITY_CHANGE` | 人气值变化。 | — |

## 四、同一动作多 cmd 并发

与 `API.WebSocket.md` 备注一致：一次业务动作常同时推送多个 cmd。例如上舰会同时推送 `GUARD_BUY` + `USER_TOAST_MSG` + `NOTICE_MSG`；投喂礼物会推送 `SEND_GIFT` +（达到广播条件时）`NOTICE_MSG` + `COMBO_SEND`。消费方需按 `cmd` 去重 / 合并。

## 五、注意

- 升级专项 cmd（`GUARD_BUY` / `USER_TOAST_MSG` / 财富等级升级提示 / 灯牌升级提示）属低频事件，需长时间挂在高活跃房间才能稳定抓到；本页等级 **字段口径** 以高频载体事件（`ENTRY_EFFECT` / `INTERACT_WORD*`）实测为准。
- `INTERACT_WORD_V2`、`ONLINE_RANK_V3` 等新版事件已 protobuf 化，明文 JSON 解析会失败，必须按 wire-format 解码。
- protover 3（brotli）是 Web 端默认压缩；若运行环境无 brotli，可在鉴权包用 protover 2 改走 zlib。
- 抓取需登录态（`uid` + `buvid3`），匿名连接被风控拒绝。
