# INTMAP — first QSoC block through the flow (2026-09-26, branch `qsoc`)

Claude Code was opened in this repository and ran the slash commands. The owner
answered every question and signed every gate; the answers were typed back verbatim.
`logs/` holds each turn in order.

| Step | Command / answer | Outcome |
|---|---|---|
| 1 | `/spec_parser spec/intmap_spec.md` (copy of `QNSC_Interrupt_Map_MAS.md` V2.2) | 21 requirements (14 functional, 1 timing, 3 interface, 3 constraint). REQ-020 decided without a question: the line indices come from `qnsc_pkg` because `rtl_rule.md` §2 overrides the literal indices in the spec's snippet. Two questions asked |
| 2 | `017: b, 021: a` | REQ-017 (connection to the core in `design/top`) becomes an integration note; REQ-021 becomes a simulation-only X check |
| 3 | `none`, then `yes` | Gate 1 signed; `schemas/structured_spec.json` written |
| 4 | `/config_ui`, `none`, `yes` | No parameters (MAS §8); the RV32IM constraints C1–C5 recorded as N/A; 14 style rules from `rtl_rule.md`; Gate 2 signed; `schemas/final_config.json` |
| 5 | `/rtl_generator` | `src/rtl/m_qnsc_intmap.sv` and `filelist.f`; Verilator `-Wall`, naming and hardcode clean. It stopped before touching the QSoC repository and asked |
| 6 | `yes` (QSoC branch prepared by the owner) | RTL copied to `vlsi_deep_training/design/intmap/rtl/`, `intmap.f` updated, `make check` pass; synthesis `not_run` (no Yosys here); nothing committed |

Result in QSoC: pull request "feat(intmap): add the interrupt map RTL, generated with
the teacher's AI flow". The RTL is used exactly as generated.

## Findings

1. **RV32IM hardcoding.** Upstream `rtl_rule.md` conflicts with the mandatory QNSC
   Naming Rule V1.0 (`i_resetn_`/`reg_`/`PR_`/`LP_`/`irq` vs `i_rst_n_`/`r_`/`P_`/`C_`/`int`),
   and `rtl_generator.md` lists the 18 RV32IM modules and a GF180 synthesis gate. Both
   were adapted on `qsoc` before the run. `sva_generator.md` (68 mentions of RV32IM),
   `tb_generator.md` (14) and `verification.md` (16) need the same before they can run
   on another IP.
2. `config_ui.md` falls back to the RV32IM parameter table and constraints C1–C5; it
   handled a block without parameters by marking them N/A.
3. Pre-flight assumes Linux Environment Modules (`module load oss-cad-suite`) and the
   GF180 PDK. Plain Verilator was enough for lint.
4. No notion of a wrapper around vendored IP: the flow generates logic from a spec, so
   it fits blocks designed in house.
5. Suggestion: after Gate 1, emit the list of decisions that change the spec (resolved
   ambiguities, closed open items), so the engineer folds them into the spec and the
   next run does not ask again.
6. The run could not validate the JSON against the schemas (`jsonschema` was not
   permitted) and wrote `approved_at` as a date only (`date` not permitted): run it
   with those commands allowed.
7. Run as separate commands, the flow ran no simulation before the TB phase; the QSoC
   owner ran a smoke test (every input bit reaches only its own line: pass).
