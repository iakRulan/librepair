# 输出字段字典

每个案例在 `/app/output` 下产出以下文件。字段是"可复核"的最小集合：足够让另一个人（或另一个 Agent）重跑并判断对错。

## `result.json`

| 字段 | 类型 | 说明 |
|-|-|-|
| `case_id` | string | 输入标识，必须与 `case.json` 一致（缺失时回退到文件名/目录名） |
| `task` | string | 固定为 `research-file-library-repair` |
| `status` | `complete` \| `partial` | 有失败操作时为 `partial`，并在 `uncertainty` 说明 |
| `deliverables` | object | 各产物相对路径 |
| `counts.source_files` | int | 源文件总数 |
| `counts.organized_files` | int | 归位后文件数（去重后 ≤ source_files） |
| `counts.operations` | object | 各动作计数（copy/deduplicate/skip/conflict） |
| `counts.failed` / `skipped` / `conflicts` | int | 失败/跳过/冲突数 |
| `counts.duplicates` / `version_chains` | int | 重复组数、版本链数 |
| `answers` | array | 对案例问题的回复（见下） |
| `uncertainty` | array | 明确的限制与不确定项 |
| `sources` | array | 输入根路径 |
| `changes` | array | 关键步骤记录 |

## `operations.json`

每个操作对象：

| 字段 | 说明 |
|-|-|
| `op` | `copy` / `deduplicate` / `skip` / `conflict` |
| `src` / `src_path` | 相对路径 / 绝对路径 |
| `dest` | 归位后的相对路径 |
| `src_sha256` / `dest_sha256` | 复制前后的哈希，用于证明未损坏 |
| `verified` | 目标哈希是否等于源哈希 |
| `status` | `ok` / `hash_mismatch` / `failed` / `skipped` / `conflict` |
| `reason` / `error` | 非 ok 时的原因 |

## `index.json`

- `files[]`：`rel` / `kind` / `sha256` / `size` / `metadata`
- `version_chains[]`：`logical` / `members[]`（rel+sha256+mtime）/ `oldest` / `newest`
- `duplicates[]`：`sha256` / `members[]` / `canonical`

## `metadata.json`

- `metadata[rel]`：字段值
- `provenance[rel]`：字段出处（`filename:<pattern>` / `content:first_text_line` / `sidecar:<name>`）

## `answers.json`

每条：`id` / `question` / `answered` / `method` / `answer` / `note`。
`answered: false` 时必须带 `note` 说明为何无法依据索引回答——不允许编造。
