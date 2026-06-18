# 羽毛球专项规范（BADMINTON_SPEC）

> 版本：v0.2 · 2026-06-18
> 配套：[SYSTEM_DESIGN.md](./SYSTEM_DESIGN.md) v1.3
> 定位：把羽毛球片段从"一堆 mp4"变成"可结构化检索、可统计、可剪辑的标注库"。

---

## 0. 一句话目标

每段羽毛球视频，最终产出一份 **labels.jsonl** —— 每行一个 Rally（回合）的完整标注，含**文件定位 + 起止时间 + 羽毛球关键数据**三组信息。这份 jsonl 就是 NAS-MGR 整个羽毛球域的真值源。

---

## 1. 数据层级（再确认）

```
Session   一次约球（2-3 小时，一个场馆，一群人）
   └─ Match    一场对阵（单打/双打，一对阵容打一场）
        └─ Game    一局（21x3 / 15x3 / 11x1）
             └─ Rally   一回合 ← 标注的最小单元，落在 labels.jsonl 一行
```

视频文件（Asset）与 Match/Game 是 **N:N**：
- 一段 7-12 分钟视频可能跨越两个 Game
- 一场 Match 可能拆成多段视频
- 用 `match_id` + `game_idx` 把它们粘起来

---

## 2. labels.jsonl 行格式（标注最小单元）

每行一个 Rally，**JSON Lines** 格式（一行一个对象，无外层数组）。字段分四组：

### 2.1 文件定位（必填）

| 字段 | 类型 | 必填 | 含义 | 示例 |
|---|---|---|---|---|
| `id` | string | ✓ | 该 Rally 的唯一 ID（`<sha6>-r<rally_idx>` 自动生成）| `"f3a92c-r017"` |
| `asset_path` | string | ✓ | NAS 相对路径（相对挂载根）| `"badminton/2026-06-15-橙天/court3-game1.mp4"` |
| `asset_sha256` | string | ✓ | 视频文件 sha256，作为永久身份 | `"f3a92c8b…"` |

> path 可变（用户挪文件），sha256 不变。两者都存，下次扫描自动对齐。

### 2.2 时间定位（必填）

| 字段 | 类型 | 必填 | 含义 | 单位/精度 |
|---|---|---|---|---|
| `start_ts` | number | ✓ | 回合起点（视频内秒数）| 秒，1 位小数（如 `38.5`）|
| `end_ts` | number | ✓ | 回合终点 | 秒，1 位小数 |
| `duration` | number | 自动计算 | end - start | 秒 |

### 2.3 比赛上下文（必填）

| 字段 | 类型 | 必填 | 含义 | 示例 |
|---|---|---|---|---|
| `session_id` | string | ✓ | 同一次约球用同一个 ID（`yyyy-mm-dd-place-slug`）| `"2026-06-15-orange-court"` |
| `match_id` | string | ✓ | 同一场对阵 | `"2026-06-15-m01"` |
| `game_idx` | int | ✓ | 第几局（1-based）| `1` |
| `rally_idx` | int | ✓ | 全场 Rally 连续编号 | `17` |
| `match_type` | enum | ✓ | `singles`/`doubles`/`mixed` | `"doubles"` |
| `side_a` | string[] | ✓ | A 方球员名（短名）| `["KH", "老王"]` |
| `side_b` | string[] | ✓ | B 方球员名 | `["小李", "老张"]` |

### 2.4 羽毛球关键数据

#### 2.4.1 战术事实（必填，客观可观察）

| 字段 | 类型 | 取值 | 含义 |
|---|---|---|---|
| `server` | string | A方/B方某球员名 | 发球人（自带方位信息）|
| `serve_side` | enum | `deuce`(右半区) / `ad`(左半区) | 发球区 |
| `score_before` | `[a,b]` | 0-30 | 回合开始前本局比分 |
| `score_after` | `[a,b]` | 0-30 | 回合结束后本局比分 |
| `winner_side` | enum | `A` / `B` | 得分方 |
| `winner_player` | string | 球员名 / `null` | 谁打死的（双打才填）|
| `losing_reason` | enum | `winner`(得分球直接打死) / `out`(对方出界) / `net`(对方下网) / `unforced_error`(对方失误) | 得分原因 |
| `shot_count` | int | 1-100 | 这回合多少拍 |
| `last_shot_type` | enum | 见 §3 | 决胜球技术类型 |

#### 2.4.2 主观评价（选填，复盘价值）

| 字段 | 类型 | 含义 |
|---|---|---|
| `rally_quality` | int 1-5 | 回合精彩度（5=神回合）|
| `tactical_pattern` | enum | 见 §3 |
| `error_type` | enum | `technique` / `tactic` / `physical` / `mental` |
| `learning_note` | string | 文字复盘（短）|
| `opponent_pattern` | string[] | 对手习惯标签 `["反手弱","挑球高"]` |

#### 2.4.3 标签 + 集锦标记（选填）

| 字段 | 类型 | 含义 |
|---|---|---|
| `tags` | string[] | 命名空间标签（见 TAG_DICT.md），如 `["style/violent-smash","play/long-rally"]` |
| `highlight` | bool | 是否进精彩集锦 |
| `note` | string | 自由备注 |

### 2.5 元数据（系统填）

| 字段 | 类型 | 含义 |
|---|---|---|
| `created_by` | enum | `ai` / `user` / `imported` |
| `confidence` | number 0-1 | AI 置信度（人工 = 1.0）|
| `created_at` | int | Unix ts |
| `updated_at` | int | Unix ts |

---

## 3. 受控枚举（schema-level）

### 3.1 `last_shot_type`（决胜球技术）

| 值 | 中文 | 释义 |
|---|---|---|
| `smash` | 杀球 | 大力扣杀 |
| `drop` | 吊球 | 软吊到前场 |
| `clear` | 高远球 | 后场对后场 |
| `drive` | 平抽球 | 平直快球 |
| `push` | 推球 | 推到对方后场 |
| `net_shot` | 搓/放小球 | 网前小球 |
| `lift` | 挑高球 | 网前挑到后场 |
| `block` | 抵挡 / 接杀 | 防守接对方杀球 |
| `serve` | 发球直接得分 | 罕见 |

### 3.2 `tactical_pattern`（战术模式）

| 值 | 中文 |
|---|---|
| `drive_battle` | 平抽挡 |
| `front_back` | 前后控制 |
| `four_corners` | 四点拉吊 |
| `attack_block` | 进攻防守 |
| `defensive_lift` | 被动挑高 |
| `serve_attack` | 接发抢攻 |

### 3.3 `losing_reason`

| 值 | 含义 |
|---|---|
| `winner` | 我方决胜球直接打死，对方碰不到 |
| `out` | 对方击球出界 |
| `net` | 对方下网 |
| `unforced_error` | 对方非受迫失误（接发漏接、空中失误等）|

---

## 4. labels.jsonl 完整示例

```jsonl
{"id":"f3a92c-r017","asset_path":"badminton/2026-06-15-橙天/court3-game1.mp4","asset_sha256":"f3a92c8b1e4a...","start_ts":402.3,"end_ts":423.8,"duration":21.5,"session_id":"2026-06-15-orange-court","match_id":"2026-06-15-m01","game_idx":1,"rally_idx":17,"match_type":"doubles","side_a":["KH","老王"],"side_b":["小李","老张"],"server":"老王","serve_side":"deuce","score_before":[8,8],"score_after":[9,8],"winner_side":"A","winner_player":"KH","losing_reason":"winner","shot_count":23,"last_shot_type":"smash","rally_quality":5,"tactical_pattern":"four_corners","tags":["style/violent-smash","play/long-rally"],"highlight":true,"note":"23拍来回，最后反手区扣直线","created_by":"user","confidence":1.0,"created_at":1750211400,"updated_at":1750211400}
{"id":"f3a92c-r018","asset_path":"badminton/2026-06-15-橙天/court3-game1.mp4","asset_sha256":"f3a92c8b1e4a...","start_ts":428.1,"end_ts":434.6,"duration":6.5,"session_id":"2026-06-15-orange-court","match_id":"2026-06-15-m01","game_idx":1,"rally_idx":18,"match_type":"doubles","side_a":["KH","老王"],"side_b":["小李","老张"],"server":"KH","serve_side":"ad","score_before":[9,8],"score_after":[9,9],"winner_side":"B","losing_reason":"unforced_error","shot_count":4,"last_shot_type":"net_shot","rally_quality":2,"tags":["play/short-rally"],"highlight":false,"note":"接发推后场，老张反手吊网前我接漏","created_by":"user","confidence":1.0,"created_at":1750211460,"updated_at":1750211460}
```

---

## 5. 完整性约束（系统 lint 用）

- ∑rallies(match) → score_after.last 必须等于 match.final_score
- 同一 (match_id, game_idx) 内 rally_idx 必须连续递增
- 任意 rally 的 score_after - score_before 必须等于 1（恰好一方+1）
- end_ts > start_ts；多个 rally 的时间区间在同一 asset 内不重叠
- highlight=true 的 rally 应有 rally_quality ≥ 4

---

## 6. 标注工作流（先纸上跑通）

在系统未上线前，可以**先用 Excel/手写 jsonl** 跑通流程：

```
[1] 看视频 → 用脑子或纸笔记下每个回合的：
    起止秒数 / 谁发球 / 比分 / 谁赢 / 怎么赢 / 多少拍 / 决胜球类型 / 是否精彩

[2] 一行一个 Rally 写到 labels.jsonl

[3] 系统上线后直接吃这份 jsonl，无需重新标注
```

> **关键**：jsonl 是真值，catalog.db 只是它的索引视图。从今天开始攒 jsonl 就能开始积累，不用等系统。

### 6.1 片段提取与标注的交互流程（系统上线后）

详见 [SYSTEM_DESIGN.md](./SYSTEM_DESIGN.md) §6.4。标注一个回合的完整交互：

```
[A] 时间轴拖选 [start, end] → 浏览器对 proxy 区间播放（0 秒，不切文件）
[B] 微调首尾点确认 → 弹出标注面板
[C] 填关键数据 + 打标签 → 追加一行到 labels.jsonl
[D]（可选）需要原画质成片时 → 从 NAS 原片精切到 Exports/
```

绝大多数标注止于 [C]：jsonl 已存精确起止点，原片随时可重切，不必每次导出。

### 6.2 标注面板智能预填（让确认更快）

标注新回合时，面板按以下规则自动预填，用户只需改少数字段：

| 字段 | 预填来源 |
|---|---|
| `start_ts / end_ts` | 时间轴选区直接带入 |
| `session_id / match_id` | 从当前视频所属 Event 继承 |
| `match_type / side_a / side_b` | 整场不变，首回合填完后续全继承 |
| `rally_idx` | 自动 = 上一回合 + 1 |
| `score_before` | **自动 = 上一回合的 score_after（链式推导）** |
| `score_after` | 标完"谁赢"后自动给赢方 +1 |
| `server` | 按羽毛球规则（得分方继续发球）自动推荐 |
| `tags` | 受控词表自动补全（输"暴"→ style/violent-smash）|

> **比分链式推导**是核心加速器：用户只需标"谁赢了这回合"，整局比分线自动连起来，并可用 §5 的 lint 规则自检（score_after 增量必须 = +1）。

---

## 7. 关键查询（系统层提供）

| 查询意图 | 实现 |
|---|---|
| "暴力扣杀合集" | `WHERE tags LIKE '%style/violent-smash%' ORDER BY rally_quality DESC` |
| "和老王打的所有比赛" | `WHERE side_a CONTAINS '老王' OR side_b CONTAINS '老王' AND match_type='doubles'` |
| "本月多拍长回" | `WHERE shot_count > 15 AND captured_at >= 月初` |
| "我反手区被打死的回合" | `WHERE losing_reason='winner' AND winner_side != 我方 AND tags CONTAINS 'play/backhand'` |
| "胜率统计" | `GROUP BY (side_a, side_b) → COUNT(winner_side='A')/COUNT(*)` |
| "扣杀成功率" | `last_shot_type='smash' 中 winner_side=我方 的占比` |

---

## 8. 实施优先级

| 优先级 | 内容 | 何时做 |
|---|---|---|
| **P0a** | 把 §2 字段冻结、TAG_DICT.md 词表冻结 | 现在 |
| **P0b** | 写 1-2 段视频的 labels.jsonl，验证字段够用 | 本周内（手写）|
| **P1** | 标注 UI（H1 键盘流） | P2 阶段 |
| **P2** | AI 切回合候选（启发式） | P2 阶段 |
| **P3** | 集锦剪辑（按 tags / rally_quality 过滤）| P4 阶段 |

---

## 9. 修订历史

### v0.2 · 2026-06-18
- 新增 §6.1 片段提取与标注交互流程（A 区间播放 → B 确认 → C 写 jsonl → D 可选精切）。
- 新增 §6.2 标注面板智能预填规则（含比分链式推导）。
- 配套 SYSTEM_DESIGN §6.4。

### v0.1 · 2026-06-18
- 初稿：定义 labels.jsonl 行格式（4 组字段：文件定位 / 时间 / 上下文 / 关键数据）。
- 确立"jsonl 是真值，catalog.db 只是视图"的原则。
- 给出 last_shot_type / tactical_pattern / losing_reason 三个受控枚举。
- 给出完整性约束（lint 规则）。
- 给出未上线前用 Excel 也能开始攒数据的工作流。
