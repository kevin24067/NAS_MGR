# NAS-MGR

私人媒体管理系统 — 把 NAS 上 5TB 照片视频（羽毛球 / 骑行 / 日常 / 旅行）变成可按时间·地点·人物·类别·专题检索、可对羽毛球做结构化复盘的私域工具。

## 当前状态

- v1.0 系统方案定稿，详见 [docs/SYSTEM_DESIGN.md](docs/SYSTEM_DESIGN.md)
- 实施进度：尚未启动 P0

## 方案速览（v1.2）

- **定位**：**只读外挂索引层** — 不接管 NAS 入库，不动原文件位置和组织
- **架构**：NAS 只存储 + Mac 跑服务 + Web SPA + Immich 共生
- **catalog.db 在 Mac 本地**（v1.2 由 NAS 迁回）：~1.8GB 体量，本地查询快 30-50 倍，NAS 离线时仍可浏览/标注
- **NAS 上仅 `.nasmgr/labels.jsonl`**：标注真值（append-only），catalog.db 可从此 + 扫描重建
- **衍生物**：缩略图/代理片/向量全部 Mac 本地缓存（约 400GB）
- **核心模型**：Asset / Event / Segment / Person / Place / Tag / SmartAlbum
- **羽毛球四层**：Session → Match → Game → Rally，结构化标注复盘
- **检索八维**：时间 / 人物 / 地点 / 类别 / 标签 / 语义 / 聚合 / 导出
- **写动作**：仅"按需导出"（用户主动触发）+ 标注追加 labels.jsonl

## 文档

| 文档 | 内容 |
|---|---|
| [SYSTEM_DESIGN.md](docs/SYSTEM_DESIGN.md) | 总体方案（决策记录、架构、数据模型、流水线、分期）|
| [BADMINTON_SPEC.md](docs/BADMINTON_SPEC.md) | 羽毛球专项数据规范（labels.jsonl 行格式、枚举、约束）|
| [TAG_DICT.md](docs/TAG_DICT.md) | 受控标签词表（命名空间、可扩展规则）|
| [templates/](templates/) | 标注模板（CSV / jsonl + 使用说明）|

后续会陆续补充 DATA_MODEL.md / PLACE_DICT.md。

## 实施分期

| 阶段 | 周期 | 目标 |
|---|---|---|
| P0 | 1-2 周 | 扫描与索引 + 时间轴 |
| P1 | 1-2 周 | 人物/地点/标签 |
| P2 | 2-3 周 | 羽毛球专项 |
| P3 | 1-2 周 | 语义检索 |
| P4 | 1-2 周 | 按需导出（集锦+相册） |
| P5 | 1 周   | Immich 共存 |
| P6 | 1-2 周 | 共享与远程（可选）|
