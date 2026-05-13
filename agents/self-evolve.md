---
name: self-evolve
description: Cyclic self-review and improvement of the TGOSKits plugin — audit all files, find issues, fix, validate, repeat
skills:
  - superpowers:verification-before-completion
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
---

# Self-Evolve Agent

You audit and improve the TGOSKits Claude Code plugin itself. Each round: systematically scan all plugin files, classify issues, fix them, and validate before the next round. Default 5 rounds; user can specify a different count via `/self-evolve <N>`.

## Scope

All files under `.claude/` excluding `.claude/cache/` and `.claude/skills/` (skills are managed separately):

- `.claude/plugin.json` — plugin manifest
- `.claude/settings.json` — project settings
- `.claude/hooks/hooks.json` — hook registrations
- `.claude/commands/*.md` — slash commands
- `.claude/agents/*.md` — agent definitions
- `.claude/scripts/*.py` — Python hook/utility scripts
- `.claude/scripts/*.sh` — Bash scripts
- `.claude/config/*.toml` — CI configuration
- `.claude/hooks/*.md` — hook reference docs

## Audit Dimensions

Each round checks all 7 dimensions below. Report findings with severity (BLOCK / WARN / INFO).

### D1: Path Reference Correctness

Every file path referenced in any plugin file must exist relative to the project root.

Checklist:
- Every `commands/*.md` and `agents/*.md` listed in `plugin.json` exists on disk
- Every script path in `hooks.json` exists (e.g., `scripts/docker-check.py`)
- Every `Read` / reference to another agent file (e.g., `Read .claude/agents/pr-review.md`) resolves
- Every tool/script path in agent workflows (e.g., `.claude/scripts/syscall-diff.py`, `.claude/scripts/local-ci.sh`) exists
- Every config path (e.g., `.claude/config/docker-ci.toml`) exists
- Every source path referenced in bash commands (e.g., `os/StarryOS/kernel/src/syscalls/`) exists

How to check:
```bash
# Extract all relative file paths from plugin files and verify existence
for f in $(find .claude -type f \( -name "*.md" -o -name "*.json" \) ! -path ".claude/cache/*" ! -path ".claude/skills/*"); do
  grep -oP '([\w.-]+/)+[\w.-]+\.(md|py|sh|json|toml|rs)' "$f" 2>/dev/null || true
done | sort -u | while read path; do
  [ -f "$path" ] || echo "MISSING: $path"
done
```

### D2: Syntax & Validity

Every structured file must parse correctly.

How to check:
```bash
# JSON validity
python3 -m json.tool .claude/plugin.json > /dev/null 2>&1 || echo "BROKEN JSON: plugin.json"
python3 -m json.tool .claude/hooks/hooks.json > /dev/null 2>&1 || echo "BROKEN JSON: hooks.json"
# settings.json may be empty or absent — only check if non-empty and non-trivial
[ -s .claude/settings.json ] && python3 -m json.tool .claude/settings.json > /dev/null 2>&1 || true

# Python syntax
for f in .claude/scripts/*.py; do
  python3 -m py_compile "$f" 2>&1 || echo "BROKEN PYTHON: $f"
done

# Bash syntax
for f in .claude/scripts/*.sh; do
  bash -n "$f" 2>&1 || echo "BROKEN BASH: $f"
done
```

### D3: Frontmatter Consistency

Every agent and command file must have a valid YAML frontmatter with required fields.

Checklist:
- `name` field present and matches the filename (e.g., `agents/impl.md` → `name: impl`)
- `description` field present and non-empty
- Agents must have `skills:` and `tools:` fields
- Commands with args must have `args:` with `name`, `type`, `required` per arg
- No duplicate agent/command names across files

How to check:
```bash
# Extract frontmatter names and compare with filenames
for f in .claude/agents/*.md .claude/commands/*.md; do
  name=$(head -5 "$f" | grep '^name:' | sed 's/name: *//')
  expected=$(basename "$f" .md)
  [ "$name" = "$expected" ] || echo "NAME MISMATCH: $f has name '$name', expected '$expected'"
done

# Check for duplicates
for dir in agents commands; do
  dups=$(find ".claude/$dir" -name "*.md" -exec basename {} .md \; | sort | uniq -d)
  [ -z "$dups" ] || echo "DUPLICATE NAMES in $dir: $dups"
done
```

### D4: Cross-Reference Consistency

Agent/command/skill names referenced in one file must be registered in the plugin or globally available.

Checklist:
- Every agent name mentioned in `spawn the X agent` or `delegate to X` must exist in `plugin.json` agents or be a globally available agent
- Every skill in an agent's frontmatter `skills:` must be a known project skill (`.claude/skills/`) or a global skill (prefixed with `superpowers:` or another known plugin)
- Every command referenced (e.g., "run `/pr-prep`") must exist in `plugin.json` commands
- Tool names in frontmatter `tools:` must be valid Claude Code tool names

### D5: Command/Flag Correctness

Every `cargo xtask`, `docker`, and `git` command in agent workflows must use correct flags.

Known correct patterns:
- `cargo xtask starry qemu --arch <arch>` (no `--package`)
- `cargo xtask starry test qemu --arch <arch> -g <group> -c <case>` (test invocation)
- `cargo xtask arceos qemu --package <pkg> --arch <arch>` (ArceOS supports `--package`)
- `cargo xtask starry rootfs --arch <arch>` (rootfs preparation)
- `docker run --rm -v "$PWD:/workspace" ... tgoskits-ci bash -c '...'` (CI container)
- `git cherry-pick <range>` (not `git cherrypick`)

Anti-patterns to flag:
- `cargo xtask starry qemu --package <...>` — StarryOS does NOT support `--package`
- `cargo xtask starry qemu --package` in any form

### D6: Logical Completeness

Agent workflows must handle edge cases.

Checklist:
- After reporting an error that blocks progress, the workflow must say STOP / do NOT proceed
- Empty-result scenarios must be handled (e.g., "all syscalls already exist")
- Iteration loops must have both a counter increment and an exit condition
- User confirmation gates must say "ask user" or "wait for confirmation"

### D7: Hook Integration

Agent actions must correctly interact with registered hooks.

Checklist:
- Commands that run Docker must trigger `docker-check.py` (i.e., command contains "docker")
- Commands that create PRs (`gh pr create`, `git push`) must pass through `pre-pr-gate.py` gates
- The `pre-pr-gate.py` script must check for `local-ci.sh` having been run before allowing PR creation

---

## Workflow

### Round N (repeat for specified rounds, default 5):

#### Step 1: Audit

Read ALL plugin files (exhaustive — don't skip any). Check every file against the 7 dimensions above in order D1 → D7.

#### Step 2: Classify & Report

For each finding, assign:
- **Severity**: BLOCK (functional error — plugin won't work) / WARN (design issue — works but fragile) / INFO (minor improvement)
- **Dimension**: D1–D7
- **File & location**: exact file and line/section

Report after classification:
> "Self-evolve round N/5: found X BLOCK, Y WARN, Z INFO."

#### Step 3: Fix

Fix ALL BLOCK items first, then WARN items. Apply minimal, targeted changes — fix only the issue, no refactoring.

#### Step 4: Validate

After all fixes in a round:
```bash
# JSON
python3 -m json.tool .claude/plugin.json > /dev/null && echo "plugin.json: OK"
python3 -m json.tool .claude/hooks/hooks.json > /dev/null && echo "hooks.json: OK"

# Python
for f in .claude/scripts/*.py; do python3 -m py_compile "$f" && echo "$f: OK"; done

# Bash
for f in .claude/scripts/*.sh; do bash -n "$f" && echo "$f: OK"; done
```

If validation fails, fix the failures and re-validate before moving to the next round.

#### Step 5: Report round summary

> "Round N complete: fixed X BLOCK, Y WARN. Z items deferred. Moving to round N+1."

### Final Report

After all rounds:

```markdown
## Self-Evolve Complete

**Rounds executed**: N
**Total issues fixed**:
- BLOCK: X
- WARN: Y
- INFO: Z

### Round-by-round
| Round | BLOCK | WARN | INFO | Key fixes |
|-------|-------|------|------|-----------|
| 1 | ... | ... | ... | ... |

### Remaining (deferred or at limit)
- <any outstanding issues>
```

---

## Rules

- Read EVERY file in scope each round — don't rely on memory from previous rounds
- Fix BLOCK items immediately and atomically — one fix per edit
- Don't introduce new issues while fixing — keep changes minimal
- Validate syntax after EVERY batch of fixes, not just at the end
- If a round finds 0 issues, report clean but still complete all remaining rounds (issues may be introduced by fixes in later rounds, or by external changes)
- Do NOT modify `.claude/skills/` files (managed separately)
- Do NOT modify `.claude/cache/` files (runtime artifacts)
- If the same WARN issue persists for 2 consecutive rounds without a clear fix, defer it and note in the final report
