# librepair — 科研资料库与文件管理 Skill

把一批杂乱的真实科研文件，修成**可信任、可检索、可交接**的资料库：内容哈希去重、目录约定归位、版本链重建、字段出处保留、全程操作日志、可复跑校验。

## 解决什么问题

普通 Agent 做资料整理时常见的失败：只给建议不动手；移动文件不留清单/哈希/日志；同名文件直接覆盖导致版本链断裂；靠文件名猜元数据；对无权限或超大文件静默跳过却报告"全部完成"。

`librepair` 把这件事做成一条确定性的管线，每一步都留下可核对的证据。

## 安装与依赖

- Python 3.9+（仅标准库：`hashlib` / `json` / `shutil` / `argparse` / `re`）
- 无需第三方依赖；不需要联网
- 目录结构：

```
librepair/
├── SKILL.md              # 技能本体（ harness 读取）
├── README.md
├── GITHUB_URL.txt
├── references/
│   ├── output-schema.md  # 输出字段字典
│   └── rules-and-provenance.md
├── examples/
│   ├── case.example.json # 最小输入示例
│   └── selftest-summary.md
└── scripts/
    ├── libflow.py        # inventory / plan / apply / report
    └── verify.py         # 独立复核
```

## 用法（最小示例）

```bash
# 1) 盘点：对每个源文件算 sha256
python3 scripts/libflow.py inventory --case /input/case.json --out /app/work

# 2) 规划：去重组、版本链、每个文件的去向
python3 scripts/libflow.py plan    --case /input/case.json --out /app/work

# 3) 落库：复制（不移动）到 library/，并写全部证据
python3 scripts/libflow.py apply   --case /input/case.json --out /app/work

# 4) 汇总 + 复核
python3 scripts/libflow.py report  --case /input/case.json --out /app/work
python3 scripts/verify.py --out /app/work
```

输入 `case.json` 示例（完整示例见 `examples/case.example.json`）：

```json
{
  "case_id": "demo",
  "input_dir": "input",
  "rules": {"layout": "by_type", "naming": "normalize", "deduplicate": true},
  "version_notes": [{"logical": "orion log", "oldest": "legacy/v1.txt", "newest": "new/final.txt"}],
  "aliases": {"new/final.txt": "current version"},
  "questions": ["Which files are exact duplicates?"]
}
```

> 以上为**示意示例**，不是正式案例结果。

## 输出

| 文件 | 内容 |
|-|-|
| `library/` | 归位后的目录树（源文件保留，不删不改） |
| `manifest.json` | 归位树逐文件 sha256 |
| `index.json` | 可检索索引：文件、元数据、版本链、重复组 |
| `metadata.json` | 每个字段的值 + **出处**（文件名/内容/sidecar） |
| `operations.json` | 每个动作的 before/after 哈希与状态 |
| `answers.json` | 对案例问题的、有依据或明确未答的回复 |
| `result.json` | `case_id` / `status` / counts / uncertainty / sources / changes |

## 关键设计

- **内容优先**：去重按 sha256；版本链按"逻辑名 + mtime + 哈希"，不被 `final` 字样欺骗。
- **不毁原件**：只复制，不移动不删除。
- **无静默跳过**：权限、冲突、哈希不符全部落进 `operations.json`。
- **字段有出处**：元数据标注来源，无法落地的字段不写。
- **可复跑**：`verify.py` 独立重算哈希并核对每个源文件是否被交代。

## 局限

- 不做文件内容的语义理解（不判断论文结论对错）；
- 元数据抽取以文件名/sidecar/文本首行为限，复杂字段需人工补录；
- 归档包（zip）与长期保存策略需在 `rules` 中显式开启。

## 许可与版本

- 版本：libflow 1.0.0
- 纯 Python 标准库，无外部依赖。
