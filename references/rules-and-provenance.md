# 规则解析与字段出处

## 规则来源与优先级

`libflow` 从 `case.json` 里解析规则，按以下键名查找（大小写/连字符不敏感）：

| 目标 | 接受的键 |
|-|-|
| `layout` | `layout`, `directory_rules`, `organization`, `convention` |
| `naming` | `naming`, `naming_rules` |
| `deduplicate` | `deduplicate`, `dedup` |
| `keep_source` | `keep_source` |
| `template` | 任何含 `template` 的键，值为 `{type}/{year}/{ext}/{logical}` 形式的路径模板 |
| `metadata_fields` | `metadata_fields`, `schema`, `fields` |

未给出规则时默认：`layout=by_type`、`naming=keep`、`deduplicate=true`、`keep_source=true`，并在 `result.json.uncertainty` 中记录。

## 目录布局

- `by_type`：`document/` `table/` `image/` `archive/` `code/` `pdf/` `other/`
- `by_year`：`{year}/<name>`
- `flat`：全部平铺
- `template`：按 `{type} {year} {ext} {logical}` 组合

## 命名规范化

`naming=normalize` 时：空白转下划线，非法字符转下划线，连续下划线合并，保留中文与常见扩展名。

## 元数据字段字典

| 字段 | 触发模式（文件名） | 出处标记 |
|-|-|-|
| `project` | `proj`/`project` + 值 | `filename:<pattern>` |
| `year` | `20xx` | `filename:<pattern>` |
| `sample` | `sample`/`samp`/`s` + 值 | `filename:<pattern>` |
| `instrument` | `instrument`/`inst`/`device` + 值 | `filename:<pattern>` |
| `author` | `author`/`by` + 值 | `filename:<pattern>` |
| `title_guess` | 文本类文件首行 | `content:first_text_line` |
| 任意 sidecar 字段 | 同名 `.json`/`.yaml`/`.txt` | `sidecar:<name>` |

**原则**：出处必须能让人回到原始文件核对。无法落地的字段不写。

## 版本链判定

1. 文件名归一化为逻辑名：小写、非字母数字转空格、去掉版本词（`v1`/`final`/`draft`/`copy`/`backup`/`rev*`/`tmp`）、去掉纯数字与日期。
2. 同逻辑名且**哈希不同**的文件构成候选链。
3. 按 `mtime` 排序，最早为 `oldest`，最晚为 `newest`。
4. 若案例提供了 `version_notes`，作为行政记录合并，并标记 `source=administrative_note`。

## 去重判定

- 仅按 sha256 分组；同组保留最早（mtime 最小）者为 `canonical`。
- 去重**不删除**任何源文件，只在归位时跳过冗余副本，并在 `operations.json` 记录 `kept` 与 `sha256`。
