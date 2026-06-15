# 新版网页播放信息

> 当前状态：2026-06-15 实测验证。`getRoomPlayInfo` 主体公开 GET 返回 `code:0`；编码参数矩阵已在在播房间（room_id=21622811，含 avc/hevc/av1 全编码）上逐项抓取验证。

## 调用地址

```text
GET https://api.live.bilibili.com/xlive/web-room/v2/index/getRoomPlayInfo
```

## 鉴权

公开 GET，通常不需要 Cookie。带登录 Cookie（`SESSDATA`）可在部分场景降低 `-352` 风控概率并解锁更高 `qn`。

## 参数

| 字段 | 必选 | 传递方式 | 类型 | 说明 |
| --- | --- | --- | --- | --- |
| `room_id` | true | GET | int | 直播房间号。 |
| `protocol` | false | GET | string | 协议集合，逗号分隔。`0`=http_stream，`1`=http_hls。 |
| `format` | false | GET | string | 容器格式集合，逗号分隔。`0`=flv，`1`=ts，`2`=fmp4。 |
| `codec` | false | GET | string | 视频编码集合，逗号分隔。`0`=avc，`1`=hevc，`2`=av1。 |
| `qn` | false | GET | int | 清晰度，`0` 表示自动（按可用档位就近选择）。 |
| `platform` | false | GET | string | 网页端常见 `web`。 |
| `ptype` | false | GET | int | 网页端常见 `8`。 |
| `dolby` | false | GET | int | 杜比相关参数，网页端常见 `5`。 |
| `panoramic` | false | GET | int | 全景相关参数，网页端常见 `1`。 |

## 编码参数取值（2026-06-15 实测验证）

下表的 `*_name` 列为返回 JSON 中 `data.playurl_info.playurl.stream[].protocol_name` / `format[].format_name` / `codec[].codec_name` 的实际取值。

### protocol（传输协议）

| 值 | `protocol_name` | 说明 | 实测 |
| --- | --- | --- | --- |
| `0` | `http_stream` | 单一 URL 直推流（FLV）。 | ✅ |
| `1` | `http_hls` | HLS 实时 m3u8 目录（ts / fmp4）。 | ✅ |
| 其它 | — | 不返回任何流。 | ✅ `protocol=2` 返回 `stream` 为空 |

### format（容器格式）

| 值 | `format_name` | 说明 | 实测 |
| --- | --- | --- | --- |
| `0` | `flv` | 单 URL，绑定 `http_stream`。 | ✅ |
| `1` | `ts` | 基于 m3u8，绑定 `http_hls`。 | ✅ |
| `2` | `fmp4` | 基于 m3u8，绑定 `http_hls`。 | ✅ |
| 其它 | — | **任何**非法 format 编号会导致整体不返回流（含混入合法值时）。 | ✅ `format=3`、`format=0,3`、`format=0,1,2,3` 全部返回空 |

### codec（视频编码）

| 值 | `codec_name` | 说明 | 实测 |
| --- | --- | --- | --- |
| `0` | `avc` | H.264，全容器可用。 | ✅ |
| `1` | `hevc` | H.265，flv/ts/fmp4 均可用。 | ✅ |
| `2` | `av1` | **仅在 fmp4 容器中出现**。 | ✅ `codec=2` 仅返回 `(http_hls, fmp4, av1)` |
| 其它 | — | 非法编号被忽略，仍返回其它合法编码的流。 | ✅ `codec=0,1,9` 返回 avc+hevc；`codec=9` 单独请求返回空 |

### protocol × format × codec 组合矩阵（实测）

请求 `protocol=0,1&format=0,1,2&codec=0,1,2` 在多编码房间返回的全部组合：

| protocol | format | codec |
| --- | --- | --- |
| http_stream | flv | avc |
| http_stream | flv | hevc |
| http_hls | ts | avc |
| http_hls | ts | hevc |
| http_hls | fmp4 | avc |
| http_hls | fmp4 | hevc |
| http_hls | fmp4 | av1 |

关键结论：
- **protocol 与 format 强绑定**：flv 只出现在 http_stream；ts / fmp4 只出现在 http_hls。
- **av1 仅出现在 fmp4**，flv / ts 下不会返回 av1。
- **flv 不提供 av1**；ts 提供 avc / hevc。

### qn（清晰度）

`current_qn` 为本次实际选定档位，`accept_qn` 列出该流可切换的全部档位。

| 请求 `qn` | 返回 `current_qn` | 行为 |
| --- | --- | --- |
| `10000` | `10000` | 命中档位则原样返回。 |
| `400` | `400` | 命中。 |
| `250` | `250` | 命中。 |
| `150` / `80`（房间无此档） | `250` | **就近回退**到最接近的可用档（非空返回）。 |
| `20000` / `30000`（超出可用） | `10000`（封顶到最高可用） | 回退到最高可用档。 |
| `0` | 服务端默认档（实测为 `400`） | 自动选档。 |

> 修正：原猜测「qn 指定时仅返回有对应质量的选项」不完全准确。实测中 `accept_qn` 始终返回该流的全部可用档位，`qn` 仅决定 `current_qn`（选定档位），不会裁剪 `accept_qn`；请求不存在的档位会就近回退而非返回空。

### qn 档位对照（`g_qn_desc`，实测）

| `qn` | `desc` |
| --- | --- |
| `30000` | 杜比 |
| `20000` | 4K |
| `15000` | 2K |
| `10000` | 原画 |
| `400` | 蓝光 |
| `250` | 超清 |
| `150` | 高清 |
| `80` | 流畅 |

## 已验证示例

```text
https://api.live.bilibili.com/xlive/web-room/v2/index/getRoomPlayInfo?room_id=21622811&protocol=0,1&format=0,1,2&codec=0,1,2&qn=10000&platform=web&ptype=8&dolby=5&panoramic=1
```

## 返回

返回 JSON。`code:0` 时，2026-06-15 验证到的 `data` 顶层字段包括：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `room_id` | int | 房间号。 |
| `short_id` | int | 短号。 |
| `uid` | int | 主播 UID。 |
| `live_status` | int | 直播状态，`1`=直播中，`0`=未开播。 |
| `playurl_info` | object/null | 播放 URL 信息，未开播或无权限时为 `null`。 |

`playurl_info` 非空时的关键嵌套结构（实测）：

```text
playurl_info
└─ playurl
   ├─ cid                   # = room_id
   ├─ g_qn_desc[]           # qn → 清晰度名称对照
   ├─ dolby_qn              # 杜比档位，无则 null
   ├─ p2p_data              # {p2p, p2p_type, m_p2p, m_servers}
   └─ stream[]              # 按 protocol 分组
      ├─ protocol_name      # http_stream / http_hls
      └─ format[]           # 按 format 分组
         ├─ format_name     # flv / ts / fmp4
         └─ codec[]         # 按 codec 分组
            ├─ codec_name   # avc / hevc / av1
            ├─ current_qn   # 本次选定档位
            ├─ accept_qn[]  # 可切换的全部档位
            ├─ base_url     # 含清晰度标识的路径，以 ? 结尾
            └─ url_info[]   # 同一路数的多个 CDN 候选
               ├─ host      # CDN 域名（https://...）
               ├─ extra     # 鉴权 query（expires/deadline/qn/trid/sign 等）
               └─ stream_ttl
```

### 实际可播放 URL 的拼接（实测）

完整流地址 = `url_info[i].host` + `base_url` + `url_info[i].extra`：

```text
https://ec-jssz-ct-01-15.bilivideo.com           # url_info[0].host
/live-bvc/706563/live_21622811_bs_300163_bluray.flv?   # base_url（已带尾部 ?）
expires=1781493333&pt=web&deadline=...&qn=10000&trid=...&sign=...  # extra
```

- `url_info[]` 通常给出**多个 CDN 候选**（同一路流的不同 host），可做容灾/择优。
- `extra` 含时效鉴权参数（`expires`/`deadline`/`sign`），URL 有有效期，过期需重新请求 `getRoomPlayInfo`。
- HLS（ts/fmp4）的 `base_url` 指向 m3u8；FLV 的 `base_url` 直接是流文件。

### data 顶层附加字段（实测）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `all_special_types` | array | 特殊类型标记，常为空数组。 |
| `degraded_playurl` | object/null | 降级播放信息，正常为 `null`。 |
| `multi_screen_info` | string/object | 多机位信息，无则空串。 |
| `subtitle_cfg` | object/null | 字幕配置。 |
| `risk_with_delay` | int | 风控延迟标记，`0`=正常。 |
| `relay_room_id` | int | 转播源房间号，`0`=非转播。 |
| `official_type` / `official_room_id` | int | 官方房间标记。 |
| `is_hidden` / `is_locked` / `is_portrait` | bool | 隐藏/锁定/竖屏。 |
| `encrypted` / `pwd_verified` | bool | 房间加密/密码已验证。 |

## 备注

- 旧文档 `API.RealRoom.md` 记录的是 `room/v1/Room/playUrl`，本接口是当前网页端更常见的版本。
- 历史验证中部分环境曾返回 `code:-352`，但默认参数验证为 `code:0`。`-352` 为风控/上下文问题，可带登录 Cookie 缓解。
- `live_status=0`（未开播）时 `playurl_info` 为 `null`，无法验证编码矩阵；编码取值需在 `live_status=1` 的房间验证。
- 不同房间提供的编码档位差异很大：普通房间常只有 `flv/ts + avc + 单一 qn`；官方赛事/大主播房间才会提供 `hevc/av1/fmp4 + 多 qn`。
