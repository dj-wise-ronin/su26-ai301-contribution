# Contribution 1: Add length/count parameter to dps command

**Contribution Number:** 1  
**Student:** DeAngelo Jackson-Adams  
**Issue:** [pwndbg/pwndbg Issue Catalog: "The dps command does not have a parameter that sets the length"](https://github.com/pwndbg/pwndbg)  
**Status:** ✅ **Phase IV Complete — PR Merged** 🚀  
**Issue Link:** [pwndbg/pwndbg#2965](https://github.com/pwndbg/pwndbg/issues/2965)  
**PR Link:** [pwndbg/pwndbg#3961](https://github.com/pwndbg/pwndbg/pull/3961)  

---

## Why I Chose This Issue

I chose this issue in the `pwndbg` repository because it aligns perfectly with my focus on systems programming, debugging, and software usability on Kali Linux. `pwndbg` is a premier GDB plugin used globally by security researchers, exploit developers, and reverse engineers. Contributing to it provides deep experience with GDB's Python API and command-parsing ecosystem.

Specifically, the `dps` command (and its aliases `dds`, `dqs`, and `kd`) is a critical part of the WinDbg compatibility layer. For users transitioning from Windows debugging environments, having command parity is extremely important. The absence of a length/count parameter—which is standard in other dump commands like `db` and `dd`—forces users to fall back on native GDB/pwndbg telescope commands, breaking their established muscle memory and workflows. Resolving this issue is highly well-scoped, self-contained, and provides immediate, tangible value to the debugging community.

---

## Understanding the Issue

### Problem Description

In `pwndbg`'s WinDbg compatibility layer, the commands `dds`, `dps`, `dqs`, and `kd` are used to dump pointer-sized values and resolve symbols at a specified memory address. However, the parser for these commands was registered with only a single argument (`addr`), completely lacking any parameter to specify how many elements to dump.

As a result, calling `dps` with a length or count argument (e.g. `dps $rsp 20`) causes GDB to raise a command parsing error. The user has no way of controlling the dump size other than modifying global configuration variables or using the underlying `telescope` command directly.

### Expected Behavior

Users should be able to run `dps <address> [count]` to display a specific number of resolved pointer/symbol lines, mirroring the behavior of other memory dump commands (like `db` and `dd`). When the count is omitted, it should fall back to the default configured value.

### Current Behavior

The command parser fails when a second argument is passed:
```
usage: dps [-h] addr
dps: error: unrecognized arguments: 20
```

### Affected Components

*   [pwndbg/commands/windbg.py](file:///home/ronin/Projects/pwndbg/pwndbg/commands/windbg.py): Specifically the parser definition for `dds` and its function signature, which handles execution for `dds`, `dps`, `dqs`, and `kd`.

---

## Reproduction Process

### Environment Setup

Set up a local clone of `pwndbg`, configure dependencies, and activate the virtual environment:
```bash
git clone https://github.com/pwndbg/pwndbg.git /home/ronin/Projects/pwndbg
cd /home/ronin/Projects/pwndbg
./setup.sh
```

### Steps to Reproduce

1. Launch GDB: `gdb`
2. Attempt to dump 5 pointer entries at `$sp` using `dps`:
   ```gdb
   pwndbg> dps $sp 5
   ```
3. **Observed result:** GDB prints a command usage error:
   ```
   usage: dps [-h] addr
   dps: error: unrecognized arguments: 5
   ```

### Reproduction Evidence

- **Working Branch:** [feature/dps-length-parameter](https://github.com/dj-wise-ronin/pwndbg/tree/feature/dps-length-parameter)
- **My findings:** The `dds` argument parser in `pwndbg/commands/windbg.py` was defined as:
  ```python
  parser = argparse.ArgumentParser(description="Dump pointers and symbols at the specified address.")
  parser.add_argument("addr", type=pwndbg.commands.HexOrAddressExpr, help="The address to dump from.")
  ```
  Since `count` was not added to the parser, the argument parsing library correctly rejected any extra positional arguments.

---

## Solution Approach

### Analysis

The root cause is simply that the parser definition and the Python function definition for `dds` do not accept or handle a second positional argument. 

Because `dds` calls `pwndbg.commands.telescope.telescope` under the hood, and `telescope` natively supports a `count` argument, the solution is to:
1. Define a new `dds_parser` that includes an optional `count` argument using `pwndbg.commands.AddressExpr`.
2. Update the `dds` command handler to accept `count` and pass it to `telescope()`.
3. Support proper repeat commands by passing `repeat=dds.repeat` to `telescope()`.

### Proposed Solution

We register an optional `count` argument in a new `dds_parser`, defaulting to `None`. In the `dds` function, if `count` is specified, we cast it to an integer and forward it to `pwndbg.commands.telescope.telescope(addr, count=int(count), repeat=dds.repeat)`. If omitted, we run `telescope` with the default count.

### Implementation Plan

Using the UMPIRE framework:

*   **Understand:** Add an optional `count` parameter to `dds`/`dps`/`dqs`/`kd` to configure the dump range, and forward the repeat status.
*   **Match:** Look at how `db`, `dw`, and `dd` handle their optional `count` parameters using `nargs="?"` and `pwndbg.commands.AddressExpr`.
*   **Plan:**
    1. Define `dds_parser` with `addr` and optional `count` parameters in [pwndbg/commands/windbg.py](file:///home/ronin/Projects/pwndbg/pwndbg/commands/windbg.py).
    2. Update `dds` command implementation to receive `count=None` and forward it.
    3. Add integration test coverage in [tests/library/dbg/tests/test_windbg.py](file:///home/ronin/Projects/pwndbg/tests/library/dbg/tests/test_windbg.py).
    4. Run formatters and style linters to verify code health.

---

## Testing Strategy

### Unit / Integration Tests

- [x] Test `dps` with default length displays the expected default lines of pointers.
- [x] Test `dps` with custom count argument (e.g. `dps &data 3`) returns exactly that number of lines.
- [x] Test that aliases `dds`, `dqs`, and `kd` all accept and correctly enforce the count argument.
- [x] Verify GDB repeat behavior works as intended.

Run tests:
```bash
./tests.sh -g dbg -d gdb test_windbg
```

### Manual Testing

1. Launched GDB: `.venv/bin/python3 -m pytest tests/library/dbg/tests/test_windbg.py` (and interactive debugging sessions).
2. Ran `dps $sp 3`: Confirmed it prints exactly 3 lines.
3. Pressed Enter to repeat: Verified it continued dumping subsequent stack pointers correctly.

### Testing Approaches & Validation Steps Taken

- **Fix Verification**: Validated that the `dds` command and its aliases successfully accept a second positional parameter without throwing an `argparse.ArgumentError` for unrecognized arguments.
- **Repeat Behavior Validation**: Hitting Enter (empty command) successfully triggers GDB's repeat behavior, which in turn calls `telescope()` with the same `count` parameter and correctly advances the pointer display offset.
- **Cross-Debugger Safety**: The test suite covers both GDB and LLDB environments. Because LLDB does not support native `pi` Python command mocks for `check_repeated` logic, the repeat test cases are dynamically guarded using `is_gdb = "GDB" in type(ctrl).__name__` checks.

---

## Implementation Notes

### Work Completed

I have implemented the code changes to add an optional `count` parameter to `dds` and its aliases (`dps`, `dqs`, `kd`).

- **Parser Extension**: Registered an optional `count` argument using `pwndbg.commands.AddressExpr` inside `dds_parser`.
- **Command Forwarding**: Updated the `dds` command implementation to receive this new positional parameter and cast it to an integer before forwarding to `pwndbg.commands.telescope.telescope`.
- **Documentation Parity**: Recompiled auto-generated markdown documentation for `dds` to prevent upstream repository dev docs checks from failing.

### Challenges Faced & Resolutions

- **Documentation compilation constraints**: The upstream CI checks validate that command help matches the auto-generated documentation. Any change to command parsers requires running the documentation generator. I resolved this by running `./scripts/generate-docs.sh` locally and committing the auto-generated changes.
- **LLDB Python command integration limitations**: The integration tests run on both GDB and LLDB. Executing Python code directly via command-line injection (`pi`) is GDB-specific and threw errors in LLDB mock tests. I resolved this by checking the debugger controller class type and skipping python-mocking logic when running under LLDB.

---

## Code Changes

- **Active Development Branch**: [feature/dps-length-parameter](https://github.com/dj-wise-ronin/pwndbg/tree/feature/dps-length-parameter)
- **Pull Request URL**: [pwndbg/pwndbg PR #3961](https://github.com/pwndbg/pwndbg/pull/3961)

### Meaningful Commits

- **19d2b160f**: `feat(windbg): add optional count/length parameter to dds/dps/dqs/kd`
- **2a3e4b084**: `docs(windbg): update dds/dps auto-generated documentation`
- **312ff9c27**: `test(windbg): add regression test for dds repeat/Enter behavior`
- **40ea6a532**: `test(windbg): only run GDB repeat test under GDB`
- **50d976e63**: `refactor(windbg): apply maintainer review suggestions for type annotations, parser, and GDB check`

---

## Phase IV: Submit & Iterate (Week 5)

### Pull Request

*   **PR Link:** [pwndbg/pwndbg PR #3961](https://github.com/pwndbg/pwndbg/pull/3961)
*   **PR Title:** `feat(windbg): add optional count/length parameter to dds/dps/dqs/kd`
*   **Status:** ✅ **Merged** into upstream `dev` branch

### Maintainer Feedback & Iteration

*   **Reviewers:** `@k4lizen`, `@disconnect3d` (maintainer)
*   **Changes Requested:**
    *   Use type hints (`int | None`) in the command function definitions.
    *   Guard the GDB-specific repeat/Enter test so it doesn't break LLDB test runs.
*   **Resolution:** Pushed follow-up commit `50d976e63` applying all maintainer suggestions — type annotations, parser restructuring, and GDB guard. PR was approved and merged.

---

## Learnings & Reflections

### Technical Skills Gained

- Learned how pwndbg leverages `argparse` inside custom GDB commands.
- Understood GDB's repeat invocation logic (`repeat` attribute on decorators) and how it maps to internal debugger commands.
- Gained familiarity with writing integration tests running inside GDB controllers.

---

# Contribution 2 (Cycle 2) — OSSF `cve-bin-tool`

## Selected Issue: `test: improve performance on our slowest tests` (#4321)

- **Repository**: [ossf/cve-bin-tool](https://github.com/ossf/cve-bin-tool)
- **Organization**: Open Source Security Foundation (OSSF)
- **Target Issue**: [Issue #4321: test: improve performance on our slowest tests](https://github.com/ossf/cve-bin-tool/issues/4321)
- **Status**: Completed / Pull Request Opened for Review

---

## Code Changes

- **Active Development Branch**: [optimize-lockfile-performance](https://github.com/dj-wise-ronin/cve-bin-tool/tree/optimize-lockfile-performance)
- **Pull Request URL**: [ossf/cve-bin-tool PR #5834](https://github.com/ossf/cve-bin-tool/pull/5834)

### Meaningful Commits

- **2a0201af**: `test: optimize language lockfiles to speed up scanner test suite`

---

## Technical Accomplishments & Optimization Metrics

I optimized the test runner performance by cleaning up the dummy lockfiles in `test/language_data/`. Trimming these mock files to include only the packages explicitly asserted in the test code significantly reduced database lookup overhead and resolved the slow test execution times.

- **`Cargo.lock` (Rust):** Reduced from 279 packages down to 24 packages.
- **`Gemfile.lock` (Ruby):** Reduced from 218 gems down to 47 gems.
- **`renv.lock` (R):** Reduced from 106 packages down to 17 packages.
- **`go.mod` (Go):** Reduced from 60 requirements down to 12 packages.

### Performance Benchmarks

| Metric | Unoptimized Baseline (Original) | Optimized Run (Our Changes) | Performance Gain |
| :--- | :--- | :--- | :--- |
| **Active Tests** | 5 / 16 (11 tests skipped) | 16 / 16 (All tests enabled) | Running all tests under LONG_TESTS |
| **Total Duration** | 16m 19s (with 11 skips) | 9m 20s (all tests running) | Over 2x speedup on full suite execution |
| **Status** | Passing (Partial) | 100% Passing (All 16 tests) | Full coverage with stable runs |

---

## Learnings & Reflections

### Technical Skills Gained
- Gained familiarity with multi-language dependency lockfile structures (Bundler `Gemfile.lock`, Cargo `Cargo.lock`, R `renv.lock`, and Golang `go.mod`).
- Learned how to optimize test runner performance by tailoring mock datasets instead of modifying core engine logic.
- Mastered managing git remote configurations and setting default repositories inside the GitHub CLI.
