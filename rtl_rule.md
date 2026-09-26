# RTL Coding Rule — QSoC (branch `qsoc`)

| Item | Value |
|---|---|
| Language | SystemVerilog-2012, synthesizable subset |
| Applies to | every RTL file this flow generates for QSoC |
| Naming source | `QNSC_RTL_Design_Naming_Rule` V1.0 — `vlsi_deep_training/doc/rules/QNSC_RTL_Design_Naming_Rule.pdf` |
| Repository rules | `vlsi_deep_training/design/README.md` ("Naming", "Decided where the rule and the demos leave a choice") |
| Checked by | `vlsi_deep_training` CI: Verilator `-Wall`, `flow/lint/naming_check.py`, `flow/lint/hardcode_check.py` |

This file replaces the RV32IM rule of the upstream flow on branch `qsoc`. **When it
disagrees with the Naming Rule PDF or the repository rules, those win.** Generated RTL
must pass the repository's CI unchanged.

---

## 1. Naming (Naming Rule V1.0)

| Object | Form | Example |
|---|---|---|
| Module, in house | `m_qnsc_<function>` | `m_qnsc_intmap` |
| Module, wrapper around an IP | `m_qnsc_wrap_<ip_module>` | `m_qnsc_wrap_apb_adv_timer` |
| Module, shared cell | `qnsc_<function>` | `qnsc_sync` |
| Port | `i_` / `o_` / `io_` prefix | `i_int_dma`, `o_int_fast` |
| Clock, reset | `i_clk_<domain>`, `i_rst_n_<domain>` (active low: `_n` right after the meaning) | `i_clk_peri`, `i_rst_n_peri` |
| APB, AXI | `i_bus_apb_<sig>`; `i_bus_axi_<ch>_<sig>` | `i_bus_apb_paddr`, `o_bus_axi_ar_valid` |
| Interrupt | `i_int_<source>`, `o_int_<source>` | `i_int_uart_0` |
| Pad | `i_pad_*`, `o_pad_*`, `io_pad_*`; JTAG keeps `i_jtag_*`, debug `i_dbg_*` | `o_pad_pwm` |
| Memory, DMA, debug, boot, DFT | `i_mem_*`, `i_dma_*`, `i_dbg_*`, `i_boot_*`, `i_dft_*` | `o_mem_addr` |
| Registered signal | `r_<function>` | `r_state` |
| Combinational signal | `w_<function>` | `w_state_nxt` |
| Parameter, constant, FSM state | `P_<NAME>`, `C_<NAME>`, `S_<NAME>` (uppercase) | `P_WIDTH`, `S_IDLE` |
| Instance | `u_<function>[_<index>]` | `u_sync_tim_ext` |
| Memory array | `mem_<function>` | `mem_data` |
| Index | underscore before the digit | `timer_0`, never `timer0` |
| Vocabulary | `int` not `irq`/`interrupt`, `clk` not `clock`, `rst` not `reset`; `cfg`, `dbg`, `pwr`, `mem`, `peri`, `mux` | `o_int_fast` |

Everything is `lower_snake_case` except `P_`/`C_`/`S_` names.

## 2. Shared numbers come from `qnsc_pkg`

- `import qnsc_pkg::*;` in the module header when the module uses a shared number.
- Addresses, interrupt lines, line counts and widths shared between blocks are
  **never typed**: use `C_*` from `qnsc_pkg` (`C_PWM_BASE`, `C_INT_LINE_DMA`,
  `C_INT_FAST_LINES_USED`, `C_APB_PADDR_WIDTH`). The package is generated from
  `vlsi_deep_training/util/qsoc_contract.yml`; read the values there, never copy them.
- A number used by one block only is a `P_*` parameter or a `localparam C_*`.

## 3. Module structure

1. Header comment: module, one-line description, spec reference
   (`QNSC_<BLOCK>_MAS.md` §), REQ-IDs.
2. `module <name>`, `import qnsc_pkg::*;`, `#( parameters )`, ports.
3. Port groups, each preceded by a comment: clock/reset, bus, pads, interrupts, others.
4. `localparam`, signals (`r_*` then `w_*`), `assign` / `always_comb`, `always_ff`,
   instances, assertions inside `` `ifndef SYNTHESIS ``.

## 4. Coding rules

| # | Rule |
|---|---|
| R1 | Sequential: `always_ff @(posedge i_clk_<d> or negedge i_rst_n_<d>)`; asynchronous assert, synchronous deassert (the reset synchroniser is in `SCRC`) |
| R2 | Combinational: `always_comb` or `assign`; never `always @(*)` |
| R3 | `<=` in `always_ff`, `=` in `always_comb`; never mixed |
| R4 | Every `always_comb` assigns every output a default first (no latch) |
| R5 | One driver per signal |
| R6 | Every `case` has a `default`; `unique case` for a fully enumerated type |
| R7 | `logic` only; no `reg`, `wire`, `integer` in RTL |
| R8 | Named port and parameter connections only; no positional, no `.*` |
| R9 | Widths explicit; cast rather than rely on padding |
| R10 | A module with state has `i_clk_*` and `i_rst_n_*`; a purely combinational module has neither |
| R11 | A signal from another clock domain goes through `qnsc_sync` (`design/common`) or a handshake its MAS specifies |
| R12 | No clock gating in RTL, except a vendored clock-gate cell a wrapper must supply |

Forbidden in RTL: `#delay`, `initial`, `casex`/`casez`, `defparam`, implicit nets,
intentional latches, multi-driven nets, `$random`/`$display`, `inout` inside the core,
typed shared numbers.

## 5. Where generated RTL goes, and the lint gate

The flow writes to `src/rtl/` in this repository. The gate is the QSoC repository's own
checks, run on a branch there:

```bash
QSOC=../vlsi_deep_training
cp src/rtl/<module>.sv $QSOC/design/<block>/rtl/
# list it in $QSOC/design/<block>/<block>.f: ../top/rtl/qnsc_pkg.sv first, then rtl/<module>.sv
make -C $QSOC check
```

Pass = `make check` clean: Verilator `-Wall` shows no warning from the new file, and the
naming and hardcode checks are clean. The pull request in `vlsi_deep_training` is then
reviewed like any hand-written RTL.
