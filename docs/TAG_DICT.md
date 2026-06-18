# 受控标签词表（TAG_DICT）

> 版本：v0.1 · 2026-06-18
> 配套：[SYSTEM_DESIGN.md](./SYSTEM_DESIGN.md) §4.4 + [BADMINTON_SPEC.md](./BADMINTON_SPEC.md)
> 定位：所有 `tags` 字段的可选值在这里登记。新增标签必须先进这份词表。

---

## 0. 标签命名规则

格式：**`<ns>/<key>`**，全小写，连字符分词，不带空格。

| ns | 性质 | 词表 | 是否允许新词 |
|---|---|---|---|
| `category/` | 闭集 | 系统硬编码 | ❌ 不可加 |
| `event/` | 闭集 | 系统硬编码 | ❌ 不可加 |
| `play/` | 受控 | 见 §2 | ✅ 走流程加 |
| `style/` | 受控 | 见 §3 | ✅ 走流程加 |
| `topic/` | 自由 | 用户写 | ✅ 任意加 |
| `quality/` | 闭集 | 系统硬编码 | ❌ 不可加 |

**新词加入流程**（play/style）：
1. 想加之前先看现有词表是否已有近义词
2. 在本文件相应小节加一行（含中文释义、典型用法）
3. 在 commit message 写明 `tag: add play/<key>` 或 `tag: add style/<key>`

---

## 1. category/ · 品类（闭集）

| 标签 | 中文 | 适用对象 |
|---|---|---|
| `category/badminton` | 羽毛球 | Asset / Event |
| `category/cycling` | 骑行 | Asset / Event |
| `category/daily` | 日常 | Asset |
| `category/trip` | 旅行 | Event |
| `category/portrait` | 人像 | Asset |
| `category/landscape` | 风景 | Asset |

---

## 2. play/ · 羽毛球技战术（受控）

### 2.1 击球类型（与 BADMINTON_SPEC §3.1 last_shot_type 对齐）

| 标签 | 中文 | 何时打 |
|---|---|---|
| `play/smash` | 杀球 | 决胜球或重要球是杀球时 |
| `play/drop` | 吊球 | 软吊 |
| `play/clear` | 高远球 | 后场对后场拉吊 |
| `play/drive` | 平抽 | 中场快速对抗 |
| `play/push` | 推球 | 推到对方后场 |
| `play/net-shot` | 网前小球 | 搓 / 放小球 |
| `play/lift` | 挑高球 | 防守挑后场 |
| `play/block` | 接杀 | 防守抵挡杀球 |

### 2.2 回合特征

| 标签 | 中文 | 触发条件 |
|---|---|---|
| `play/long-rally` | 多拍长回 | shot_count > 15 |
| `play/short-rally` | 短回合 | shot_count ≤ 4 |
| `play/clutch-point` | 关键分 | 18-20 / 20-20 / 决胜局 11+ |
| `play/break-point` | 破发分 | 接发方得分 |

### 2.3 区域 / 站位

| 标签 | 中文 |
|---|---|
| `play/forehand` | 正手 |
| `play/backhand` | 反手 |
| `play/front-court` | 前场 |
| `play/mid-court` | 中场 |
| `play/back-court` | 后场 |

### 2.4 防守与救球

| 标签 | 中文 |
|---|---|
| `play/save` | 精彩救球 |
| `play/dive` | 鱼跃 |
| `play/recover` | 回中 |

---

## 3. style/ · 主观风格（受控）

| 标签 | 中文 | 释义 |
|---|---|---|
| `style/violent-smash` | 暴力扣杀 | 出手凶、落点重、对手没碰到 |
| `style/old-school` | 老登球路 | 拉吊为主、绵里藏针、节奏慢 |
| `style/aggressive` | 进攻型 | 全场压制 |
| `style/defensive` | 防守反击 | 多挑多接，等机会 |
| `style/sneaky` | 阴招 | 假动作、变速、骗对手 |
| `style/highlight-reel` | 集锦回合 | 看回放都觉得过瘾 |
| `style/blunder` | 翻车 | 自己一顿失误 |

---

## 4. topic/ · 用户专题（自由）

格式建议：`topic/<时间-地点-人物-事件>`，全部信息一目了然。

**示例**（仅作参考，用户自定义）：

| 标签 |
|---|
| `topic/2026-青海骑行` |
| `topic/2026-夏-儿子毕业` |
| `topic/2026-橙天羽毛球` |
| `topic/2026-与老王对决` |

---

## 5. quality/ · 质量标记（闭集）

| 标签 | 中文 | 含义 |
|---|---|---|
| `quality/highlight` | 精彩 | 候选集锦 |
| `quality/blur` | 模糊 | 失焦/抖动 |
| `quality/duplicate-near` | 近似重复 | phash 聚类提示 |
| `quality/low-light` | 弱光 | 室内灯光差 |
| `quality/best-of-burst` | 连拍最佳 | 连拍组里挑出来的 |

---

## 6. event/ · 事件级（闭集）

| 标签 | 中文 |
|---|---|
| `event/match` | 正式对阵 |
| `event/training` | 训练课 |
| `event/warmup` | 热身 |
| `event/group-photo` | 合影 |
| `event/group-meal` | 聚餐 |
| `event/award` | 颁奖 |

---

## 7. 修订历史

### v0.1 · 2026-06-18
- 初稿：定义 6 个命名空间（category/event/play/style/topic/quality）。
- play/* 给出 8 个击球类型 + 4 个回合特征 + 5 个区域 + 3 个防守标签。
- style/* 给出 7 个主观风格标签（含暴力扣杀、老登球路）。
- 明确"新词加入流程"：先查近义词，再在本文件登记，commit message 注明。
