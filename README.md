# EEE 4701 / EEE 5703 Final Project -- RV32I Core Track

Local verification workspace for the Track A RV32I core. This repository is a
starting scaffold; it does not contain a completed final-project assignment.

## Local setup

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
cd capstone/riscv
export PYTHONPATH="../../lib"
make -s SIM=icarus
```

From the repository root, the same smoke test can run with:

```bash
export PYTHONPATH="$PWD/lib"
make -s -C capstone/riscv SIM=icarus
```

## Contents

- `capstone/riscv/`: Track A DUT, specification, smoke test, released mutant pool,
  and cocotb Makefile.
- `lib/models/rv32i_model.py`: golden RV32I instruction-set simulator.
- `templates/`: fill-in-the-blank deliverable starters.
- `docs/REFERENCE_NOTES.md`: concise Track A and verification-strategy reference.

## Deliverables intentionally not completed

Team-specific verification plan, coverage model, random environment, findings,
strategies, lint waiver record, and sign-off report are left for the team to build.
