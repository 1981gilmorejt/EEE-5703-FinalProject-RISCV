# Verification plan — Final Project Team 9 — RV32I Track A

**Author:** Justin Gilmore

**Specification:** `capstone/riscv/SPEC.md` (no version identifier is stated)

**Date:** November 23, 2026

> This plan defines what evidence would support sign-off, how that evidence
> will be collected, and which gaps remain. Passing tests or a coverage
> percentage alone do not establish correctness.

## 1. Scope and verification approach

**DUT:** `capstone/riscv/rv32i_core.sv`, a single-cycle core implementing the
12-instruction subset listed in the specification. The instruction/data
memories and top-level wrapper are the fixed simulation harness.

**In scope**

- Correct architectural behavior for `add`, `sub`, `and`, `or`, `xor`, `addi`,
  `andi`, `ori`, `lw`, `sw`, `beq`, and `jal`.
- Register writeback, the `x0` invariant, PC/control-flow behavior, synchronous
  active-low reset of the PC, and data-memory addressing.
- Unsupported encodings acting as NOPs: PC advances by four and architectural
  registers and data memory are unchanged.
- Track A correctness/security experiment: test representative unsupported
  opcodes and unsupported field combinations for unintended architectural
  changes.

**Out of scope**

- Byte/halfword operations and other RV32I instructions as functional
  instructions.
- Unaligned word accesses; the specification limits the contract to aligned
  word accesses.
- Exhaustive testing of all 2^32 instruction words, physical timing, synthesis,
  and behavior outside the stated single-cycle simulation contract.

**Environment and assumptions**

- Use the supplied cocotb testbench, Icarus for local simulation, and
  `lib/models/rv32i_model.py` as the instruction-set reference model (ISS).
- `program.hex` is loaded into instruction memory at simulation startup. The
  current harness does not expose a run-time instruction-memory write port.
  Any generated program must therefore be supplied before simulation starts,
  or use a separately reviewed loading mechanism.
- The instruction image is limited to the harness's 64 words. Data-memory
  indices follow `(effective_address >> 2) mod 64`.
- The register file is not reset. Tests initialize registers through executed
  instructions before relying on their values; they do not assume an initial
  value for an unwritten register. Tests store a value before relying on a
  data-memory read.
- Expected state is advanced using the ISS once per executed instruction.
  Comparisons are made after the active clock edge has settled, using the
  register and data-memory debug read ports. PC behavior is checked through
  subsequent instruction effects and control-flow markers; the harness does
  not expose a dedicated architectural PC debug port.

**Testbench architecture**

- **Stimulus/program generation:** directed programs first; then seeded,
  constrained-random programs over legal instruction forms and bounded
  execution.
- **Driver:** controls clock/reset and starts each preloaded program.
- **Monitor:** samples architectural state at the defined post-edge checkpoint.
- **Scoreboard:** compares the DUT state with the independently expressed ISS
  model and counts comparisons and mismatches.
- **Coverage collector:** samples executed/observed behavior, not merely
  requested generator values. Coverage and scoreboard checking are separate
  responsibilities.

## 2. Feature list

Each feature maps to the Track A contract in `SPEC.md`. Instruction names use
the standard RV32I encodings implemented by the supplied assembler. Any
encoding that is not one of the specified operations is treated as unsupported.

| ID | Required feature | Specification reference | Priority | Verification |
|----|------------------|-------------------------|----------|--------------|
| F1 | `add`: write `rs1 + rs2` to `rd` | Supported instruction subset | H | ISS scoreboard; directed operand cases |
| F2 | `sub`: write `rs1 - rs2` to `rd` | Supported instruction subset | H | ISS scoreboard; directed operand cases |
| F3 | `and`: write `rs1 & rs2` to `rd` | Supported instruction subset | H | ISS scoreboard; directed operand cases |
| F4 | `or`: write `rs1 \| rs2` to `rd` | Supported instruction subset | H | ISS scoreboard; directed operand cases |
| F5 | `xor`: write `rs1 ^ rs2` to `rd` | Supported instruction subset | H | ISS scoreboard; directed operand cases |
| F6 | `addi`: add sign-extended immediate to `rs1` | Supported instruction subset; immediates | H | ISS scoreboard; positive/negative and boundary immediates |
| F7 | `andi`: AND `rs1` with sign-extended immediate | Supported instruction subset; immediates | H | ISS scoreboard; directed bit-pattern cases |
| F8 | `ori`: OR `rs1` with sign-extended immediate | Supported instruction subset; immediates | H | ISS scoreboard; directed bit-pattern cases |
| F9 | `lw`: load a word from the effective address | Supported instruction subset; memory map | H | ISS scoreboard; address and data checks |
| F10 | `sw`: store a word at the effective address | Supported instruction subset; memory map | H | ISS scoreboard; memory checks |
| F11 | `beq`: branch on equality; otherwise advance by four | Supported instruction subset | H | ISS scoreboard; taken/not-taken and target cases |
| F12 | `jal`: jump and write `pc + 4` to `rd` | Supported instruction subset | H | ISS scoreboard; link and target checks |
| F13 | Any unsupported instruction encoding is a NOP | Supported instruction subset | H | Representative encoding classes; state-preservation checks |
| F14 | `x0` reads as zero and ignores writes | Register model | H | Directed writes to `x0`; readback checks |
| F15 | Register writes and non-writeback behavior occur at the specified edge | Register model | H | Edge-aware ISS comparison; sentinel-register checks |
| F16 | Effective addresses and memory indexing follow the stated mapping | Memory map; `lw`/`sw` semantics | H | Directed offsets, indices, and alias cases |
| F17 | While reset is asserted, the next active edge sets PC to zero | Reset | H | Directed reset/restart program |

## 3. Coverage model

Coverage is derived from the contract and sampled from executed behavior. A bin
is included only when it represents a meaningful verification question.
Coverage does not replace ISS comparisons.

| ID | Coverpoint | Planned bins | Why these bins matter |
|----|------------|---------------|-----------------------|
| C1 | Executed instruction | All 12 supported instructions; unsupported encoding classes tracked separately | Shows which contract operations and NOP cases actually executed |
| C2 | Destination class | `x0`, nonzero register, for instructions that write a destination | Finds write suppression and destination decode errors |
| C3 | Arithmetic/logical result class | Zero, nonzero, and selected sign/bit-boundary results | Exercises result selection and boundary-sensitive behavior without treating every data value as a distinct scenario |
| C4 | Immediate class by format | Zero, positive, negative, and selected representable boundary values | Targets sign extension and immediate reconstruction |
| C5 | BEQ condition and displacement | Equal/not-equal operands; zero/forward/backward displacement and target address bit 1 set/clear, with targets mapping into the loaded image | Distinguishes branch decision from target direction, fall-through, and instruction-memory word indexing |
| C6 | Load/store address class | Word indices 0, interior, and 63; a different full address mapping to an already-used index | Exercises address calculation and modulo aliasing |
| C7 | Reset event | Asserted at start; asserted after execution has begun | Covers reset/restart timing that the smoke test does not stress |

**Required crosses**

| ID | Cross | Targeted failure interaction |
|----|-------|-----------------------------|
| X1 | Writeback instruction × destination class | Incorrect write suppression or unintended write to `x0` |
| X2 | BEQ outcome × displacement direction | Incorrect taken/not-taken selection or target/fall-through calculation |
| X3 | Load/store operation × address class | Different address-generation or memory-indexing defects in loads and stores |

**Applicability and exclusions**

- Destination-class coverage applies only to instructions that write `rd`;
  `sw` and `beq` are not counted as destination writes.
- Immediate bins are sampled only for instruction formats that contain an
  immediate. Min/max values must be representable by that format and satisfy
  the word-alignment contract where applicable.
- No unsupported encoding is excluded from testing as “illegal stimulus”:
  the specification requires unsupported encodings to behave as NOPs.
- No cross is removed from the denominator merely because it is difficult to
  reach. Any structurally inapplicable category is excluded by defining the
  coverpoint's applicable population, with that decision documented here.

## 4. Test matrix

All tests use the ISS for expected architectural state and must assert that the
expected number of comparisons actually occurred. `test_final.py` and its
associated Makefile integration are planned work; only `test_smoke.py` is
currently present.

| Test | Features / coverage | Kind | Pass criterion |
|------|---------------------|------|----------------|
| `test_smoke` (existing) | F1–F12; intended C1 contribution (not yet instrumented) | Directed bring-up | All expected register and data-memory checks match the ISS; program image matches the test program |
| `test_arithmetic_edges` (planned) | F1–F8, F14–F15; C1–C4; X1 | Directed | Arithmetic/logical outputs match the ISS for zero, bit-boundary, signed-immediate, and `rd=x0` cases |
| `test_memory_addresses` (planned) | F9–F10, F14–F16; C1, C2, C4, C6; X1, X3 | Directed | Loads/stores use the sign-extended effective address and specified word index; stores and subsequent loads agree with ISS |
| `test_branch_paths` (planned) | F11, F15; C1, C5; X2 | Directed | Taken and not-taken branches with zero, forward, and backward displacements reach the expected marker; skipped instructions have no side effects |
| `test_jal_link_and_target` (planned) | F12, F14–F15; C1, C2, C4 | Directed | Target instruction executes, skipped instruction does not, and link value equals the jump instruction's `pc + 4`, including `rd=x0` |
| `test_unsupported_nop` (planned) | F13, F15; C1 | Directed encoding classes | Selected unsupported opcodes and unsupported field combinations advance to the next instruction without register or memory changes |
| `test_reset_restart` (planned) | F17; C7 | Directed | Assert reset for one active edge while a side-effect-free instruction is at the current PC, deassert before the next edge, and verify execution restarts from address zero |
| `test_random_programs` (planned) | F1–F16; C1–C6; X1–X3 | Seeded constrained-random | Legal generated programs are probed with the ISS before simulation; every executed checkpoint matches; seed, coverage, comparisons, and mismatches are reported |
| Assertion checks (planned) | F12, F14–F16 | Assertions, if supported by the selected simulator | Each property passes and has a non-vacuous activation/cover result |

Random stimulus will be constrained to legal encodings, initialized operands,
the 64-word program limit, aligned word accesses, and bounded execution. Directed
tests remain responsible for required corners that random sampling may miss.
The seed is logged so failures can be reproduced. Any reset injection must also
reset or otherwise correctly update the reference model.

## 5. Ambiguities, risks, and decisions

| ID | Ambiguity or risk | Source / evidence | Decision for this plan | Residual risk / status |
|----|-------------------|------------------|------------------------|------------------------|
| R1 | “Everything else” is a broad description of unsupported encodings; the spec does not list every invalid opcode/funct-field combination. | Supported instruction subset; RTL decode inspected | Treat only canonical encodings of the 12 listed instructions as supported. Test representative unsupported major opcodes and unsupported field combinations, requiring NOP behavior. | The 2^32 encoding space is not exhaustively tested. RTL inspection shows some opcode cases use default ALU behavior without fully rejecting unsupported function fields; the planned tests may expose a contract violation that must be resolved before sign-off. |
| R2 | Side effects while reset is asserted are not specified beyond the PC reset behavior. | Reset section; RTL has separate register/memory write logic not gated by `rst_n` | Require PC to reset on the active edge. Assert reset for one edge while a side-effect-free instruction is at the current PC, then deassert before the next edge. | Behavior of instruction side effects while reset is low remains unspecified and unverified. |
| R3 | Register-file initial values are undefined by reset; data-memory initial state is not part of the core reset contract. | Reset section; fixed harness implementation | Initialize every register and memory location the test relies on through program behavior; do not infer reset behavior from initial simulator values. | Unwritten registers and memory locations are not assigned a functional initial-state guarantee. |
| R4 | Instruction memory is loaded from `program.hex` at time zero; no run-time load interface is exposed. | `rv32i_imem.sv`, `rv32i_top.sv` | Directed programs use preloaded images. Decide and document a deterministic pre-simulation image-generation flow before claiming constrained-random program coverage. | Random stimulus is not fully implementable through the current public harness interface alone; avoid hierarchical memory writes unless explicitly approved and documented. |
| R5 | The harness has architectural register and data-memory read ports, but no dedicated PC debug port. | `rv32i_top.sv`, observability hooks | Verify PC/control flow with marker instructions, links, skipped side effects, and ISS execution order rather than adding hierarchical peeks. | Direct per-cycle PC comparison is not part of this plan. |
| R6 | The provided assembler and ISS reside in the same Python model package. A shared encoding/modeling error could make generated programs and expected values agree incorrectly. | `lib/models/rv32i_model.py` | Probe programs in the ISS before simulation and retain directed known-answer cases, including hand-reviewed instruction words and architectural outcomes. | Common-mode errors in the supplied model/assembler cannot be ruled out completely. |
| R7 | BEQ/JAL can produce a PC with bit 1 set, while instruction memory indexes by `address >> 2`, discarding that bit. | Memory map; `rv32i_imem.sv` | Follow the specified word-index mapping and include a branch/JAL target with bit 1 set that maps into the loaded image. | This verifies the stated simulator contract, not behavior of a full RV32I implementation with compressed instructions. |

The phrase “and the rest of RV32I” is interpreted together with the explicit
12-instruction table and the statement that everything else is a NOP. The
remaining gap is not which behavior is intended, but the impracticality of
testing every unsupported 32-bit encoding.

## 6. Closure criteria and sign-off evidence

Sign-off is a documented engineering judgment, not a claim that the core is
bug-free.

- **Clean baseline:** all enabled tests pass on clean RTL. The current smoke
  test has passed locally with Icarus; this is bring-up evidence, not closure.
- **Checking evidence:** zero mismatches; report the number of ISS comparisons
  and assert that it is nonzero and equal to the expected checkpoint count.
- **Functional coverage:** hit every required bin and cross, or classify each
  uncovered item as inapplicable with justification, unreachable with evidence,
  or accepted residual risk. Report numerator, denominator, and any model
  changes; do not use a percentage alone.
- **Mutation:** preserve the observed released-pool result of 8/8 found as a
  baseline. Run the final suite against the course hidden pool; report the
  exact Gradescope found/total/threshold result and investigate every survivor
  as a reachability, observability, or checker gap.
- **Assertions:** report which planned properties were implemented and
  activated. Do not count an assertion as evidence if its antecedent never
  occurred. If the selected tool cannot run a property, report it as not run.
- **Lint:** run Verilator with `-Wall`; record tool output and justify any
  waivers rather than silently suppressing warnings.
- **Reproducibility:** record simulator, command, random seed(s), test results,
  comparison count, coverage report, and mutation results.
- **Risk closure:** every row in Section 5 has a disposition and residual-risk
  statement in the final report.

**Not verified by this plan:** every possible unsupported instruction word;
byte/halfword and unaligned accesses; undefined initial contents; behavior of
register or memory side effects while reset is asserted; physical timing,
metastability, and synthesis behavior; and any assertion/formal property not
actually run and reported.

**Disclosure:** GitHub Copilot assisted in drafting this verification plan. Reviewed by Justin Gilmore.