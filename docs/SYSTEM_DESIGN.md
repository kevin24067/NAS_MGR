# NAS-MGR 系统设计文档

> 版本：v1.1 · 修订日期：2026-05-22
> 状态：方案定型，待进入实施分期 P0
> 维护者：KH
>
> **v1.1 重大修订**：本系统**仅作为外挂索引层**，不接管 NAS 上的素材入库与组织。
> 原文件位置、目录结构、命名方式由用户/其他工具（手机同步、Immich 等）维护，
> NAS-MGR 只负责"扫描发现 → 索引 → 标注/检索 → 按需导出"。

---

## 0. 一句话目标

把 NAS 上**已存在**的 5TB 照片视频（羽毛球 / 骑行 / 日常 / 旅行）以**只读外挂索引**的方式接入，做到可按时间·地点·人物·类别·专题检索、可对羽毛球视频做结构化复盘、可按需导出主题集锦——**而完全不改变 NAS 上的任何原文件位置和组织方式**。

---

## 1. 决策定型（Decision Record）

| 决策项 | 选择 | 含义 |
|---|---|---|
| **A. 部署形态** | A2 | NAS 只做存储，所有服务（API、Worker、AI）跑在 Mac 上 |
| **B. 元数据库位置** | B1 | SQLite 主库 `catalog.db` 存放在 NAS 的 `.nasmgr/` 隐藏目录 |
| **C. 索引触发** | C2 | 显式 `nasmgr scan` 命令触发；后续可演进到 watch |
| **D. 照片栈选型** | D2 | Immich 接管日常照片/视频浏览与人脸；NAS-MGR 自研羽毛球/骑行/专题层 |
| **E. 羽毛球 AI 程度** | E1 | AI 只生成"回合切分候选"，比分/发球人/得分方/技术标签全部走人工标注 |
| **F. 地图服务** | F1 | 高德地图 API（在线，需 GCJ02 纠偏） |
| **G. 前端形态** | G1 | 单 Web SPA |
| **H. 标注 UI** | H1 | Web 端键盘流（空格/数字键/ASD 等快捷键） |

> **v1.1 关键变更**：决策 C 由"入库触发"改为"索引触发"。NAS-MGR 不再有"入库管线"概念，只做被动扫描。原 `inbox/` 暂存区方案废弃。

---

## 2. 核心架构原则（v1.1 重写）

### 2.1 NAS-MGR 是外挂索引，不是文件管理器

| ✅ NAS-MGR 做 | ❌ NAS-MGR 不做 |
|---|---|
| 扫描 NAS 上指定路径，发现媒体文件 | 移动、重命名、删除任何文件 |
| 读取 EXIF/元数据，建立 SQLite 索引 | 在 NAS 上规定目录结构 |
| 在本地缓存生成缩略图、向量、人脸 | 在 NAS 上写衍生物（默认） |
| 提供检索 + 标注 + 按需导出 | 接管手机同步、自动归档 |
| 标注数据持久化 | 改写原始文件的 metadata（除非明确导出） |

**核心不变量**：在 NAS 上的原文件**只读**。索引可以全部清空重建，原文件永远不变。

### 2.2 拓扑

```
┌──────────────────────────────────────────────────────────────────────┐
│                          NAS（用户已有的存储）                          │
│  现有目录结构 ← 由用户/手机同步/Immich 等工具维护                        │
│  例：/media/photos/2024/...                                            │
│       /media/badminton/2026-05-22-橙天/                               │
│       /media/cycling/2026-05-20-meishan/                              │
│  NAS-MGR 只读消费这些路径，不改任何文件                                 │
│                                                                       │
│  /.nasmgr/                ← NAS-MGR 唯一在 NAS 上写入的目录            │
│      catalog.db           ← SQLite 主库（可重建）                      │
│      catalog.db-wal                                                    │
│      labels.jsonl         ← 每日标注备份（人工产物，不可丢）           │
└────────────┬─────────────────────────────────────────────────────────┘
             │ SMB 挂载（autofs）→ /Volumes/nas-media（只读挂载）
             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                            Mac（计算 + UI）                            │
│  nasmgr-api      FastAPI · REST/JSON                                  │
│  nasmgr-worker   AI/抽帧/embed 的本地 worker                           │
│  nasmgr-web      Web SPA · 浏览 / 标注 / 导出                          │
│  ~/Library/.../nasmgr-cache/    ← 衍生物本地缓存（缩略图/代理片/向量）  │
│  immich          Docker · 接管日常浏览（D2 共生）                      │
│  高德 API        地理编码                                              │
└────────────┬─────────────────────────────────────────────────────────┘
             │ 用户主动触发导出
             ▼
        /Volumes/nas-media/Exports/  ← 仅在导出时写入 NAS
        或本地 ~/Movies/Exports/
```

### 2.3 衍生物存放策略

| 衍生物 | 默认位置 | 可选位置 | 说明 |
|---|---|---|---|
| 缩略图（256/512/1024） | Mac 本地 `~/Library/Caches/nasmgr/thumbs/` | NAS `.nasmgr/cache/` | 本地默认，加载快；多设备共享时可选 NAS |
| 视频代理片（720p） | Mac 本地 `~/Library/Caches/nasmgr/proxies/` | NAS | 同上 |
| 关键帧 | Mac 本地 | NAS | 同上 |
| CLIP 向量 / 人脸 | Mac 本地 sqlite-vec 文件 | NAS | 默认本地，重建廉价 |
| 标注数据 | **NAS `.nasmgr/catalog.db` 主存** + 每日 JSONL 导出 | — | 人工产物，必须落 NAS |
| 集锦/导出成片 | 用户指定目标目录 | — | 写到哪由用户在导出时决定 |

> 默认所有可重建产物在 Mac 本地，NAS 上只有 `.nasmgr/catalog.db` 和 `.nasmgr/labels.jsonl` 两个文件——一个保存索引和标注（可由 JSONL 重建），一个是标注的纯文本备份（人工劳动结晶，不可丢）。

---

## 3. 索引发现机制（替代原"入库管线"）

### 3.1 配置驱动的扫描根

`nasmgr.config.yaml`（项目根，git 管理）：

```yaml
scan_roots:
  - path: /Volumes/nas-media/photos
    kind: photo                  # 默认按 EXIF 分类
    recursive: true

  - path: /Volumes/nas-media/badminton
    kind: badminton              # 暗示走羽毛球处理插件
    recursive: true
    metadata_overrides:
      default_category: category/badminton

  - path: /Volumes/nas-media/cycling
    kind: cycling
    recursive: true

  - path: /Volumes/nas-media/daily
    kind: daily

ignore_patterns:
  - "**/.*"                      # 隐藏文件
  - "**/@eaDir/**"               # 群晖缩略图缓存
  - "**/Thumbs.db"
  - "**/.DS_Store"

extensions:
  photo: [jpg, jpeg, heic, heif, png, raw, dng, arw, cr2, nef]
  video: [mp4, mov, m4v, mkv, avi, mts]
```

**关键点**：用户可以**任意调整 NAS 上的目录结构**，只要在配置里更新扫描根即可。系统不绑定任何特定布局。

### 3.2 扫描命令

```bash
nasmgr scan                       # 全量增量扫描所有 scan_roots
nasmgr scan --root /path          # 仅扫指定根
nasmgr scan --since 2026-05-01    # 只看修改时间晚于该日期
nasmgr scan --dry-run             # 只报告变更，不写库
```

### 3.3 扫描流程

```
[1] 遍历 scan_roots 中所有路径（受 ignore_patterns 过滤）
   │
   ▼
[2] 对每个文件：
    ├─ 已在 asset 表（按 path）：
    │   ├─ mtime + size 一致 → 跳过
    │   └─ 不一致 → 重新读 EXIF / 重算 hash → 更新行
    │
    └─ 不在 asset 表：
        ├─ 读 EXIF/ffprobe 获取元数据
        ├─ 算 sha256（增量大文件用流式）
        ├─ 检查同 sha256 是否已存在（重定位检测）
        │   └─ 如存在 → 仅在 asset_path_history 加一条新路径
        ├─ 否则插入新 asset 行
        └─ 入队衍生任务（thumb / proxy / clip_embed / face / rally_detect 候选）
   │
   ▼
[3] 检测被删除的 asset：
    asset 表中存在但磁盘上找不到的文件 → 标 status=missing（不删）
```

### 3.4 文件位置变更的处理

由于不接管文件管理，**用户随时可能在 NAS 上移动/重命名文件**（手动整理、Immich 重组、新装其他工具）。设计要点：

- **sha256 是身份**，path 只是定位
- 扫描发现一个 sha256 已知但 path 变更的文件 → 自动更新 path，不重新生成衍生物
- 一个 asset 历史 path 列表存在 `asset_path_history`，便于追溯
- `status = active | missing | suspicious`：missing 不主动删，等下一轮扫描确认或用户清理

---

## 4. 数据模型（v1.1 调整）

### 4.1 Asset 表字段调整

```sql
CREATE TABLE asset (
  id              INTEGER PRIMARY KEY,
  path            TEXT NOT NULL UNIQUE,    -- NAS 上的绝对路径或相对挂载点路径
  scan_root_id    INTEGER NOT NULL,        -- 来自哪个 scan_root（决定默认 kind）
  sha256          TEXT NOT NULL,
  phash           TEXT,
  media_type      TEXT NOT NULL,
  mime            TEXT,
  size_bytes      INTEGER,
  mtime           INTEGER,                  -- 文件修改时间（增量扫描凭据）
  width           INTEGER, height INTEGER,
  duration        REAL,
  captured_at     INTEGER,                  -- EXIF 拍摄时间
  first_seen_at   INTEGER NOT NULL,         -- 第一次被本系统发现的时间
  last_scan_at    INTEGER NOT NULL,         -- 最近一次确认存在的时间
  gps_lat         REAL, gps_lon REAL,
  gps_lat_gcj     REAL, gps_lon_gcj REAL,
  camera_make     TEXT, camera_model TEXT, lens TEXT,
  quality_score   REAL,
  privacy_level   INTEGER DEFAULT 0,
  status          TEXT DEFAULT 'active'     -- active | missing | suspicious
);
CREATE UNIQUE INDEX idx_asset_sha ON asset(sha256);
CREATE INDEX idx_asset_captured ON asset(captured_at);
CREATE INDEX idx_asset_root ON asset(scan_root_id);
CREATE INDEX idx_asset_status ON asset(status);

-- 历史路径（处理用户移动/重命名）
CREATE TABLE asset_path_history (
  asset_id   INTEGER NOT NULL REFERENCES asset(id),
  path       TEXT NOT NULL,
  noted_at   INTEGER NOT NULL,
  reason     TEXT                            -- 'first_seen' | 'moved' | 'restored'
);

-- 扫描根（配置在 yaml，运行时同步到表）
CREATE TABLE scan_root (
  id           INTEGER PRIMARY KEY,
  path         TEXT NOT NULL UNIQUE,
  kind_hint    TEXT,                          -- photo|badminton|cycling|...
  config_json  TEXT
);
```

### 4.2 删除的字段/概念

相对 v1.0 的清理：

- ❌ `asset_path_alias`（不再处理"入库重复"，因为不入库）
- ❌ `canonical_id`（不再做"同源副本降级"，原文件用户自己管理）
- ❌ `archive/` 目录概念
- ❌ `inbox/` 目录概念
- ❌ "import" job kind

### 4.3 其余实体不变

Event / Segment / Person / Place / Tag / SmartAlbum / Job / Embedding / FTS5 都按 v1.0 4.2 节定义保留。羽毛球四层模型（Session/Match/Game/Rally）不变。

---

## 5. 处理流水线（v1.1 重写）

```
┌─────────────────────────────────────────────────────────────┐
│  nasmgr scan         ← 用户主动触发（或定时）                 │
│  发现 / 更新 / 检测移动 / 标 missing                          │
└────────────┬────────────────────────────────────────────────┘
             │ 投递衍生任务（每个新/变更的 asset）
             ▼
┌─────────────────────────────────────────────────────────────┐
│  nasmgr-worker      Mac 本地后台 worker                      │
│  ├─ gen_thumb            缩略图 → 本地缓存                    │
│  ├─ transcode_proxy      720p 代理片 → 本地缓存               │
│  ├─ gen_keyframes        关键帧 → 本地缓存                    │
│  ├─ clip_embed           CLIP 向量 → sqlite-vec              │
│  ├─ face_detect          人脸 → face 表                      │
│  ├─ face_cluster         周期：聚类                           │
│  ├─ rally_detect_candidate  羽毛球：切回合候选                │
│  └─ event_mining         周期：聚合 Session/Match/Trip       │
└────────────┬────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│  Web UI                                                       │
│  ├─ 浏览 / 检索（八维查询）                                    │
│  ├─ 羽毛球标注（键盘流）                                       │
│  ├─ 专题（SmartAlbum）                                        │
│  └─ 按需导出 ←— 唯一一个会写文件的动作                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 按需导出（v1.1 强调）

这是 NAS-MGR 唯一会**主动产生新文件**的动作，且只在用户明确触发时执行。

### 6.1 导出类型

| 类型 | 输入 | 输出 | 写到哪 |
|---|---|---|---|
| 集锦视频 | Segment 列表 + 规则 | 拼接的 MP4 | 用户指定目录（默认 NAS `Exports/`） |
| 相册 zip | Asset 列表 | zip 包，含原图或缩放图 | 同上 |
| 元数据导出 | 查询结果 | JSON / CSV | 同上 |
| 标注备份 | 全量 segment + tag | JSONL | 默认 NAS `.nasmgr/labels.jsonl`（每日自动） |
| 边车元数据（可选） | 原文件 + asset 行 | XMP/JSON sidecar 写到原文件旁 | 需要用户开关明确同意 |

### 6.2 导出 YAML 规则示例

```yaml
type: highlight_video
filter:
  segment.kind: rally
  segment.tags: [style/violent-smash]
  asset.captured_at: { gte: 2026-01-01 }
order_by: rally_quality desc
limit: 50
intro_seconds: 0.5
outro_seconds: 0.5
transition: fade
output: /Volumes/nas-media/Exports/violent-smash-2026.mp4
```

### 6.3 边车文件（可选高级特性）

用户开启 `enable_sidecar: true` 后，标注变更可同步写一个 `<原文件名>.nasmgr.json` 到原文件旁，作为冗余备份。**这是写操作，需要 NAS 挂载点为可写，且默认关闭。**

---

## 7. 与 Immich 的边界（v1.1 微调）

| 事项 | NAS-MGR | Immich |
|---|---|---|
| NAS 上原始素材 | 只读扫描发现 | External Library 模式只读 |
| 缩略图 | Mac 本地缓存 | 自己缓存 |
| 人脸 | 自研索引 | 自身识别 |
| 浏览/手机同步 | 不做 | **专责** |
| 羽毛球/骑行/专题 | **专责** | 不做 |

两者**互不干扰**，对 NAS 都是只读消费方。即使两者都装也不会"打架"，因为都不动原文件。

---

## 8. 非功能需求（v1.1 调整）

| 维度 | 要求 |
|---|---|
| **可重建性** | catalog.db 删除可从扫描 + labels.jsonl 重建（标注不丢，索引重跑） |
| **零侵入** | 删除 NAS-MGR 后 NAS 上只剩 `.nasmgr/` 一个目录可手动删，原文件无任何变化 |
| 备份 | 用户自管 raw 备份；`.nasmgr/labels.jsonl` 也建议进备份链 |
| 隐私 | privacy_level=high → 跳过 AI 处理 |
| 性能 | 扫描 ≥ 5000 文件/分钟（仅元数据，不含 hash 大文件）；查询 P95 < 200ms |
| 扫描鲁棒性 | 部分路径不可用时跳过 + 报告，不中断整次扫描 |
| 文件移动追踪 | 用户重命名/移动后下次扫描自动识别（sha256 匹配） |

---

## 9. 实施分期（v1.1 调整）

| 阶段 | 周期 | 目标 | 完工标志 |
|---|---|---|---|
| **P0 · 扫描与索引** | 1-2 周 | nasmgr scan + 缩略图 + Web 时间轴 | 全 5TB 扫完，时间轴可滚 |
| **P1 · 人物/地点/标签** | 1-2 周 | 人脸聚类、Place 词表、Tag 命名空间 | 三维筛选 |
| **P2 · 羽毛球专项** | 2-3 周 | Match/Rally + 回合候选 + 键盘流标注 | 一场比赛 5-8 分钟标完 |
| **P3 · 语义检索** | 1-2 周 | CLIP + FTS5 混合排序 | "夕阳骑车"能召回 |
| **P4 · 按需导出** | 1-2 周 | 集锦 YAML 规则 + GPX 视频对齐 | 一键出"暴力扣杀合集.mp4" |
| **P5 · Immich 共存** | 1 周 | Immich 容器同 raw 路径，互不干扰 | 手机同步通畅 |
| **P6 · 共享与远程**（可选） | 1-2 周 | 反向代理 + 多用户 token | 家人/球友远程访问 |

---

## 10. 仓库目录草案

```
NAS-MGR/
├── docs/
│   ├── SYSTEM_DESIGN.md          ← 本文
│   ├── DATA_MODEL.md
│   ├── BADMINTON_SPEC.md
│   ├── PLACE_DICT.md
│   └── TAG_DICT.md
├── nasmgr.config.yaml            ← 扫描根配置（用户填）
├── nasmgr/
│   ├── api/                      ← FastAPI
│   ├── worker/                   ← 异步任务
│   ├── core/                     ← 数据模型/规则引擎
│   ├── scanner/                  ← 扫描发现器（替代 importers/）
│   ├── ai/                       ← CLIP / face / rally_detect
│   ├── exporter/                 ← 集锦/相册/元数据导出
│   └── cli/                      ← nasmgr 命令行
├── web/                          ← Web SPA
├── deploy/
│   ├── docker-compose.immich.yml
│   └── autofs.conf.sample
└── tests/
```

---

## 11. 待办（Open Questions）

1. **NAS 型号/系统**：影响 SMB 性能调优。
2. **共享需求时间表**：6 个月内是否引入家人/球友访问？
3. **羽毛球录制硬件**：单机位/双机位/未来上球场摄像头？
4. **常去地点清单**：列 5-10 个 known_places 作为 P1 词表种子。
5. **GPX 来源**：码表（佳明/迈金）还是手机 App（咕咚/Strava）？
6. **NAS 现有目录结构**：让我看一眼顶层目录树（`tree -L 2 /Volumes/nas-media`），用来配 `scan_roots` 初始模板。

---

## 12. 修订历史

### v1.1 · 2026-05-22
- **重大架构修订**：废弃"入库管线"概念，改为只读外挂索引。
- 删除 inbox/raw/derived/archive 目录强约束；NAS 现有结构保持不变。
- 衍生物默认存 Mac 本地缓存，NAS 上仅 `.nasmgr/catalog.db` + `labels.jsonl`。
- 新增 `nasmgr.config.yaml` 配置驱动扫描根。
- 新增"文件移动追踪"机制（sha256 身份 + asset_path_history）。
- 调整 Asset 表字段：删除 canonical_id / asset_path_alias，新增 scan_root_id / mtime / status / first_seen_at / last_scan_at。
- 实施分期 P0 由"基础入库"改为"扫描与索引"。
- 强调"按需导出"是唯一会写文件的动作。

### v1.0 · 2026-05-22
- 初稿定型：8 项决策（A2/B1/C2/D2/E1/F1/G1/H1）。
- 通用核心数据模型 + 羽毛球四层 + 处理流水线 + 八维检索。

---

## 13. 文档维护约定

- 本文件是**整个项目的源头真理**，任何与代码/讨论的冲突以本文件为准。
- 每次设计变更必须**先改本文档**，再进入实施。
- 决策记录只增不删；推翻的决策作为修订段落保留。
- 实施过程中发现设计漏洞，记录到第 11 节，下次定稿合并。
