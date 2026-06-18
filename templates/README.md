# 标注模板说明

## 文件清单

| 文件 | 用途 |
|---|---|
| `badminton-rally-template.csv` | 在 Excel/Numbers 里逐行手填，每行一个 Rally |
| `badminton-rally-template.jsonl` | 直接写 jsonl 一行一个 Rally |

## 用 CSV 模板的工作流（推荐起步）

1. 打开 `badminton-rally-template.csv`（Excel/Numbers/Google Sheets 都行）
2. **第一行表头不要动**，从第二行开始填（前两行已有示例可参考后删掉）
3. **多人字段**用分号分隔：`KH;老王`（CSV 里逗号会冲突）
4. **多个 tags** 也用分号：`style/violent-smash;play/long-rally`
5. 一场打完导出成 CSV，丢给系统转成 jsonl 入库

## 关键填写规则

| 字段 | 必填 | 提示 |
|---|---|---|
| `asset_path` | ✓ | 视频在 NAS 上的相对路径 |
| `asset_sha256` | 可空着 | 系统扫描时自动补 |
| `start_ts / end_ts` | ✓ | 视频内秒数，1 位小数 |
| `session_id` | ✓ | `yyyy-mm-dd-地点缩写`，同一次约球同一个 ID |
| `match_id` | ✓ | `yyyy-mm-dd-mNN`，同一场对阵同一个 ID |
| `game_idx` | ✓ | 第几局（1/2/3）|
| `rally_idx` | ✓ | 全场连续编号 |
| `score_before_a/b` `score_after_a/b` | ✓ | 拆成 4 列填，避免 CSV 数组麻烦 |
| `winner_player` | 双打才填 | 单打留空 |
| `last_shot_type` | ✓ | smash/drop/clear/drive/push/net_shot/lift/block/serve |
| `losing_reason` | ✓ | winner/out/net/unforced_error |
| `tactical_pattern` | 选填 | drive_battle/front_back/four_corners/attack_block 等 |
| `error_type` | 失误时填 | technique/tactic/physical/mental |
| `tags` | 选填 | 见 docs/TAG_DICT.md，分号分隔 |
| `highlight` | 选填 | TRUE/FALSE |

## Lint 自检（提交前自己看一眼）

- 同一 game 内 rally_idx 连续递增
- 每个 rally 的 score_after - score_before 恰好 +1
- end_ts > start_ts
- 一个 game 的最后一个 rally，score_after 必须等于该 game 的最终比分
- highlight=TRUE 的尽量 rally_quality ≥ 4

## 转 jsonl

CSV 攒到一定量后，用 Python 一行就能转：

```python
import csv, json
rows = list(csv.DictReader(open('badminton-rally-template.csv')))
with open('labels.jsonl', 'w') as f:
    for r in rows:
        if not r['id']: continue
        # side_a/side_b/tags 拆分号
        for k in ['side_a','side_b','tags']:
            r[k] = [x for x in r[k].split(';') if x]
        # 比分合并
        r['score_before'] = [int(r.pop('score_before_a') or 0), int(r.pop('score_before_b') or 0)]
        r['score_after']  = [int(r.pop('score_after_a')  or 0), int(r.pop('score_after_b')  or 0)]
        for k in ['game_idx','rally_idx','shot_count','rally_quality']:
            r[k] = int(r[k]) if r[k] else None
        for k in ['start_ts','end_ts']:
            r[k] = float(r[k])
        r['highlight'] = r['highlight'] == 'TRUE'
        f.write(json.dumps(r, ensure_ascii=False) + '\n')
```

P0 系统上线后这步会自动化。
