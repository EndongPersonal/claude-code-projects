# Verilog Lint Fixer

A Claude Code skill that iteratively detects and fixes Verilog/SystemVerilog lint issues using Verilator.

## Quick Start

1. Install Verilator: `winget install Verilator`
2. Tell Claude Code: "fix lint in <file.v>"
3. It will loop: check → fix → re-check until clean

## What It Fixes

| Category | Examples |
|---|---|
| Errors | Syntax errors, semantic errors |
| Width mismatches | `WIDTH` — bit width inconsistencies |
| Unused signals | `UNUSED`, `UNDRIVEN` |
| Case issues | `CASEINCOMPLETE`, `CASEOVERLAP` |
| Assignment style | `BLKSEQ` — blocking vs non-blocking |
| Naming | `DECLFILENAME`, `VARHIDDEN` |
| Timescale | `TIMESCALEMOD` — missing timescale |
| Optimizations | `UNOPTFLAT`, `IFDEPTH`, `SYNCASYNCNET` |

## Usage Examples

```
# Fix lint in a single file
"fix lint in rtl/alu.v"

# Fix lint in all verilog files in a directory
"clean up verilog in ./rtl/"

# Check without fixing
"check lint in top.sv" 

# Suppress specific warnings
"fix lint in fifo.v, ignore UNUSED and DECLFILENAME"
```

## Requirements

- **Verilator** (primary) — `winget install Verilator`
- **iverilog** (fallback) — `winget install IcarusVerilog`
- Git recommended (for safe rollback)
