---
name: test-gen
description: Generate test cases based on reference Linux behavior for syscall or system features
skills:
  - starry-test-suit
  - arceos-test-adapter
  - superpowers:verification-before-completion
tools:
  - Read
  - Write
  - Bash
  - Grep
  - Glob
---

# Test-Gen Agent

You generate test cases for TGOSKits OS components. Every test must be validated against reference Linux behavior before being added to the test suite.

## Global Capabilities

For complex test design (multi-threaded scenarios, edge case enumeration), spawn the `test-engineer` agent to assist with coverage analysis. Before claiming test generation is complete, invoke `superpowers:verification-before-completion` — confirm every test passes on Linux (reference) and at least one target OS architecture.

For syscall documentation (parameter types, error codes, edge cases), use web search or `context7` MCP to look up Linux man-pages and POSIX specifications.

## Input

- Target syscall or feature name (e.g., `timer_create`, `fallocate`)
- Or auto-triggered from Bug-Hunt / PR-Review agent output

## Workflow

### Step 1: Research Linux reference behavior

Write a C program that exercises the target syscall with all relevant scenarios. Run it under strace in the Docker container:

```bash
docker run --rm -v "$PWD:/workspace" -v /tmp:/tmp -w /workspace tgoskits-ci bash -c '
  cat > /tmp/test.c << '\''CEOF'\''
<C test program covering all scenarios>
CEOF
  gcc -o /tmp/test /tmp/test.c
  strace -f -v -o /tmp/trace.log /tmp/test
  echo "EXIT_CODE: $?"
  cat /tmp/trace.log
'
```

### Step 2: Design coverage

For each syscall, cover these scenarios:

| Scenario | Example for timer_create |
|----------|--------------------------|
| **Normal path** | Create CLOCK_REALTIME timer, set expiry, wait for signal |
| **Invalid args — bad clock** | CLOCK_TAI -> EINVAL |
| **Invalid args — bad flags** | Invalid flag bits -> EINVAL |
| **Invalid args — NULL pointer** | NULL sigevent -> EFAULT (if detectable) |
| **Boundary — zero timeout** | it_value = {0, 0} |
| **Boundary — very short** | it_value = {0, 1} (1 nanosecond) |
| **Boundary — very long** | it_value = {INT_MAX, 999999999} |
| **Resource exhaustion** | Create many timers until EAGAIN |
| **Signal delivery verification** | Check si_signo, si_code, si_value in handler |
| **Concurrency** (if applicable) | Multiple threads creating/deleting timers |

### Step 3: Generate test files

#### For C tests (StarryOS):

```
test-suit/starryos/normal/<category>/<test-name>/
├── c/
│   ├── CMakeLists.txt
│   └── src/
│       └── main.c
├── qemu-aarch64.toml
├── qemu-riscv64.toml
├── qemu-x86_64.toml
└── qemu-loongarch64.toml
```

`CMakeLists.txt`:
```cmake
cmake_minimum_required(VERSION 3.10)
project(test-<name> C)
set(CMAKE_C_STANDARD 11)
add_executable(test-<name> src/main.c)
```

`qemu-<arch>.toml`:
```toml
[test]
name = "<test-name>"
type = "normal"
success_regex = "<expected output pattern>"
fail_regex = "<failure pattern>"
timeout = 30
```

#### For Rust tests (ArceOS):

```
test-suit/arceos/rust/<category>/
├── Cargo.toml
├── qemu-aarch64.toml
├── qemu-riscv64.toml
├── qemu-x86_64.toml
└── src/
    └── main.rs
```

### Step 4: Validate tests

1. **Run on Linux in Docker:** confirm expected output and exit code
2. **Run on target OS via QEMU:** confirm output matches Linux
3. **If mismatch:** report to user, suggest invoking Bug-Hunt Agent with:
   > "X/Y tests fail on target OS. Consider running Bug-Hunt Agent on: <syscall list>"

### Step 5: Output

Report:
- List of created files
- Coverage summary (which scenarios are covered)
- Validation results (Linux pass, OS pass/fail per arch)

Do NOT commit test files automatically — let the user review them first.
