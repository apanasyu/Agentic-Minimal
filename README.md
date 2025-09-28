# Agentic‑Minimal

*A bloat‑resistant, test‑first template for building agentic workflows in Python with VS Code, Cursor/Codex/Windsurf, and lightweight formal methods.*

<p align="center">
  <a href="#"><img alt="license" src="https://img.shields.io/badge/license-MIT-blue.svg"></a>
  <a href="#"><img alt="python" src="https://img.shields.io/badge/python-3.11%2B-3776AB"></a>
  <a href="#"><img alt="tests" src="https://img.shields.io/badge/tests-pytest%20%2B%20hypothesis-brightgreen"></a>
  <a href="#"><img alt="lint" src="https://img.shields.io/badge/lint-ruff-informational"></a>
  <a href="#"><img alt="types" src="https://img.shields.io/badge/types-mypy-inactive"></a>
</p>

---

## Why this exists
Complex projects collapse when too many low‑level agents/code paths accumulate without a clear architecture. *Agentic‑Minimal* enforces a **Planner → Decomposer → Implementer → Tester → Fixer → Evaluator** loop with strict contracts, automated tests, and quality gates. It mirrors how effective human orgs scale: a capable architect sets the plan; specialized workers execute; supervisors verify and correct.

**Design goals**
- Minimal moving parts; no agent chatter.
- Contracts everywhere (Pydantic I/O models + invariants).
- Tests first (unit, property‑based, metamorphic, sanity checks).
- Quality gates (lint, type, coverage) before merge.
- Optional lightweight formal methods (TLA+) for stage invariants.

---

## What we’re building
A reusable **starter kit** and methodology you can drop into any repo to:
1. **Draft a plan** (architect prompt) based on a declarative `workflow.yaml`.
2. **Decompose** into small, independent tasks with clear DoD.
3. **Generate tests first**.
4. **Implement the minimum** to satisfy tests and contracts.
5. **Auto‑evaluate** with gates (lint/type/coverage/sanity/metamorphic).
6. **Iterate** via a small Fixer role that applies minimal diffs.

The included example is a tiny **CSV→JSON ETL**; swap it for your target (e.g., a mini‑RAG). The point is the *discipline*, not the domain.

---

## Architecture (high‑level)
- **Planner (Architect)**: produces `.artifacts/plan.md` from `workflow.yaml`.
- **Decomposer (Systems)**: outputs `.artifacts/decomposition.md` with task DAG and stubs.
- **Tester**: generates tests/fixtures first.
- **Implementer (Coder)**: writes minimal code (unified diffs only).
- **Fixer (Debugger)**: smallest changes to pass all gates.
- **Evaluator (Judge)**: JSON report of contracts/tests/coverage/lint/type.

Contracts: `src/core/contracts.py` define I/O schemas and invariants. Optional `tla/` adds stage invariants.

---

## Repository layout
```
agentic-minimal/
├─ README.md
├─ pyproject.toml
├─ Makefile
├─ workflow.yaml
├─ prompts/
│  ├─ 00_planner.md
│  ├─ 01_decomposer.md
│  ├─ 02_implementer.md
│  ├─ 03_tester.md
│  ├─ 04_fixer.md
│  └─ 05_evaluator.md
├─ src/
│  ├─ app.py
│  ├─ core/
│  │  ├─ contracts.py
│  │  ├─ states.py
│  │  └─ evaluation.py
│  └─ tasks/
│     └─ etl_csv_json.py
├─ tests/
│  ├─ test_contracts.py
│  ├─ test_etl_csv_json.py
│  ├─ test_sanity.py
│  └─ test_metamorphic.py
├─ .vscode/
│  ├─ settings.json
│  ├─ tasks.json
│  └─ launch.json
├─ .pre-commit-config.yaml
└─ tla/
   ├─ pipeline.tla
   └─ pipeline.cfg
```

---

## Getting started
```bash
# 1) Clone and enter
git clone https://github.com/<you>/agentic-minimal.git
cd agentic-minimal

# 2) Install (poetry or pip)
pipx run poetry install || pip install -r requirements.txt

# 3) Pre-commit hooks (optional)
pre-commit install || true

# 4) Run the full check suite
make check   # lint + types + tests + evaluation
```

Open in **VS Code** and use the **Tasks** panel for: *Plan → Decompose → Implement → Test → Eval*.

---

## Stable prompts (Cursor/Codex/Windsurf)
Use these role prompts to guarantee consistent outputs:

- **Architect** → `.artifacts/plan.md`
  > Read `workflow.yaml` and `prompts/00_planner.md`. Produce `.artifacts/plan.md` only (600–900 words) with sections: Overview, Modules, Contracts, Risks, Test Strategy, Quality Gates, Milestones.

- **Decomposer** → `.artifacts/decomposition.md`
  > Read `.artifacts/plan.md` and `prompts/01_decomposer.md`. Emit a single checklist, a task DAG, concrete stubs (files/functions), and definition‑of‑done/acceptance tests.

- **Tester‑First** → diffs under `tests/`
  > Using `prompts/03_tester.md` and the decomposition, generate unit/property/metamorphic/golden tests. Do **not** modify `src/`.

- **Implementer** → minimal diffs under `src/`
  > Apply `prompts/02_implementer.md` and implement only the listed stubs with type hints and docstrings. No new modules.

- **Fixer** → smallest diffs to pass
  > Given failing tests or lint/type errors, apply `prompts/04_fixer.md` and return the minimal patch to pass all gates.

- **Evaluator** → `.artifacts/eval.json`
  > Run `prompts/05_evaluator.md` and produce a JSON gate report `{"contracts", "tests", "coverage", "lint", "mypy", "gates": {"overall"}}`.

---

## Quality gates (default)
- **Ruff** clean (style + lint)
- **Mypy** clean (strict-ish typing)
- **Coverage ≥ 80%** via `pytest --cov`
- **Contracts/invariants** hold (Pydantic + sanity tests)
- **Metamorphic** properties pass (e.g., column permutation invariance)

---

## Testing strategy
- **Unit tests** for every public function.
- **Property‑based tests** with Hypothesis for core transforms.
- **Metamorphic tests** (e.g., invariance under whitespace/column order).
- **Sanity checks** for end‑to‑end smoke runs and round‑trip validation.

Run:
```bash
make test           # quick test run
pytest -q           # directly
pytest -q --cov=src --cov-report=term-missing
```

---

## Formal methods (optional, lightweight)
Use `tla/pipeline.tla` and `pipeline.cfg` to model stage transitions and invariants (`InvNonNeg`, `InvProgress`). This is intentionally minimal: it catches category errors early without heavy ceremony. Run with TLC/Apalache if installed.

---

## CI
Add a simple GitHub Actions workflow at `.github/workflows/ci.yml`:
```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install -r requirements.txt || pipx run poetry install
      - run: make check
```

---

## Roadmap
- Pluggable **skills**: importable, verified worker agents (parse JSON, call REST APIs, etc.).
- **Workflow DSL**: stronger type‑checked DAGs with per‑node contracts.
- **Self‑healing supervisor**: auto‑replan on gate failure.
- **Coverage‑guided prompting**: generate tests for uncovered paths.

---

## Contributing
PRs welcome. Keep changes minimal and test‑backed. New features must add contracts, tests, and pass all gates. Follow `ruff` and `mypy` configurations.

---

## License
MIT
