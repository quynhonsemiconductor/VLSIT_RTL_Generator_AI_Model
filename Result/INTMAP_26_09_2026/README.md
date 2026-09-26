# INTMAP run — 2026-09-26 (branch `qsoc`)

First QSoC block through this flow. Input: `spec/intmap_spec.md` (copy of
`QNSC_Interrupt_Map_MAS.md` V2.2). Output merged into QSoC as
`design/intmap/rtl/m_qnsc_intmap.sv` (PR quynhonsemiconductor/vlsi_deep_training#31).

| Phase | Result |
|---|---|
| `/spec_parser` | 20 requirements (14 functional, 2 interface, 4 constraint). One ambiguous (REQ-020, X on an input, score 0.5): resolved by the owner as a simulation-only assertion. Gate 1 approved |
| `/config_ui` | No parameters (MAS §8); Gate 2 self-approved |
| `/rtl_generator` | 1 module, REQ-001..020 tagged; Verilator `-Wall`: no warning from the module; QSoC `make check` clean |
| `/tb_generator`, `/sva_generator`, `/verification` | Not run: SIM is a later QSoC stage. A local smoke simulation walked every input bit and passed |

Findings for the flow (reported to the teacher):

1. `rtl_rule.md` conflicts with the QSoC Naming Rule V1.0; replaced on `qsoc`.
2. `rtl_generator.md` hardcodes the 18 RV32IM modules and the GF180 synthesis gate; made generic on `qsoc`.
3. `/config_ui` has an RV32IM parameter table as the fallback; not needed for a block without parameters.
4. Pre-flight assumes Linux Environment Modules (`module load`); plain Verilator is enough for lint.
5. A spec whose code snippet disagrees with a repository rule (literal line indices vs `qnsc_pkg` constants) needs the rule to win explicitly; recorded as REQ-015.
