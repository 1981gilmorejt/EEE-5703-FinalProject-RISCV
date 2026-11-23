# Verification plan — ALU

**Author:** Justin Gilmore
**Spec version:** 1
**Date:** September 19, 2026

> A vplan is an argument that "done" is definable before you write tests. Every
> row here should be something you could defend in a signoff meeting. Rows you
> cannot defend are the honest content of the Risk section, not omissions.

## 1. Scope

**In scope:** The default WIDTH=8 ALU, all eight defined operations, outputs,
flags, combinational timing, and specified corner cases.
**Out of scope:** Undefined opcodes
**Assumptions about the environment:** `a` and `b` are WIDTH-bit inputs (WIDTH=8), `op`
is one of `3'd0` through `3'd7`, and tests sample outputs after one timestep.
The ALU has no clock, reset, or retained state.

## 2. Feature list

One row per behavior the spec requires. `Req` cites a line or section of the
spec — if you cannot cite one, that is a spec gap and it belongs in section 5.

| ID | Feature | Req | Priority | How it is verified |
|----|---------|-----|----------|--------------------|
| F1 | **Functional:** ADD produces the modulo-$2^{WIDTH}$ result | operations, flags, corner cases | H | directed |
| F2 | **Functional:** SUB produces the modulo-$2^{WIDTH}$ result | operations, flags, corner cases | H | directed |
| F3 | **Protocol:** the DUT returns `result`, `zero`, `carry`, and `overflow` for each operation | ports, flags | H | directed |
| F4 | **Ambiguous:** the 3-bit `op` field has only eight defined encodings | parameters, operations | L | directed |

## 3. Coverage model

Coverpoints and their bins. A cross earns its place only if the interaction is
where a bug would hide — say which bug.

| ID | Coverpoint | Bins | Why these bins |
|----|-----------|------|----------------|
| C1 | F1 ADD result and flag boundary | `ADD` | An ADD bin is enough to exercise the simplified feature: modulo result generation and carry/overflow behavior for the ADD operation. |
| C2 | F2 SUB result and flag boundary | `SUB` | A SUB bin is enough to exercise the simplified feature: modulo result generation and carry/overflow behavior for the SUB operation. |
| C3 | F3 output port behavior | `valid outputs` | One bin captures the protocol requirement that `result`, `zero`, `carry`, and `overflow` are produced as valid outputs for each operation. |
| C4 | F4 opcode encoding coverage | `defined opcodes` | One bin captures the parameter/operations requirement that the 3-bit `op` field remains within the defined 8-opcode space. |

**Crosses**

| ID | Cross | The interaction it is aimed at |
|----|-------|-------------------------------|
| X1 |       |  |

**Illegal / ignore bins:** <what is excluded and on what authority in the spec>

## 4. Test matrix
<!-- What tests will allow you to determine whether the features work or not -->
Every feature maps to at least one test; every test traces back to a feature.
An untraceable test is a test nobody can defend.

| Test | Features | Kind | Pass criterion |
|------|----------|------|----------------|
| `test_c1_add_result_and_flags` | F1 / C1 | directed | ADD gives the expected answer and status flags, including boundary cases. |
| `test_c2_sub_result_and_flags` | F2 / C2 | directed | SUB gives the expected answer and status flags, including boundary cases. |
| `test_c3_output_ports` | F3 / C3 | directed | The ALU reports the expected answer and flags for several operations. |
| `test_c4_opcode_space_coverage` | F4 / C4 | directed | All eight operation choices produce the expected outputs. |

## 5. Risk and spec ambiguity

| # | Ambiguity or risk | Where in the spec | How I resolved it | Residual |
|---|-------------------|-------------------|-------------------|----------|
| 1 |                   |                   | asked / assumed / tested both | |

## 6. Closure criteria

State the numbers *before* you start, so closure is a measurement and not a
negotiation.

- Functional coverage target: **75%** of the model in section 3
- Mutation target: **__/__** on the hidden pool
- Assertions: every property paired with a passing `cover` (no vacuous passes)
- What I will report as **not verified**:
	- Undefined opcode behavior.
	- Parameter values other than the default `WIDTH=8`.

**Disclosure:** GitHub Copilot assisted with constructing two parts of the coverage model, two corresponding features, and two corresponding tests.