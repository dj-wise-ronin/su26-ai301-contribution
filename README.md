# Contribution 1: Add length/count parameter to dps command

**Contribution Number:** 1  
**Student:** DeAngelo Jackson-Adams  
**Issue:** [pwndbg/pwndbg Issue Catalog: "The dps command does not have a parameter that sets the length"](https://github.com/pwndbg/pwndbg)  
**Status:** Phase III Complete (Implementation & Testing Verified)  

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

---

## Implementation Notes

### Code Changes

- **Files modified:**
  - [pwndbg/commands/windbg.py](file:///home/ronin/Projects/pwndbg/pwndbg/commands/windbg.py)
  - [tests/library/dbg/tests/test_windbg.py](file:///home/ronin/Projects/pwndbg/tests/library/dbg/tests/test_windbg.py)

---

## Pull Request

*(To be submitted)*

---

## Learnings & Reflections

### Technical Skills Gained

- Learned how pwndbg leverages `argparse` inside custom GDB commands.
- Understood GDB's repeat invocation logic (`repeat` attribute on decorators) and how it maps to internal debugger commands.
- Gained familiarity with writing integration tests running inside GDB controllers.
