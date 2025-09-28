# Prompt Playbook: From README → Working Pipeline

Use this copy-paste playbook in Cursor/Codex/Windsurf after cloning the repo. It turns a brand-new problem into a working, test-first pipeline with minimal bloat.

---

## Phase 0 — New Problem Brief (single source of truth)

Create `problem_brief.md` and fill it in:

```markdown
# <PROJECT_TITLE>
## Goal
<One paragraph of the user-visible outcome.>
## Inputs (raw)
<Data sources, examples, formats, sizes, constraints.>
## Outputs (desired)
<Artifacts, APIs, schemas, reports, SLAs.>
## Non-Goals
<What we explicitly won’t do.>
## Constraints
<Latency, cost, privacy, model limits, allowed deps.>
## Risks/Unknowns
<Bullets of biggest uncertainties.>
## Benchmarks / Success Criteria
<Objective metrics & acceptance tests in plain language.>
```

---

## Phase 1 — Architect (Planner)

**Prompt:**

```text
SYSTEM: You are the Architect for this repo. Read `problem_brief.md` and `workflow.yaml`.
OUTPUT ONLY a single markdown file `.artifacts/plan.md` (600–900 words) with EXACT sections:
1) Overview, 2) Modules, 3) Data Contracts, 4) Risks & Mitigations,
5) Test Strategy (unit, property-based, metamorphic, sanity), 6) Quality Gates,
7) Milestones (M0..M3) with acceptance criteria per milestone.

Rules:
- Reference concrete Pydantic model names for all I/O.
- Choose the smallest viable module set.
- Include a rollback plan if gates fail.
- Do not modify any code. No extra commentary.
```

---

## Phase 2 — Systems (Decomposer)

**Prompt:**

```text
SYSTEM: You are the Systems Decomposer. Read `.artifacts/plan.md`.
Emit ONLY `.artifacts/decomposition.md` with:
- Task DAG (ID, depends_on).
- For each task: definition-of-done, CLI entry point, files & functions to create (paths/signatures).
- Exact test list the Tester will write (filenames).
- Minimal fixtures to create (paths & sample rows).

Constraints:
- Prefer 3–7 tasks max.
- No new dependencies without an explicit justification line: "NewDep: <name> — reason".
- Do not write code. No chit-chat.
```

---

## Phase 3 — Tester-First

**Prompt:**

```text
SYSTEM: You are the Test Engineer. Read `.artifacts/decomposition.md`.
Output a unified diff that ONLY adds tests and fixtures under `tests/` and `tests/fixtures/`.

Required:
- Unit tests for every public function named in the decomposition.
- Hypothesis property-based tests for core transforms.
- Metamorphic tests as listed in `workflow.yaml` (e.g., permutation invariance, whitespace tolerance).
- Sanity e2e test on a tiny fixture; assert schema compliance.
- No edits to `src/`.

Return only a clean patch (git diff). No prose.
```

_Run:_

```bash
make test
```

(Expect failures until implementation exists.)

---

## Phase 4 — Implementer (Minimal patches)

**Prompt:**

```text
SYSTEM: You are the Implementer. Read `.artifacts/decomposition.md` and the failing test logs.
Output a unified diff touching ONLY the files/functions listed by the Decomposer.

Rules:
- Follow `src/core/contracts.py` models; add fields only if Plan requires.
- Small, pure functions where possible; clear docstrings & type hints.
- No new modules, no extra files, no side effects beyond I/O described.
- If a dependency is unavoidable, add a single line "NewDep:" and include minimal `pyproject.toml` changes in the patch.

Return only a clean patch. No prose.
```

_Run:_

```bash
make test
make lint
make type
```

---

## Phase 5 — Fixer (Minimal delta to green)

**Prompt:**

```text
SYSTEM: You are the Fixer. Read failing test/type/lint output.
1) Bullet root causes (max 5).
2) Return a minimal patch that fixes ALL current failures without adding new features.
No refactors beyond what is necessary. No prose outside the two sections.
```

_Then:_

```bash
make check   # lint + mypy + tests + evaluator
```

---

## Phase 6 — Evaluator (Objective gate report)

**Prompt:**

```text
SYSTEM: You are the Evaluator. Execute the checklist in `prompts/05_evaluator.md`.
Emit ONLY `.artifacts/eval.json` with:
{"contracts": [...], "tests": {"passed": N, "failed": M}, "coverage": <float>,
 "lint": <bool>, "mypy": <bool>, "gates": {"overall": <bool>, "reasons": [..]}}
```

---

## Guardrail Prompts (use when the model drifts)

**Stay within files**

```text
Do not create or modify any file not listed in `.artifacts/decomposition.md`.
If you believe a new file is necessary, STOP and list it under "Proposed additions" with a one-line justification.
```

**Unified diff only**

```text
Return a single unified diff that applies cleanly with `git apply`. No explanations, no code blocks, no Markdown fences.
```

**No new deps**

```text
Do not add dependencies. If strictly necessary, add one line "NewDep: <name> — reason" and include minimal `pyproject.toml` changes in the patch.
```

**Test-first discipline**

```text
If tests are missing for any public function in the plan, STOP and generate tests first (patch to `tests/` only).
```

---

## Quick Switch: New Problem → Update `workflow.yaml`

Update:

- `project.name`, `project.goal`
- `contracts.input_schema` / `output_schema` (Pydantic models)
- `sanity_checks` commands (tiny e2e)
- `metamorphic_tests` (domain-meaningful invariances)
- `quality_gates.coverage_min` (raise as repo matures)

_Run:_

```bash
make plan
make decompose
```

---

## Example Specializations

### A) Mini-RAG (fast, robust)

- **Contracts**: `Document`, `Chunk`, `Query`, `RetrievedDoc`, `RerankedDoc`, `Report`.
- **Metamorphic**: query paraphrase invariance; retrieval stability under small noise; top-k rerank monotonicity.
- **Sanity**: e2e on 10 docs, latency < 1s, deterministic report header.

### B) JSON ETL with Schema Evolution

- **Contracts**: `LegacyRow` → `NormalizedRecord`.
- **Metamorphic**: column permutation; missing optional fields; numeric precision bounds.
- **Sanity**: round-trip and idempotency.

---

## Sprint Loop (repeatable micro-cycle)

1. Refine plan for the next Milestone in `.artifacts/plan.md`.  
2. Decompose new tasks.  
3. Tester-first adds tests.  
4. Implementer minimal patch.  
5. Fixer to green.  
6. Evaluator writes gate report.  
7. Merge only if `gates.overall == true`.

---

## One-liner incantations (for your LLM IDE)

- **Architect**  
  `Architect role: read problem_brief.md + workflow.yaml; emit .artifacts/plan.md exactly as spec.`

- **Decomposer**  
  `Systems role: read plan; emit .artifacts/decomposition.md with DAG, stubs, tests; no code.`

- **Tester**  
  `Tester role: emit patch adding tests/fixtures only; property + metamorphic + sanity; no src edits.`

- **Implementer**  
  `Implementer role: emit minimal patch for listed files; follow contracts; no new modules.`

- **Fixer**  
  `Fixer role: list root causes then emit smallest patch to pass gates.`

- **Evaluator**  
  `Evaluator role: emit .artifacts/eval.json with gate verdicts only.`

---

## When things go sideways (triage prompts)

**Too much code / bloat**

```text
Reduce to the minimum set of modules to satisfy acceptance criteria. Remove any module not referenced by tests or contracts. Return a patch that deletes code + updates tests if needed.
```

**Ambiguous specs**

```text
List the top 5 ambiguities blocking implementation, each with a default resolution. Apply defaults and proceed to produce the minimal diff.
```

**Flaky tests**

```text
Stabilize tests by isolating non-determinism (random seeds, time). Convert flaky assertions into property-based predicates with tolerances.
```
