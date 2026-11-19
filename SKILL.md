---
name: librepair
description: "Repair a research file library: inventory files with content hashes, organize them into a convention-driven tree, deduplicate by content (not filename), reconstruct version chains, extract metadata with provenance, build a search index, and emit a re-runnable manifest + operations log + result.json. Trigger on: '科研资料管理', '资料库整理', '文件库修复', '版本链', '去重', '文件清单', 'file library repair', 'research data management', 'organize my files', 'deduplicate by hash'. Not for: paper reading/writing, figure drawing, or general cloud storage."
---

# librepair — Research File-Library Repair with Evidence

> **脚本路径**：本技能安装于 `/skills/weekly-skill/`，脚本绝对路径为 `/skills/weekly-skill/scripts/`（工作目录是 `/app/work`，不要用相对路径）。

Turn a messy batch of research files into a library that can be trusted, searched and handed over. The failure mode this skill exists to prevent: a "professional-looking" reorganization that moves files without a manifest, overwrites same-named files, silently skips unreadable paths, guesses metadata from filenames, and cannot be re-run or checked.

**Method in one line:** hash every source file, decide placement/dedup/version from content and rules, copy (never move) into the organized tree, then write a manifest + operations log + index + `result.json` that a reviewer can re-run and verify.

## When to use

- Input: a case file (default `/input/case.json`) describing the source files/directory, the organization rules, version notes, aliases and questions; plus the actual files.
- Output: under `/app/output` — `library/` (organized tree), `manifest.json`, `index.json`, `metadata.json`, `operations.json`, `answers.json`, `result.json`.
- Works for any mix of PDFs, documents, tables, images, archives and code; handles deep directories, non-ASCII filenames and permission errors without silent skips.

## Pipeline (run in order; do not skip phase 3 or 4)

All scripts are in `scripts/` next to this file. Run from `/app/work`.

### Phase 1 — Inventory (hash everything, no guessing)

```bash
python3 /skills/weekly-skill/scripts/libflow.py inventory --case /input/case.json --out /app/work
```

Produces `work/inventory.json`: for every source file — absolute path, path relative to the input root, size, mtime, permission bits, detected kind and **sha256**. Read this file; it is the single source of truth. Never re-walk the raw tree in your head.

If a path is unreadable, `libflow` records it in the operation log at apply time; do not silently drop it. If the case file is a directory rather than JSON, the script synthesizes a case from it.

### Phase 2 — Plan (decide, then show your work)

```bash
python3 /skills/weekly-skill/scripts/libflow.py plan --case /input/case.json --out /app/work
```

Produces `work/plan.json` with:
- resolved rules (layout, naming, dedup, keep-source, optional `{type}/{year}/...` template),
- one action per source file: `copy` / `deduplicate` / `skip` / `conflict`,
- duplicate groups (by sha256, not by name) and version chains (logical name + mtime, validated by hash).

Check the plan before applying: every source file must have exactly one action. A `conflict` means two files map to the same destination — fix the plan/rules, do not let the script overwrite.

### Phase 3 — Apply (copy, never move; log everything)

```bash
python3 /skills/weekly-skill/scripts/libflow.py apply --case /input/case.json --out /app/work
```

Copies sources into `work/library/`, normalizing names if asked, deduplicating by content, and writes:
- `operations.json` — every action with `src`, `dest`, source and destination sha256, `verified` flag and `status` (`ok` / `hash_mismatch` / `failed` / `skipped` / `conflict`),
- `index.json` — searchable index (files + metadata + version chains + duplicates),
- `metadata.json` — per-file metadata **with provenance** (which field came from filename pattern / content / sidecar),
- `answers.json` — answers to the case's questions, each grounded in the index or explicitly marked unanswered,
- `manifest.json` — the organized tree with a sha256 per file.

### Phase 4 — Report + verify (the quality bar)

```bash
python3 /skills/weekly-skill/scripts/libflow.py report --case /input/case.json --out /app/work
python3 /skills/weekly-skill/scripts/verify.py --out /app/work
```

`report` writes `result.json` with `case_id`, `status`, deliverables, counts, answers, uncertainty, sources and changes. `verify` independently re-hashes the organized tree, confirms every source file is accounted for, and fails (non-zero) on missing files, hash mismatches, unaccounted sources or failed operations.

**Copy the deliverables to `/app/output`** so the platform can inspect them:

```bash
mkdir -p /app/output
cp -r /app/work/library /app/work/manifest.json /app/work/index.json \
      /app/work/metadata.json /app/work/operations.json /app/work/answers.json \
      /app/work/result.json /app/output/
```

## Rules that must hold

1. **Content over filenames.** Duplicates are decided by sha256; version chains by normalized logical name + mtime + hash, never by "final" in a filename.
2. **Never destroy the source.** The source tree is copied, not moved or deleted. If the task explicitly requires in-place reorganization, say so in `result.json.uncertainty` and still write a full before/after log.
3. **No silent skips.** Every unreadable, conflicting or failed path appears in `operations.json` with a reason. A report that says "all done" while files were skipped is a bug.
4. **Metadata carries provenance.** Each field records where it came from; a field you cannot ground is omitted, not invented.
5. **Honest answers.** A question that the index cannot ground is returned with `answered: false` and a note — never a plausible guess.
6. **Re-runnable.** Running `inventory → plan → apply → report → verify` twice on the same input produces the same manifest hashes and a `valid: true` verdict.

## Exception handling

| Situation | Behaviour |
|-|-|
| Permission denied on a file/dir | recorded as `skip`/`failed` with the OS error; counted in `result.json` |
| Two files map to one destination | `conflict` action; not overwritten |
| Hash mismatch after copy | `hash_mismatch` status; `verify` fails |
| Case has no explicit rules | default `by_type` layout, recorded in `uncertainty` |
| Case has no version notes | chains inferred from filenames + mtime, recorded in `uncertainty` |
| Case file missing/unparseable | `load_case` falls back to treating the path as a directory; if that fails, stop and report honestly |

## Self-check before submitting

```bash
python3 /skills/weekly-skill/scripts/verify.py --out /app/work
python3 -c "import json;r=json.load(open('/app/work/result.json'));print(r['case_id'], r['status'], r['counts'])"
```

Both must be clean: `valid: true`, `status: complete` (or `partial` with the limits named in `uncertainty`).