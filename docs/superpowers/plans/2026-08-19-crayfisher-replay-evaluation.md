# CrayFisher Replay and Evaluation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add isolated runtime witnesses, security reports, evaluation variants, reproducibility documentation, and CI evidence for CrayFisher v1.

**Architecture:** A replay adapter executes a declared target fixture and returns typed observations; it cannot change static review verdicts. Reporting counts only witnessed chains as confirmed findings, while evaluation preserves separate denominators for discovery, composition, support, and replay.

**Tech Stack:** Python 3.12, uv, Pydantic v2, Pytest, GitHub Actions, optional Docker fixture adapter

**Spec:** `docs/design/architecture.md`

## Global Constraints

- This plan starts after the core and agent-pipeline plans are complete.
- Replay uses synthetic credentials and disposable directories or containers.
- Tests never contact a production endpoint.
- Detect, compose, support, and replay success remain separate metrics.
- Disclosure status is not inferred from a local finding.

---

### Task 1: Replay Protocol and Positive/Negative/Positive Controls

**Files:**
- Create: `src/crayfisher/replay.py`
- Create: `tests/fixtures/cross-stage-agent/`
- Create: `tests/unit/test_replay.py`
- Create: `tests/e2e/test_cross_stage_fixture.py`

**Interfaces:**
- Produces: `ReplayAdapter.run(case: ReplayCase) -> ReplayObservation`
- Produces: `confirm_chain(chain: Chain, plan: ReplayPlan, adapter: ReplayAdapter) -> ConfirmationResult`

- [ ] **Step 1: Write failing control tests**

```python
def test_positive_negative_positive_is_witnessed() -> None:
    result = confirm_chain(chain(), replay_plan(), fixture_adapter())
    assert result.status is WitnessStatus.WITNESSED
    assert result.positive_terminal is True
    assert result.negative_terminal is False
    assert result.restored_terminal is True


def test_required_link_drop_breaks_terminal_effect() -> None:
    result = confirm_chain(chain(), plan_with_link_drops(), fixture_adapter())
    assert all(not control.terminal_effect for control in result.link_drops)
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_replay.py tests/e2e/test_cross_stage_fixture.py -q`

Expected: FAIL because replay types and fixture are missing.

- [ ] **Step 3: Implement the safe fixture and replay adapter**

Port the already verified Arm A mechanism from the previous workspace as the first fixture: `save_note(name, content)` writes through a path argument, the unsafe path reaches a disposable startup file, and a later synthetic login lifecycle produces a sentinel effect. Port the Arm B hardened behavior as the negative control. Every run uses `tmp_path`; it never writes a real user startup file or uses a host credential.

- [ ] **Step 4: Run replay tests**

Run: `uv run pytest tests/unit/test_replay.py tests/e2e/test_cross_stage_fixture.py -q`

Expected: PASS without external network access.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/replay.py tests/fixtures/cross-stage-agent tests/unit/test_replay.py tests/e2e/test_cross_stage_fixture.py
git commit -m "feat: witness cross-stage chains"
```

### Task 2: Report and CVSS Boundary

**Files:**
- Create: `src/crayfisher/reporting.py`
- Create: `docs/config/artifacts.md`
- Create: `tests/unit/test_reporting.py`

**Interfaces:**
- Produces: `build_report(run: CompletedRun) -> SecurityReport`
- Produces: `write_report(report: SecurityReport, store: ArtifactStore) -> None`

- [ ] **Step 1: Write failing reporting tests**

```python
def test_unwitnessed_chain_is_not_confirmed() -> None:
    report = build_report(run_with_supported_unwitnessed_chain())
    assert report.confirmed_findings == ()
    assert len(report.unwitnessed_candidates) == 1


def test_cvss_requires_witness_and_rationale() -> None:
    with pytest.raises(ValueError, match="witness"):
        build_report(run_with_unwitnessed_cvss())
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_reporting.py -q`

Expected: FAIL because reporting does not exist.

- [ ] **Step 3: Implement JSON and Markdown projections**

Report confirmed, unwitnessed, unknown, and rebutted records separately. Validate CVSS 3.1 vector syntax and require per-metric rationale; score `0.0` maps to severity `None`. Document every artifact field in `docs/config/artifacts.md`.

- [ ] **Step 4: Run reporting tests**

Run: `uv run pytest tests/unit/test_reporting.py -q`

Expected: PASS for witnessed, unwitnessed, malformed CVSS, and no-finding runs.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/reporting.py docs/config/artifacts.md tests/unit/test_reporting.py
git commit -m "feat: report witnessed vulnerability chains"
```

### Task 3: Evaluation Registry, Baselines, and Ablations

**Files:**
- Create: `evaluation/registry.json`
- Create: `evaluation/variants.json`
- Create: `src/crayfisher/evaluation.py`
- Create: `docs/evaluation/protocol.md`
- Create: `tests/unit/test_evaluation.py`

**Interfaces:**
- Produces: `run_evaluation(registry: EvaluationRegistry, variant: Variant) -> EvaluationResult`
- Produces: variants `full`, `no_chain`, `no_defender`, `prompt_only`

- [ ] **Step 1: Write failing denominator and deduplication tests**

```python
def test_metrics_keep_stage_denominators_separate() -> None:
    result = summarize(example_trials())
    assert result.discovery.denominator != result.replay.denominator


def test_findings_are_deduplicated_by_root_cause() -> None:
    result = summarize(two_variants_same_root_cause())
    assert result.unique_root_causes == 1
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_evaluation.py -q`

Expected: FAIL because evaluation types are missing.

- [ ] **Step 3: Implement frozen target registry and metrics**

Registry entries require repository URL, commit SHA, target type, ground-truth source, disclosure state, allowed execution mode, and replay availability. Protocol documents inclusion/exclusion, baseline commands, repeated-run count, model/spec versions, token/cost accounting, root-cause deduplication, and confidence intervals.

- [ ] **Step 4: Run evaluation tests**

Run: `uv run pytest tests/unit/test_evaluation.py -q`

Expected: PASS for empty, partial, repeated, duplicate, and failed-run inputs.

- [ ] **Step 5: Commit**

```bash
git add evaluation src/crayfisher/evaluation.py docs/evaluation/protocol.md tests/unit/test_evaluation.py
git commit -m "feat: define CrayFisher evaluation protocol"
```

### Task 4: Quick Start, Development Guide, Reproducibility, and CI

**Files:**
- Create: `README.md`
- Create: `docs/getting-started.md`
- Create: `docs/config/run-config.md`
- Create: `docs/development-guide.md`
- Create: `docs/reproducibility.md`
- Create: `.github/workflows/ci.yml`
- Modify: `docs/README.md`

**Interfaces:**
- Produces: documented commands `uv sync --frozen`, `uv run crayfisher spec check`, `uv run crayfisher scan`, `uv run pytest`

- [ ] **Step 1: Add a failing documentation command smoke test**

Create `tests/e2e/test_documented_commands.py` that invokes the exact local fixture commands from `docs/getting-started.md` and asserts the run summary schema, not prose.

- [ ] **Step 2: Verify the test fails**

Run: `uv run pytest tests/e2e/test_documented_commands.py -q`

Expected: FAIL until commands and documents agree.

- [ ] **Step 3: Write canonical user and contributor documents**

Quick Start contains one install command, one spec validation command, one fixture scan, one artifact query, and exact expected artifact names. Development Guide explains how to add an observation adapter, active specification rule, semantic fixture, replay fixture, and acceptance test. Reproducibility pins target SHA, dependency lockfile, model identity, prompt/spec version, seed, environment and exact CI commands.

- [ ] **Step 4: Add CI and run the complete gate**

CI uses SHA-pinned actions, Python 3.12, `uv sync --frozen`, Ruff, Basedpyright, unit, integration, and local-only E2E tests. On failure it uploads the disposable run artifact directory.

Run: `uv run ruff check . && uv run ruff format --check . && uv run basedpyright && uv run pytest -q`

Expected: all commands exit 0.

- [ ] **Step 5: Commit**

```bash
git add README.md docs .github/workflows/ci.yml tests/e2e/test_documented_commands.py
git commit -m "docs: make CrayFisher reproducible"
```
