# NAS-MGR

私人媒体管理系统 — 把 NAS 上 5TB 照片视频（羽毛球 / 骑行 / 日常 / 旅行）变成可按时间·地点·人物·类别·专题检索、可对羽毛球做结构化复盘的私域工具。

## 当前状态

- v1.0 系统方案定稿，详见 [docs/SYSTEM_DESIGN.md](docs/SYSTEM_DESIGN.md)
- 实施进度：尚未启动 P0

## 方案速览

- **架构**：NAS 只存储 + Mac 跑服务 + Web SPA + Immich 共生
- **存储**：raw/ 只读 + derived/ 可重建 + index/ 元数据库 (SQLite + sqlite-vec + FTS5)
- **核心模型**：Asset / Event / Segment / Person / Place / Tag / SmartAlbum
- **羽毛球四层**：Session → Match → Game → Rally，结构化标注复盘
- **检索八维**：时间 / 人物 / 地点 / 类别 / 标签 / 语义 / 聚合 / 导出

## 文档

| 文档 | 内容 |
|---|---|
| [SYSTEM_DESIGN.md](docs/SYSTEM_DESIGN.md) | 方案定稿（决策记录、架构、数据模型、流水线、分期） |

后续会陆续补充 DATA_MODEL.md / BADMINTON_SPEC.md / PLACE_DICT.md / TAG_DICT.md。

## 实施分期

| 阶段 | 周期 | 目标 |
|---|---|---|
| P0 | 1-2 周 | 基础入库 + 时间轴 |
| P1 | 1-2 周 | 人物/地点/标签 |
| P2 | 2-3 周 | 羽毛球专项 |
| P3 | 1-2 周 | 语义检索 |
| P4 | 1-2 周 | 集锦与骑行 |
| P5 | 1 周   | Immich 接入 |
| P6 | 1-2 周 | 共享与远程（可选）|
