# Verilog Lint Fixer

Automatically detect and fix all lint warnings/errors in Verilog/SystemVerilog files. Iteratively runs the linter, applies fixes, and re-checks until the module is clean.

## When to Use

- User says "fix lint", "check lint", "clean up verilog", "lint this module", or similar
- User provides a Verilog (.v) or SystemVerilog (.sv) file and wants it lint-clean
- User asks to improve Verilog code quality

## Prerequisites — Verilator Installation

Verilator is the primary linter. Install it before first use.

### Windows (current platform)

```bash
# Option A: winget (recommended)
winget install Verilator

# Option B: if winget fails, use scoop
scoop install verilator

# Option C: manual — download from https://veripool.org/verilator/
```

After installation, verify:
```bash
verilator --version
```

If `verilator` is not available, fall back to `iverilog` for basic syntax checking:
```bash
winget install IcarusVerilog
```

### Linux / macOS

```bash
# Ubuntu/Debian
sudo apt-get install verilator

# macOS
brew install verilator
```

## External Report Integration

Besides running Verilator/iverilog directly, this skill can read lint/check reports produced by **other EDA tools** and apply fixes based on those reports.

### Supported Report Formats

| Tool | Report Type | How to Parse |
|---|---|---|
| Spyglass | Lint report (.rpt) | Parse `Module/File/Line/Severity/Rule/Message` columns |
| Design Compiler | Lint log | Parse `[Warning|Error]\s+at\s+<file>:<line>` patterns |
| Vivado | Messages log | Parse `[<severity>\s+<code>]\s+<file>:<line>: <msg>` |
| IUS/Xcelium | IRUN log | Parse `\w+:\s+<file>,\s*<line>:\s*<msg>` |
| VCS | Lint log | Parse `Warning-\w+\s+at\s+<file>:<line>` |
| Generic CSV/Text | Any structured report | Line-oriented parsing — extract file, line, severity, message |

### How to Use

User provides the report file path alongside the Verilog source:

```
fix lint in rtl/top.v using report spyglass_lint.rpt
```

Or simply:
```
fix lint with reports/lint_report.txt
```

### Report-Based Fix Workflow

1. **Read the report** — identify all Error/Warning entries, extract file path, line number, rule code, and description
2. **Cross-reference with source** — read each flagged file, locate the exact lines mentioned in the report
3. **Apply fixes** — use the same fix rules as Verilator-based fixing (Step 4), but guided by the external report's rule descriptions
4. **Verify** — if Verilator is available, run `--lint-only -Wall` as a sanity check after fixes; if not, at minimum re-read the modified code to confirm correctness

### Priority Merging

When both an external report AND Verilator output are available:
- Deduplicate: if the same line/issue is flagged by both tools, fix once
- Complementary: if one tool catches something the other missed, fix all unique issues
- Conflict: if tools disagree (one says fix, other says it's fine), prefer the stricter interpretation and note the conflict for the user

## Workflow

### Step 1: Identify Target Files

Determine the Verilog file(s) to lint:
- If user specifies a file path → use that
- If user specifies a directory → find all `.v` and `.sv` files recursively
- If user specifies a module name → search for the matching file

```bash
# Example: find all verilog files in a directory
find . -name "*.v" -o -name "*.sv" | head -50
```

### Step 2: Run Lint Check

Run Verilator in lint-only mode:

```bash
verilator --lint-only -Wall <file.v>
```

Key flags:
- `--lint-only` : only check, don't compile
- `-Wall` : enable all warnings
- `-Wno-<WARNING>` : suppress specific warnings if user requests
- `--top-module <name>` : specify top module if auto-detection fails
- `-I<dir>` : include directories for `include` files

If Verilator is not available, fall back to iverilog syntax check:
```bash
iverilog -t null -Wall <file.v>
```

Capture ALL output (stdout + stderr).

### Step 3: Parse Lint Output

Verilator output format:
```
%Error:   file.v:42:3: Assignment in condition
%Warning-WIDTH: file.v:15:8: Operator assign expects 8 bits, but assign's RHS yields 16 bits
%Warning-UNUSED: file.v:20:5: Signal is not used: 'unused_sig'
```

Extract from each line:
- **Severity**: `%Error` or `%Warning-<TYPE>`
- **File**: source file path
- **Line**: line number
- **Column**: column number
- **Message**: description of the issue

### Step 4: Fix Each Lint Issue

Common lint fixes (apply in order of severity — Errors first, then Warnings):

| Lint Type | Typical Fix |
|---|---|
| `%Error` | Must fix — usually syntax or semantic errors |
| `UNUSED` | Remove unused signals/variables, or add `/* verilator lint_off UNUSED */` if intentional |
| `UNDRIVEN` | Add driver logic, or initialize the signal |
| `UNOPTFLAT` | Break combinational loops, add pipeline registers |
| `WIDTH` | Match bit widths explicitly with cast or resize: `assign a = 8'(b);` |
| `CASEINCOMPLETE` | Add `default` case to `case` statement |
| `CASEOVERLAP` | Remove overlapping case items |
| `TIMESCALEMOD` | Add `` `timescale 1ns/1ps `` directive |
| `BLKSEQ` | Use non-blocking assignments (`<=`) in sequential blocks |
| `SYNCASYNCNET` | Add synchronizer for async signals |
| `VARHIDDEN` | Rename inner variable to avoid shadowing outer scope |
| `DECLFILENAME` | Rename file to match module name, or add `/* verilator lint_off DECLFILENAME */` |
| `UNUSEDGENVAR` | Remove unused genvar declarations |
| `IFDEPTH` | Simplify nested if-else chains |

**Fix Rules:**
1. Read the file before modifying — understand the context
2. Make minimal, targeted changes — don't rewrite entire modules
3. Preserve original logic and intent
4. If a warning is intentional (e.g., unused port in a testbench), add `/* verilator lint_off <TYPE> */` pragma instead of removing
5. Always maintain proper Verilog syntax after each fix

### Logic Preservation Verification

After all fixes are applied, **verify that the original circuit logic has not changed**:

1. **Signal connectivity check** — ensure all input/output port connections remain identical
2. **Assignment equivalence** — verify every `assign`, `always` block still produces the same outputs for the same inputs
3. **No dead code introduction** — fixes should not make previously-driven signals undriven or vice versa
4. **Bit-width consistency** — width changes (e.g., casts) must not truncate or extend meaningful bits in a way that alters behavior
5. **Quick sanity method** — if the user has a simulation testbench, suggest running it post-fix:
   ```bash
   iverilog -o sim.vvp testbench.v top.v && vvp sim.vvp
   ```
   Compare output against pre-fix golden results.
6. **Diff review** — always show a summary of what changed (line-level) so the user can visually confirm no logic drift:
   ```bash
   git diff <file>   # if git tracked
   diff <original> <fixed>  # otherwise
   ```

**Hard rule**: If any fix risks altering functional behavior (e.g., changing operator precedence, modifying sensitivity list, reordering case branches with side effects), **flag it explicitly** and ask the user before applying. Safe fixes (adding casts, removing unused signals, adding default cases, pragma suppressions) can proceed automatically.

### Step 5: Re-run Lint Check

After applying fixes, run Verilator again:

```bash
verilator --lint-only -Wall <file.v>
```

### Step 6: Iterate

Repeat Steps 3-5 until:
- **Zero Errors, Zero Warnings** → ✅ Done! Report success.
- **Same warnings persist after 3 attempts** → Report remaining issues to user with explanation. Some warnings may need user decision (e.g., intentionally unused ports in testbenches).
- **New warnings introduced by fixes** → Continue fixing (these are usually easier).

## Output Format

When complete, report:

```
✅ Verilog Lint Clean: <filename>

Summary:
  - Errors fixed: N
  - Warnings fixed: M
  - Remaining (suppressed): K
  - Iterations: X

Fixes applied:
  - Line 42: WIDTH mismatch — added explicit 8-bit cast
  - Line 15: UNUSED signal 'temp' — removed
  - Line 30: CASEINCOMPLETE — added default case
  ...
```

## Advanced Usage

### Lint specific warning types only:
```bash
verilator --lint-only -Wall -Wno-UNUSED -Wno-DECLFILENAME <file.v>
```

### Lint with include paths:
```bash
verilator --lint-only -Wall -I./includes -I./rtl <file.v>
```

### Lint entire project:
```bash
verilator --lint-only -Wall --top-module top_module file1.v file2.v file3.v
```

### Generate lint report only (no fix):
Just run Step 2 and Step 3, report findings without modifying files.

## Notes

- Always back up files before modifying (or ensure git is initialized so changes are revertable)
- For large codebases, lint one module at a time to keep fixes manageable
- Some Verilator warnings are style preferences (DECLFILENAME, UNUSED in testbenches) — use `lint_off` pragmas rather than restructuring code
- SystemVerilog (.sv) files may require `--sv` flag: `verilator --lint-only -Wall --sv <file.sv>`
