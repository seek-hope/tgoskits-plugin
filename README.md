# TGOSKits Plugin

Claude Code project-local plugin for the [TGOSKits](https://github.com/rcore-os/tgoskits) OS/kernel monorepo.

Provides Docker-based local CI, automated hooks for activity logging and PR gates, slash commands for testing and PR workflow, and specialized agents for bug hunting, PR review, test generation, and driver auditing.

## Installation

Copy the entire directory to `.claude/` in your TGOSKits workspace:

```bash
git clone https://github.com/seek-hope/tgoskits-plugin.git /tmp/tgoskits-plugin
cp -r /tmp/tgoskits-plugin/* .claude/
```

Or add as a git submodule:

```bash
git submodule add https://github.com/seek-hope/tgoskits-plugin.git .claude/
```

## Structure

```
.claude/
├── plugin.json              # Plugin manifest
├── settings.json            # Hook registrations
├── hooks/
│   ├── hooks.json           # Command-based hooks (PreToolUse, PostToolUse, Stop)
│   └── pre-pr-gate.md       # PR gate reference documentation
├── commands/
│   ├── test.md              # /test — quick/full/single-arch CI testing
│   └── pr-prep.md           # /pr-prep — 5-phase PR workflow
├── agents/
│   ├── bug-hunt.md          # 5-phase bug discovery → PR (2D classification)
│   ├── pr-review.md         # Semantic code review with auto-fix
│   ├── test-gen.md          # Linux-reference test generation
│   └── driver-audit.md      # 4-layer driver architecture audit
├── scripts/
│   ├── local-ci.sh          # Docker image management + CI runner
│   ├── docker-check.py      # Docker daemon pre-check hook
│   ├── pre-pr-gate.py       # PR/push gate hook
│   ├── post-tool-use-log.py # Activity logger (log.md)
│   ├── stop-hook.py         # Session-end journal generator
│   ├── journal-generator.py # log.md → [task]-journal.md
│   └── syscall-diff.py      # Linux vs OS syscall behavior comparison
├── config/
│   └── docker-ci.toml       # CI matrix and image configuration
└── cache/
    └── .gitkeep
```

## Prerequisites

- Docker (CI tests run in containers)
- `gh` CLI (for PR creation)
- Git remotes: `upstream` pointing to `rcore-os/tgoskits`, `origin` pointing to your fork

## Usage

```bash
# Quick CI check
/test quick

# Full CI matrix (all OSes, all architectures)
/test full

# Single-architecture test
/test starry aarch64

# Full PR workflow
/pr-prep "feat(axfs): add fallocate support"
```

## License

Apache-2.0
