# 自测记录（示例，非平台正式案例）

- 日期：2026-11-19
- 输入：本地构造的 10 文件小批次（含 2 组重复、2 条版本链、中文/特殊字符无关的纯 ASCII 名）
- 命令：

```bash
python3 scripts/libflow.py inventory --case case.json --out out
python3 scripts/libflow.py plan    --case case.json --out out
python3 scripts/libflow.py apply   --case case.json --out out
python3 scripts/libflow.py report  --case case.json --out out
python3 scripts/verify.py --out out
```

## 结果摘要

| 指标 | 值 |
|-|-|
| 源文件 | 10 |
| 归位文件 | 8 |
| copy / deduplicate | 8 / 2 |
| 失败 / 跳过 / 冲突 | 0 / 0 / 0 |
| 重复组 | 2 |
| 版本链 | 2 |
| verify | `valid: true` |

## 观察到的真实问题与修复（示例价值）

1. **相对路径解析错误**：源路径相对输入根，而落库时按 case 文件目录解析，导致 `FileNotFoundError`。修复：规划中同时记录 `src_path`（绝对路径）。
2. **逻辑名归一化不彻底**：`orion_sampleA_2023_v1` 中的 `_v1` 因正则消耗了前一个 `_` 而残留，导致同文档的不同版本未能成链。修复：改为"先分词、再按词丢弃版本/日期词"。
3. **问题回答过度自信**：最初用 token 命中判断"已回答"，几乎所有问题都返回 true。修复：按问题类型做真实推导（重复组/版本链/元数据查找），无法落地时显式 `answered: false`。

> 本文件为**技能自测示例**，用于说明方法，不冒充平台正式案例结果。
