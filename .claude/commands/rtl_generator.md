# Phase 3a · RTL Generator — /rtl_generator   (QSoC, branch `qsoc`)

**Goal:** generate the SystemVerilog modules of one QSoC block from the approved
requirements and configuration, tag REQ-IDs, lint each module as it is written, and
hand the result to the QSoC repository's own checks.

**Not in this command:** SVA (`/sva_generator`), testbench, simulation.

This replaces the upstream RV32IM version: the module list, spec sections and tools
are read from the artifacts, not hardcoded.

---

## Fixed rules

1. Write one module, lint it, and only move on when it is clean.
2. Implement only what the spec says. If the spec is ambiguous: stop and ask.
3. Follow `rtl_rule.md` (QSoC naming, `qnsc_pkg`, coding rules) throughout.
4. Shared numbers come from `qnsc_pkg` (`vlsi_deep_training/design/top/rtl/qnsc_pkg.sv`),
   never typed.
5. Gate 3 opens after `/sva_generator`, not here.

## Step 1 — Pre-flight (stop on the first failure)

1. `schemas/final_config.json`: `gate_2_approved` must be `true`; read `parameters`.
2. `schemas/structured_spec.json`: `gate_1_approved` must be `true`; read the
   requirements and their `rtl_modules` (the module list and what each implements).
3. The spec file named in `structured_spec.json` (`source_spec`): the section each
   requirement cites.
4. `rtl_rule.md`.
5. `QSOC=../vlsi_deep_training` exists and has `design/top/rtl/qnsc_pkg.sv`.
6. `verilator --version` works (Homebrew, apt or oss-cad-suite; no `module load`
   needed).

Print:
```
Pre-flight OK: Gate 1/2 approved · N requirements · M modules · Verilator X · qnsc_pkg found
```

## Step 2 — Generate each module, leaves first

For each module in `rtl_modules`, in dependency order (a module after the ones it
instantiates):

**2a.** Read the spec sections of the requirements mapped to it.

**2b.** Write `src/rtl/<module>.sv`:

```systemverilog
//==============================================================================
// Module      : <module>
// Description : <one line from the spec>
// Spec ref    : <QNSC_<BLOCK>_MAS.md> §<n>
// REQ-IDs     : REQ-xxx, REQ-yyy
//==============================================================================
module <module>
  import qnsc_pkg::*;
#(
  // P_* parameters from final_config.json, only those this module uses
) (
  // clock and reset (only if the module has state)
  // ---- <port group> ---- // REQ-xxx
);
  // localparam C_*, r_* then w_* declarations
  // assign / always_comb (defaults first) // REQ-xxx
  // always_ff                              // REQ-xxx
  // instances (named connections)
`ifndef SYNTHESIS
  // assertions: placeholders, filled by /sva_generator
`endif
endmodule
```

Tag each port group and each logic block with the REQ-IDs it implements.

**2c.** Lint it:
```bash
verilator --lint-only -Wall -Wno-fatal "$QSOC/design/top/rtl/qnsc_pkg.sv" \
  src/rtl/<module>.sv [modules it instantiates] --top-module <module>
```
Clean = no warning from `src/rtl/`. Fix, do not waive: `LATCH`, `MULTIDRIVEN`,
`WIDTH*`, `UNUSED*` (re-read the spec), `UNOPTFLAT`. `UNUSEDPARAM` inside `qnsc_pkg`
is expected (a block uses few contract constants) and is waived by the QSoC repo.

## Step 3 — Filelist and whole-block lint

`src/rtl/filelist.f`: `qnsc_pkg.sv` first, then the modules leaves first. Lint the
top module with all of them.

## Step 4 — QSoC repository gate (replaces the GF180 synthesis gate)

Copy the modules into `$QSOC/design/<block>/rtl/`, list them in
`$QSOC/design/<block>/<block>.f` (`../top/rtl/qnsc_pkg.sv` first; paths relative to
the filelist), and run on a branch there:

```bash
make -C "$QSOC" check
```

Pass = clean. Synthesis is a later sign-off stage in QSoC (`make syn BLOCK=<block>`,
Yosys + slang); run it only if the tools are installed, and record `not run` otherwise.

Write `schemas/synth_report.json` with `"status": "not_run"` or the Yosys result, and
`"qsoc_check": "pass"`.

## Step 5 — Report

```
╔══ RTL GENERATOR REPORT (QSoC) ═══════════════════════╗
║  Modules: M/M · lint clean                            ║
║  QSoC make check: pass                                ║
║  REQ-IDs tagged: N                                    ║
║  Files: src/rtl/*.sv, src/rtl/filelist.f              ║
║  Next: /sva_generator (Gate 3), or a PR in QSoC       ║
╚═══════════════════════════════════════════════════════╝
```

## Errors

| Situation | Action |
|---|---|
| Gate 1 or Gate 2 not approved | Stop; run `/spec_parser` or `/config_ui` |
| Spec unclear | Stop and ask; never guess |
| A shared number is needed but not in `qnsc_pkg` | Stop: it belongs in `util/qsoc_contract.yml` first (a QSoC PR) |
| `make check` fails in QSoC | Fix the generated RTL here, copy again, re-run |
| `src/rtl/<module>.sv` exists | Ask before overwriting |
