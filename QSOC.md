# Using this flow for QSoC (branch `qsoc`)

Fork of `nguyenquanicd/VLSIT_RTL_Generator_AI_Model`, adapted for the QSoC training chip
(`quynhonsemiconductor/vlsi_deep_training`). `main` mirrors the teacher's repository;
`qsoc` carries these changes.

| Changed | Why |
|---|---|
| `rtl_rule.md` | The RV32IM rule conflicted with the mandatory `QNSC_RTL_Design_Naming_Rule` V1.0 (`i_resetn_` vs `i_rst_n_`, `reg_` vs `r_`, `PR_`/`LP_` vs `P_`/`C_`, `irq` vs `int`); replaced by the QSoC rule |
| `.claude/commands/rtl_generator.md` | The upstream version hardcoded the 18 RV32IM modules and the GF180 synthesis gate; it now reads the module list from `structured_spec.json` and hands the RTL to the QSoC repository's `make check` |

**Scope:** blocks designed in house (INTMAP, SYSDBG, SCRC, SYSCSR). Wrappers around
vendored IP are written with emacs verilog-mode in the QSoC repository, not here.

**Layout:** clone this repository next to `vlsi_deep_training` (`../vlsi_deep_training`).
The input spec is a copy of the block's MAS in `spec/`; the outputs are in `schemas/`
and `src/rtl/`; the RTL enters QSoC through a normal pull request.

**Environment:** Verilator from Homebrew or apt is enough for lint; `sourceme.sh`
(Environment Modules, GF180) is not needed. Synthesis and SVA simulation are later
QSoC stages.

**Keep in sync:** `git switch main && git pull upstream main && git push origin main`,
then `git switch qsoc && git merge main`.
