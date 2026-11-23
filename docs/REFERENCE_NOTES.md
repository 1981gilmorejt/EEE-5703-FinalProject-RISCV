# Track A Reference Notes

## Core contract

The DUT is a small single-cycle RV32I subset with 12 supported instructions:
`add`, `sub`, `and`, `or`, `xor`, `addi`, `andi`, `ori`, `lw`, `sw`, `beq`,
and `jal`. The specification defines everything outside this contract as a NOP.
Architectural state is 31 general-purpose registers plus `x0`, 64 data-memory
words, and the program counter. Registers and data memory are compared against
`lib/models/rv32i_model.py` by the verification environment.

## Verification plan checkpoints

- Plan: feature IDs, traceability, test matrix, coverage cross, and residual risks.
- Environment: driver, monitor, and reference-model scoreboard.
- Stimulus: distinguish legal contract constraints from team-selected shaping constraints;
  probe programs with the ISS before simulation.
- Coverage: cross opcode with destination class, and branch outcome with direction;
  defend every ignored or illegal bin.
- Mutation: score clean and released/hidden pools; classify survivors as reachability
  or observability gaps.
- Lint: run Verilator with `-Wall`, recording justified waivers only.
- Sign-off: verdict, lint/coverage/mutation/strategy/security dashboard, and residual risk.

## Relevant course techniques

- Directed corner tests from a specification matrix.
- Constrained-random stimulus and pre-simulation probing.
- Reference-model checking after each instruction or at architectural checkpoints.
- Functional coverage used to identify and close stimulus holes.
- Assertions bound to internal RTL without modifying the DUT.
- Formal bounded checks and induction where tractable.
- Mutation studies to measure where the suite detects injected faults.

## Track A security/correctness experiment

The track-level question is whether any encoding outside the 12-instruction
contract changes architectural state. A conforming NOP changes only `pc` by 4.
The experiment should distinguish unsupported opcodes and table-opcode encodings
with unsupported field combinations, and it should state that a finite sample
cannot establish the result for every 32-bit encoding.

## Local execution

The smoke test and mutation harness run locally. The guide's Eustis setup command
is not required here; activate the local virtual environment and use the supplied
Icarus/Verilator toolchain instead.
