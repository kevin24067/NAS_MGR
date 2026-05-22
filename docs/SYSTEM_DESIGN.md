# NAS-MGR 系统设计文档

> 版本：v1.0 · 定稿日期：2026-05-22
> 状态：方案定型，待进入实施分期 P0
> 维护者：KH

---

## 0. 一句话目标

把分散在 NAS 上的 5TB 照片视频（羽毛球 / 骑行 / 日常 / 旅行）从"文件堆"变成"可按时间·地点·人物·类别·专题随意检索、可对羽毛球视频做结构化复盘、可一键导出主题集锦"的私人媒体管理系统。

---

## 1. 决策定型（Decision Record）

| 决策项 | 选择 | 含义 |
|---|---|---|
| **A. 部署形态** | A2 | NAS 只做存储，所有服务（API、Worker、AI）跑在 Mac 上 |
| **B. 元数据库位置** | B1 | SQLite 主库 `catalog.db` 存放在 NAS 的 `index/` 目录，单点真理 |
| **C. 入库触发** | C2 | 显式 `nasmgr import` 命令触发，可控可排查；后续可演进到 watch |
| **D. 照片栈选型** | D2 | Immich 接管日常照片/视频的浏览与人脸；NAS-MGR 自研羽毛球/骑行/专题层 |
| **E. 羽毛球 AI 程度** | E1 | AI 只生成"回合切分候选"，比分/发球人/得分方/技术标签全部走人工标注 |
| **F. 地图服务** | F1 | 高德地图 API（在线，需 GCJ02 纠偏） |
| **G. 前端形态** | G1 | 单 Web SPA |
| **H. 标注 UI** | H1 | Web 端键盘流（空格/数字键/ASD 等快捷键） |

锁定这 8 个决策后，本文档其余部分的所有约束都是这套组合的直接推论。

---

## 2. 总体架构

### 2.1 拓扑

```
┌──────────────────────────────────────────────────────────────────────┐
│                              NAS（存储 + 元数据）                       │
│   /media/raw/         5TB 原始素材，只读                                │
│   /media/inbox/       待入库暂存区                                       │
│   /media/derived/     缩略图 / 代理片 / 关键帧 / 导出                    │
│   /media/index/       catalog.db（SQLite 主库） + 向量索引              │
└────────────┬─────────────────────────────────────────────────────────┘
             │ SMB 挂载（autofs）→ /Volumes/nas-media
             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                            Mac（计算 + UI）                            │
│   nasmgr-api    FastAPI · 暴露 REST/JSON                               │
│   nasmgr-worker AI/转码/扫描的后台 worker                                │
│   nasmgr-web    Vue/Svelte SPA · 浏览 + 标注 + 导出                     │
│   immich        Docker 起的 Immich，接管日常照片/视频与人脸             │
│   高德 API      地理编码 / 反查 / GCJ02                                 │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 服务边界

| 服务 | 职责 | 不做的事 |
|---|---|---|
| **nasmgr-api** | 业务读写、查询、专题、导出 | 不直接跑 AI（投递到 worker） |
| **nasmgr-worker** | 扫描入库、转码、AI 嵌入、人脸、回合切分 | 不暴露外部接口，只消费任务 |
| **nasmgr-web** | 浏览/搜索/羽毛球标注/集锦剪辑 | 不直连 NAS 文件，统统过 API |
| **immich**（D2 混合） | 日常照片视频展示、人脸识别、移动端同步 | 不参与羽毛球结构化标注 |

> Immich 与 NAS-MGR 各管各的库，**通过同一份 raw/ 物理路径共享素材**。两边的元数据库不打通；用户在浏览器里有"日常→Immich，赛事→NAS-MGR"的清晰心智。

---

## 3. 存储约定

### 3.1 NAS 目录结构（最终态）

```
/media/
├── inbox/                           # 待入库（用户/同步工具往这里扔）
│   ├── phone/                       # 手机自动同步
│   ├── camera/                      # 相机/运动相机拷卡
│   ├── badminton/YYYY-MM-DD/        # 球场视频
│   └── cycling/YYYY-MM-DD/          # 骑行视频 + GPX
│
├── raw/                             # 已入库 · 系统管理 · 只读
│   ├── 2026/05/22/
│   │   ├── IMG_2034_<hash6>.heic
│   │   ├── VID_2034_<hash6>.mp4
│   │   └── badminton-court-A_<hash6>.mp4
│   └── ...
│
├── derived/                         # 衍生物 · 可重建可删
│   ├── thumbnails/{256,512,1024}/   # 三档缩略图
│   ├── proxies/                     # 视频 720p H.264 代理片
│   ├── keyframes/                   # 视频每 N 秒抽一帧
│   └── exports/                     # 集锦/合集导出
│
├── archive/                         # 同源低质副本降级存放
│
└── index/                           # 索引 · 全部可重建
    ├── catalog.db                   # SQLite 主库（FTS5 + sqlite-vec）
    ├── catalog.db-wal               # WAL
    └── vec/                         # 向量索引落盘
```

### 3.2 命名规则

- `raw/` 文件名：`<原文件主名>_<sha256前6位>.<ext>`（防重命名冲突，hash 锁身份）
- `derived/` 路径：与 raw 路径对齐，例 `derived/proxies/2026/05/22/<hash6>.mp4`
- 所有路径在数据库里存**相对 `/media/` 的相对路径**，便于换挂载点

### 3.3 备份与重建

| 内容 | 是否备份 | 说明 |
|---|---|---|
| `raw/` | 必须异地备份 | 原片不可再生，至少 RAID + 离线副本 |
| `derived/` | 不备份 | 全部可从 raw 重建 |
| `index/catalog.db` | 每日快照 | 备份用于快速恢复，但永远可从 raw + 标注导出文件重建 |
| 标注数据 | 每日导出 JSONL 到 `index/exports/labels/` | 哪怕 catalog.db 全毁，标注也不丢 |

---

## 4. 数据模型

### 4.1 核心实体一览

```
Asset       一个物理文件（照片或视频）
Event       活动聚合（badminton_session / badminton_match / cycling / trip / daily）
Segment     视频内时间片（rally / game_break / climb / descent / ...）
Person      人物
Place       地点（known + geocoded 双层）
Tag         扁平命名空间标签
Embedding   向量索引（CLIP 图文 / face）
SmartAlbum  专题（保存的查询规则）
Job         异步任务记录
```

### 4.2 表结构（SQLite DDL 草案，关键字段示意）

```sql
-- 4.2.1 Asset
CREATE TABLE asset (
  id            INTEGER PRIMARY KEY,
  path          TEXT NOT NULL UNIQUE,         -- 相对 /media/ 路径
  sha256        TEXT NOT NULL UNIQUE,
  phash         TEXT,                          -- 感知哈希，去重
  media_type    TEXT NOT NULL,                 -- photo|video
  mime          TEXT,
  size_bytes    INTEGER,
  width         INTEGER, height INTEGER,
  duration      REAL,                          -- 视频时长秒
  captured_at   INTEGER,                       -- Unix ts (EXIF)
  ingested_at   INTEGER NOT NULL,              -- 入库时间
  gps_lat       REAL, gps_lon REAL,            -- WGS84
  gps_lat_gcj   REAL, gps_lon_gcj REAL,        -- 国内显示用
  camera_make   TEXT, camera_model TEXT, lens TEXT,
  quality_score REAL,                          -- 启发式质量分
  canonical_id  INTEGER REFERENCES asset(id),  -- 同源副本主指针
  privacy_level INTEGER DEFAULT 0,             -- 0=normal, 1=high(skip AI)
  status        TEXT DEFAULT 'active'          -- active|archived|deleted
);
CREATE INDEX idx_asset_captured ON asset(captured_at);
CREATE INDEX idx_asset_gps ON asset(gps_lat, gps_lon);
CREATE INDEX idx_asset_canonical ON asset(canonical_id);

-- 4.2.2 Asset 路径别名（处理完全重复）
CREATE TABLE asset_path_alias (
  asset_id  INTEGER NOT NULL REFERENCES asset(id),
  path      TEXT NOT NULL,
  source    TEXT,                              -- inbox/phone, inbox/camera, ...
  noted_at  INTEGER NOT NULL
);

-- 4.2.3 Event
CREATE TABLE event (
  id            INTEGER PRIMARY KEY,
  kind          TEXT NOT NULL,                 -- badminton_session|badminton_match|cycling|trip|daily
  parent_id     INTEGER REFERENCES event(id),  -- session→match
  title         TEXT,
  start_at      INTEGER NOT NULL,
  end_at        INTEGER,
  place_id      INTEGER REFERENCES place(id),
  payload_json  TEXT,                          -- 领域专属字段
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL
);
CREATE INDEX idx_event_kind_time ON event(kind, start_at);
CREATE INDEX idx_event_parent ON event(parent_id);

-- 4.2.4 Asset ↔ Event N:N
CREATE TABLE asset_event (
  asset_id INTEGER NOT NULL REFERENCES asset(id),
  event_id INTEGER NOT NULL REFERENCES event(id),
  role     TEXT,                                -- main|context|companion
  PRIMARY KEY (asset_id, event_id)
);

-- 4.2.5 Segment（视频内时间片）
CREATE TABLE segment (
  id            INTEGER PRIMARY KEY,
  asset_id      INTEGER NOT NULL REFERENCES asset(id),
  event_id      INTEGER REFERENCES event(id),
  kind          TEXT NOT NULL,                 -- rally|game_break|climb|descent|...
  start_ts      REAL NOT NULL,                 -- 视频内秒
  end_ts        REAL NOT NULL,
  payload_json  TEXT,
  created_by    TEXT,                          -- ai|user|system
  confidence    REAL,                          -- AI 置信度（人工时为 1.0）
  created_at    INTEGER NOT NULL,
  -- 生成列：常查字段提取
  rally_idx       INTEGER GENERATED ALWAYS AS (json_extract(payload_json,'$.rally_idx')) STORED,
  game_idx        INTEGER GENERATED ALWAYS AS (json_extract(payload_json,'$.game_idx')) STORED,
  winner_side     TEXT    GENERATED ALWAYS AS (json_extract(payload_json,'$.winner_side')) STORED,
  shot_count      INTEGER GENERATED ALWAYS AS (json_extract(payload_json,'$.shot_count')) STORED,
  last_shot_type  TEXT    GENERATED ALWAYS AS (json_extract(payload_json,'$.last_shot_type')) STORED,
  rally_quality   INTEGER GENERATED ALWAYS AS (json_extract(payload_json,'$.rally_quality')) STORED
);
CREATE INDEX idx_seg_asset ON segment(asset_id, start_ts);
CREATE INDEX idx_seg_event ON segment(event_id, kind);
CREATE INDEX idx_seg_shot ON segment(last_shot_type);
CREATE INDEX idx_seg_winner ON segment(winner_side);

-- 4.2.6 Person（人物）
CREATE TABLE person (
  id          INTEGER PRIMARY KEY,
  name        TEXT NOT NULL,
  aliases     TEXT,                            -- JSON 数组
  role        TEXT,                            -- self|family|friend|teammate|opponent
  notes       TEXT,
  created_at  INTEGER NOT NULL
);

-- 人脸聚类簇
CREATE TABLE face (
  id          INTEGER PRIMARY KEY,
  asset_id    INTEGER NOT NULL REFERENCES asset(id),
  bbox        TEXT,                            -- JSON [x,y,w,h]
  embedding   BLOB,                            -- 512f
  cluster_id  INTEGER,                         -- 同人聚类
  person_id   INTEGER REFERENCES person(id),   -- 命名后回填
  confidence  REAL
);
CREATE INDEX idx_face_cluster ON face(cluster_id);
CREATE INDEX idx_face_person ON face(person_id);

CREATE TABLE asset_person (
  asset_id  INTEGER NOT NULL REFERENCES asset(id),
  person_id INTEGER NOT NULL REFERENCES person(id),
  source    TEXT,                              -- face_recog|user_tag
  PRIMARY KEY (asset_id, person_id)
);

-- 4.2.7 Place（双层）
CREATE TABLE place (
  id          INTEGER PRIMARY KEY,
  name        TEXT NOT NULL,
  layer       TEXT NOT NULL,                   -- known|geocoded
  aliases     TEXT,
  geo_polygon TEXT,                            -- known 用 GeoJSON
  center_lat  REAL, center_lon REAL,
  region_path TEXT,                            -- "广东省/深圳市/福田区"
  notes       TEXT
);
CREATE INDEX idx_place_layer ON place(layer);

-- 4.2.8 Tag（命名空间）
CREATE TABLE tag (
  id          INTEGER PRIMARY KEY,
  ns          TEXT NOT NULL,                   -- category|event|play|style|topic|quality
  key         TEXT NOT NULL,
  display     TEXT,
  description TEXT,
  is_managed  INTEGER NOT NULL,                -- 1=受控, 0=自由
  parent_id   INTEGER REFERENCES tag(id),
  UNIQUE(ns, key)
);

-- 多态标签关联（targets：asset / event / segment）
CREATE TABLE tag_link (
  tag_id      INTEGER NOT NULL REFERENCES tag(id),
  target_kind TEXT NOT NULL,                   -- asset|event|segment
  target_id   INTEGER NOT NULL,
  source      TEXT,                            -- user|ai|rule
  PRIMARY KEY (tag_id, target_kind, target_id)
);
CREATE INDEX idx_taglink_target ON tag_link(target_kind, target_id);

-- 4.2.9 Embedding 向量索引（sqlite-vec）
-- 由 sqlite-vec 扩展提供：vec_asset(rowid, embedding(512))
-- 业务侧只引用 rowid，不直接 DDL

-- 4.2.10 SmartAlbum（专题）
CREATE TABLE smart_album (
  id          INTEGER PRIMARY KEY,
  name        TEXT NOT NULL UNIQUE,
  rule_json   TEXT NOT NULL,                   -- 查询规则
  cover_id    INTEGER REFERENCES asset(id),
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);

-- 4.2.11 Job（异步任务）
CREATE TABLE job (
  id          INTEGER PRIMARY KEY,
  kind        TEXT NOT NULL,                   -- scan|transcode|embed|face|rally_detect|export
  payload     TEXT,
  state       TEXT NOT NULL,                   -- queued|running|done|failed|canceled
  attempts    INTEGER DEFAULT 0,
  error       TEXT,
  created_at  INTEGER NOT NULL,
  started_at  INTEGER,
  finished_at INTEGER
);
CREATE INDEX idx_job_state ON job(state, kind);

-- 4.2.12 FTS5 全文检索（驼峰命名/标题/笔记）
CREATE VIRTUAL TABLE fts_text USING fts5(
  target_kind, target_id UNINDEXED,
  text,
  tokenize='porter unicode61'
);
```

### 4.3 羽毛球四层模型（关键专项）

```
Session   一次约球（2-3h，一群人在一个场馆）
  │  Event(kind=badminton_session)
  │  payload: { court, ball_brand, attendees[] }
  │
  ├─ Match  一场对阵（一对一/一对二）
  │     Event(kind=badminton_match, parent_id=session.id)
  │     payload: {
  │       match_type: singles|doubles,
  │       format: "21x3"|"15x3"|"11x1",
  │       side_a: [person_id...], side_b: [person_id...],
  │       final_score: [[21,18],[19,21],[21,15]],
  │       winner: A|B,
  │       duration_minutes: 35
  │     }
  │     │
  │     ├─ Game  一局
  │     │     Segment(kind=game) on whichever asset
  │     │     payload: { game_idx, score_final:[a,b], duration }
  │     │
  │     └─ Rally  一回合
  │           Segment(kind=rally) on asset
  │           payload: {
  │             rally_idx, game_idx,
  │             server: person_id, serve_side: deuce|ad,
  │             score_before:[a,b], score_after:[a,b],
  │             winner_side: A|B, winner_player: person_id,
  │             losing_reason: out|net|winner|unforced_error,
  │             shot_count, last_shot_type,
  │             rally_quality: 1-5,
  │             tactical_pattern: drive_battle|front_back|four_corners|attack_block,
  │             error_type: technique|tactic|physical|mental,
  │             tags: [style/violent-smash, play/smash, ...],
  │             learning_note, opponent_pattern
  │           }
  │
  └─ Asset...  一段或多段视频，与 Match/Game 是 N:N
```

**关键不变量**：
- 视频边界 ≠ 比赛边界。一段 7-12 分钟视频可能横跨一局的中段，也可能涵盖两局过半。
- `Segment(kind=rally)` 上的 `start_ts / end_ts` 是**视频内秒数**；`rally_idx` 全场连续编号；`game_idx` 局内编号。
- `Match.payload.final_score` 是真值；任何回合 `score_after` 必须最终能与 final_score 对上（可作 lint）。

### 4.4 Tag 命名空间规范（强约束）

| ns | 性质 | 词表来源 | 示例 |
|---|---|---|---|
| `category/*` | 闭集 | 系统硬编码 | `category/badminton`, `category/cycling`, `category/daily`, `category/trip` |
| `event/*` | 闭集 | 插件维护 | `event/match`, `event/training`, `event/group-photo` |
| `play/*` | 受控可扩展 | 词表文件，新增需确认 | `play/smash`, `play/drop`, `play/clear`, `play/long-rally` |
| `style/*` | 受控可扩展 | 词表文件 | `style/violent-smash`(暴力扣杀), `style/old-school`(老登球路) |
| `topic/*` | 自由 | 用户输入 | `topic/2026-青海骑行`, `topic/儿子毕业` |
| `quality/*` | 闭集 | 系统硬编码 | `quality/highlight`, `quality/blur`, `quality/duplicate-near` |

新词进入 `play/*` `style/*` 受控词表前，UI 需弹出"是否确认新词，已有近义词：…"，迫使用户合并。

---

## 5. 处理流水线

### 5.1 入库管线（C2 命令式）

```
nasmgr import [--dry-run] [--source phone|camera|badminton|cycling]
```

```
inbox/* 文件
   │
   ▼
[1] 探测（probe）           ffprobe / exiftool 拿元数据
   │
   ▼
[2] 计算 sha256 + phash      去重判断
   │
   ▼
[3] 判定                     ├─ 新文件      → 进 raw/
                             ├─ 完全重复    → 登记 alias，不入 raw
                             └─ 同源低质    → 进 archive/，挂 canonical_id
   │
   ▼
[4] 物理移动                 raw/YYYY/MM/DD/<name>_<hash6>.<ext>
   │
   ▼
[5] 写入 asset 表            含 EXIF/GPS/duration/quality_score
   │
   ▼
[6] 投递衍生任务             transcode_proxy, gen_thumb, gen_keyframes,
                             clip_embed, face_detect, rally_detect_candidate
```

### 5.2 衍生处理（worker 异步）

| Job kind | 触发 | 输入 | 输出 |
|---|---|---|---|
| `transcode_proxy` | 视频入库 | raw 视频 | derived/proxies/.../*.mp4（720p H.264） |
| `gen_thumb` | 入库 | raw 文件 | derived/thumbnails/{256,512,1024}/ |
| `gen_keyframes` | 视频入库 | proxy | derived/keyframes/.../sec_NNN.jpg |
| `clip_embed` | 入库 | 缩略图 / 关键帧 | sqlite-vec 向量 |
| `face_detect` | 照片入库 / 视频关键帧 | InsightFace | face 行 + 聚类 |
| `face_cluster` | 周期 | face.embedding | 更新 cluster_id |
| `rally_detect_candidate` | 羽毛球 Asset | proxy 音频 + 运动量 | 候选 segment(kind=rally, created_by=ai) |
| `event_mining` | 周期 | Asset 时空数据 | 产出/更新 Event |

### 5.3 事件聚合规则（Event Mining）

| Event 类型 | 触发条件 | 默认参数 |
|---|---|---|
| `badminton_session` | GPS 落入 known_place(layer=known, tag=羽毛球场) ∧ 持续 > 30min | 间隔 < 30min 合并 |
| `cycling` | 有同名 GPX 同行 OR 轨迹 > 5km ∧ 平均速度 12-40km/h | 间隔 < 60min 合并 |
| `trip` | 距家 > 50km ∧ 持续 > 12h | 跨日延伸；归家时收尾 |
| `daily` | 不强制建 Event；按月做虚拟分组 | 仅作浏览容器 |

事件一旦创建，**用户拥有最终修改权**：可以重命名、合并、拆分、删除。系统不再覆盖用户的修改。

### 5.4 羽毛球回合切分启发式（E1 仅做候选）

```
输入：proxy 视频
信号源：
  - 音频：球拍击球瞬时高频脉冲 + 鞋摩擦地板的间歇
  - 运动量：帧差或简易光流幅度（OpenCV）
  - 静默检测：silencedetect 找到长静默段（可能是回合间）
启发式：
  1. 静默 ≥ 3s 视为回合分隔
  2. 击球次数 < 2 的"片段"丢弃（漏切）
  3. 段长 < 2s 或 > 60s 标记为 low-confidence
输出：segment(kind=rally, created_by=ai, confidence=0.x, payload={start,end})
```

人工标注覆盖 AI 候选时，`created_by` 改为 `user`，`confidence=1.0`。

### 5.5 标注 UI（H1 键盘流）

| 键 | 动作 |
|---|---|
| `space` | 暂停/播放 |
| `←/→` | -1s / +1s |
| `shift + ←/→` | -5s / +5s |
| `,` / `.` | 上一回合 / 下一回合 |
| `[` / `]` | 设置当前回合的 start / end |
| `1` / `2` | 标得分方为 A / B |
| `s` | 弹出发球人选择（数字键选） |
| `q/w/e/r/t/y/u` | 快速打 last_shot_type（smash/drop/clear/drive/push/net/lift） |
| `g` | rally_quality + 1 |
| `b` | rally_quality - 1 |
| `h` | toggle highlight |
| `n` | 加 note |
| `ctrl+s` | 保存 |
| `?` | 显示快捷键帮助 |

**标注期望速度**：10 分钟视频 5-8 分钟标完。

### 5.6 集锦导出

```
导出规则（YAML）：
  filter:
    segment.kind: rally
    segment.tags: [style/violent-smash]
    asset.captured_at: { gte: 2026-01-01 }
  order_by: rally_quality desc
  limit: 50
  intro_seconds: 0.5         # 每段开头补 0.5s
  outro_seconds: 0.5
  transition: fade            # fade|cut
  output: derived/exports/violent-smash-2026.mp4

执行：
  1. 解析规则 → 拿到 segment 列表
  2. 计算每段的 [start-intro, end+outro]
  3. ffmpeg 抽段 + concat
  4. 落到 derived/exports/
```

---

## 6. 检索接口（八维查询）

API 设计成"统一查询端点 + 组合过滤器"：

```
POST /api/search
{
  "target": "asset | event | segment",
  "filters": {
    "captured_at":  { "gte": ..., "lt": ... },
    "person_ids":   [12, 34],
    "place_ids":    [5],
    "categories":   ["badminton"],
    "tags":         ["style/violent-smash", "play/smash"],
    "text":         "暴力扣杀",          // FTS5 + CLIP 混合
    "where_sql":    "shot_count > 15"   // 受控范围 SQL where 片段
  },
  "order_by": [["rally_quality","desc"]],
  "limit":  100,
  "offset": 0
}
```

**混合召回**：text 字段先走 FTS5；命中不足时调 CLIP 文本嵌入，做向量召回；最终融合排序。

**聚合统计**：另开 `/api/stats` 端点，支持 `group_by` 在 person/place/month/category/shot_type/winner_side 上聚合。

---

## 7. 与 Immich 的边界（D2 共生约定）

| 事项 | NAS-MGR | Immich |
|---|---|---|
| `raw/` 目录 | 写入方（入库管线） | 只读消费方（External Library 模式） |
| 缩略图 | 自己生成在 derived/thumbnails/ | 自己生成（独立缓存） |
| 人脸 | 自研（InsightFace + 自己的 person 表） | Immich 内置 |
| 浏览/手机同步 | 不做 | **由 Immich 负责** |
| 羽毛球/骑行专项 | **由 NAS-MGR 负责** | 不做 |
| 元数据 | catalog.db | Immich Postgres（独立） |

> 用户心智："手机同步、看日常照片 → Immich App；想复盘羽毛球、剪集锦、看专题 → NAS-MGR Web"。

---

## 8. 非功能需求

| 维度 | 要求 |
|---|---|
| 可重建性 | `derived/` `index/` 任意时刻清空都能从 `raw/` 重建（标注从 JSONL 恢复） |
| 备份 | `raw/` 异地，`catalog.db` 每日快照，标注每日 JSONL 导出 |
| 隐私 | `Asset.privacy_level=high` 时跳过 AI、跳过缩略图（或仅本地缩略图） |
| 性能 | 入库 ≥ 500 文件/分钟（不含转码）；查询 P95 < 200ms（10 万级数据集） |
| 容量预估 | 元数据 < 5GB；CLIP 向量 ≤ 10GB；代理片 ≤ 原视频 1/5；缩略图 ≤ 100GB |
| 冷热分层 | 拍摄日期 > 3 年只索 Asset 元数据 + 缩略图，AI 按需触发 |
| 安全 | API 仅 127.0.0.1 监听；后期家庭共享走反向代理 + 单用户 token |
| 鲁棒性 | 所有 worker 任务可重入；NAS 短暂离线时排队，恢复后自动续跑 |

---

## 9. 实施分期

| 阶段 | 周期 | 目标 | 完工标志 |
|---|---|---|---|
| **P0 · 基础入库** | 1-2 周 | nasmgr import + 缩略图 + Web 时间轴 | 全 5TB 扫完，时间轴可滚 |
| **P1 · 人物/地点/标签** | 1-2 周 | 人脸聚类、Place 词表、Tag 命名空间 | 能按"人 × 地点 × 月份"筛选 |
| **P2 · 羽毛球专项** | 2-3 周 | Match/Rally 模型、回合候选、键盘流标注 | 一场比赛 5-8 分钟标完 |
| **P3 · 语义检索** | 1-2 周 | CLIP 文本召回 + FTS5 + 混合排序 | "夕阳骑车"能召回 |
| **P4 · 集锦与骑行** | 1-2 周 | YAML 规则导出 + GPX 视频对齐 | 一键出"暴力扣杀合集.mp4" |
| **P5 · Immich 接入** | 1 周 | Immich 容器 + External Library 指向 raw/ | 手机同步通畅 |
| **P6 · 共享与远程**（可选） | 1-2 周 | 反向代理 + 多用户 token | 家人/球友远程访问 |

---

## 10. 仓库目录草案（实施时落地）

```
NAS-MGR/
├── docs/
│   ├── SYSTEM_DESIGN.md          ← 本文
│   ├── DATA_MODEL.md             ← 表结构详解
│   ├── BADMINTON_SPEC.md         ← 羽毛球四层模型 + 标注规范
│   ├── PLACE_DICT.md             ← known_places 词表
│   └── TAG_DICT.md               ← 受控词表
├── nasmgr/
│   ├── api/                      ← FastAPI
│   ├── worker/                   ← 异步任务
│   ├── core/                     ← 数据模型/规则引擎
│   ├── importers/                ← 入库管线
│   ├── ai/                       ← CLIP / face / rally_detect
│   └── cli/                      ← nasmgr 命令行
├── web/                          ← Vue/Svelte SPA
├── deploy/
│   ├── docker-compose.immich.yml
│   └── autofs.conf.sample
└── tests/
```

---

## 11. 待办（Open Questions）

定稿前未解决，进入 P0 前需要确认：

1. **NAS 型号/系统**：群晖/威联通/TrueNAS/自建？影响 SMB 性能调优和后续是否在 NAS 跑容器。
2. **共享需求时间表**：是否在 6 个月内引入家人/球友访问？影响是否要早早把权限模型留好钩子。
3. **羽毛球录制硬件**：手机三脚架定机位？双机位？是否打算后续上稳定球场摄像头？影响 rally_detect 启发式的参数。
4. **常去地点清单**：先列 5-10 个 known_places（家、公司、橙天羽毛球馆、美沙集合点……）作为 P1 词表种子。
5. **GPX 来源**：码表（佳明/迈金）还是手机 App（咕咚/Strava）？影响导入器写哪种格式优先。

---

## 12. 文档维护约定

- 本文件是**整个项目的源头真理**，任何与代码/讨论的冲突以本文件为准。
- 每次设计变更必须**先改本文档**，再进入实施。
- 决策记录（Decision Record）只增不删；如某项决策被推翻，加"v2 修订"段落，保留 v1。
- 实施过程中发现设计漏洞，记录到第 11 节"待办"，下次定稿合并。

---

> **当前状态**：v1.0 方案定型，可启动 P0。
> 下一步建议：用户回答第 11 节的 5 个问题后，开 P0 实施票。
