# CrayFisher Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the strict Python package, immutable domain model, executable specification manifest, and deterministic run artifacts for CrayFisher v1.

**Architecture:** Parse all untrusted JSON at Pydantic boundaries, convert it to immutable domain records, and store every stage result under a content-derived run key. Prompt formalization is represented by a traceability manifest; inactive rules remain visible but cannot execute.

**Tech Stack:** Python 3.12, uv, Pydantic v2, Typer, Rich, Ruff, Basedpyright, Pytest

**Spec:** `docs/design/architecture.md`

## Global Constraints

- Python version is exactly `>=3.12,<3.13` for v0.1.
- Every source file remains at or below 250 pure LOC.
- Public signatures contain no `Any`, `cast`, `# type: ignore`, or mutable default.
- All domain records are immutable.
- Every active specification rule has one acceptance test.
- No LLM prose is asserted in tests.

---

### Task 1: Strict Package and CLI Skeleton

**Files:**
- Create: `pyproject.toml`
- Create: `src/crayfisher/__init__.py`
- Create: `src/crayfisher/cli.py`
- Create: `tests/unit/test_cli.py`
- Create: `.gitignore`

**Interfaces:**
- Produces: console command `crayfisher`, `crayfisher.cli.app: typer.Typer`

- [ ] **Step 1: Write the failing CLI test**

```python
from typer.testing import CliRunner

from crayfisher.cli import app


def test_help_lists_scan_and_spec_commands() -> None:
    result = CliRunner().invoke(app, ["--help"])
    assert result.exit_code == 0
    assert "scan" in result.stdout
    assert "spec" in result.stdout
```

- [ ] **Step 2: Verify the test fails**

Run: `uv run pytest tests/unit/test_cli.py -q`

Expected: FAIL because the package and `app` do not exist.

- [ ] **Step 3: Add the minimal package and strict tool configuration**

Create `pyproject.toml` with the following project boundary. Resolve these ranges once with `uv lock`; subsequent CI uses the committed lockfile.

```toml
[project]
name = "crayfisher"
version = "0.1.0"
requires-python = ">=3.12,<3.13"
dependencies = [
  "httpx>=0.28,<1",
  "pydantic>=2.12,<3",
  "rich>=14,<15",
  "typer>=0.21,<1",
]

[project.scripts]
crayfisher = "crayfisher.cli:app"

[dependency-groups]
dev = [
  "basedpyright>=1.36,<2",
  "pytest>=9,<10",
  "ruff>=0.14,<1",
]

[build-system]
requires = ["hatchling>=1.28,<2"]
build-backend = "hatchling.build"

[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.basedpyright]
typeCheckingMode = "strict"
include = ["src", "tests"]
```

Create `src/crayfisher/cli.py` with named command boundaries from the first commit:

```python
from pathlib import Path

import typer

app = typer.Typer(no_args_is_help=True)


@app.command()
def scan(path: Path = typer.Argument(..., exists=True, file_okay=False)) -> None:
    """Inspect PATH with the configured CrayFisher pipeline."""
    typer.echo(path)


@app.command()
def spec() -> None:
    """Inspect or validate the executable specification."""
    typer.echo("specification support is installed")
```

- [ ] **Step 4: Run package gates**

Run: `uv run ruff check . && uv run basedpyright && uv run pytest tests/unit/test_cli.py -q`

Expected: all commands exit 0.

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml src/crayfisher tests/unit/test_cli.py .gitignore
git commit -m "build: bootstrap strict CrayFisher package"
```

### Task 2: Immutable Evidence and Chain Domain

**Files:**
- Create: `src/crayfisher/model.py`
- Create: `src/crayfisher/ids.py`
- Create: `tests/unit/test_model.py`

**Interfaces:**
- Produces: `SourceRef`, `Capability`, `Primitive`, `BridgeCandidate`, `Bridge`, `Chain`, `ReviewVerdict`, `WitnessStatus`
- Produces: `primitive_id(primitive: Primitive) -> str`, `chain_id(primitive_ids: tuple[str, ...]) -> str`

- [ ] **Step 1: Write tests for invalid ranges, immutability, and stable IDs**

```python
def test_source_ref_rejects_reversed_lines() -> None:
    with pytest.raises(ValueError, match="start_line"):
        SourceRef(Path("a.py"), 4, 3, "0" * 64)


def test_primitive_id_is_order_independent_for_evidence() -> None:
    assert primitive_id(example_primitive((ref_a, ref_b))) == primitive_id(
        example_primitive((ref_b, ref_a))
    )
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_model.py -q`

Expected: FAIL because domain types are missing.

- [ ] **Step 3: Implement frozen dataclasses and canonical SHA-256 IDs**

Use `StrEnum`, `NewType` only where it prevents accidental ID mixing, and `json.dumps(..., sort_keys=True, separators=(",", ":"))` for canonical hashes. Validate `1 <= start_line <= end_line` and a 64-character lowercase hex digest.

- [ ] **Step 4: Verify model tests and types**

Run: `uv run pytest tests/unit/test_model.py -q && uv run basedpyright src/crayfisher/model.py src/crayfisher/ids.py`

Expected: PASS and zero type errors.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/model.py src/crayfisher/ids.py tests/unit/test_model.py
git commit -m "feat: define evidence-backed chain domain"
```

### Task 3: Formalization Manifest and Specification Loader

**Files:**
- Create: `specifications/manifest.json`
- Create: `specifications/pipeline.json`
- Create: `specifications/agent-flows.json`
- Create: `specifications/policies/agent-authorization.json`
- Create: `specifications/policies/tool-result-injection.json`
- Create: `specifications/policies/sandbox-escape.json`
- Create: `specifications/policies/exploit-chain.json`
- Create: `src/crayfisher/specification.py`
- Create: `tests/unit/test_specification.py`

**Interfaces:**
- Produces: `SpecificationSet.load(root: Path) -> SpecificationSet`
- Produces: `SpecificationSet.active_rules(stage: Stage) -> tuple[RuleSpec, ...]`
- Consumes: manifest schema defined in `docs/methodology/prompt-formalization.md`

- [ ] **Step 1: Write failing traceability tests**

```python
def test_every_active_rule_has_existing_implementation_and_test(specs: SpecificationSet) -> None:
    for rule in specs.rules:
        if rule.active:
            assert rule.implementation.exists()
            assert rule.acceptance_test.startswith("tests/")


def test_a7_is_candidate_rule(specs: SpecificationSet) -> None:
    rule = specs.by_id("agent-flow-a7")
    assert rule.execution_class is ExecutionClass.CANDIDATE


def test_manifest_preserves_every_formalized_rule(specs: SpecificationSet) -> None:
    assert len(specs.rules) == 545
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_specification.py -q`

Expected: FAIL because no manifest loader exists.

- [ ] **Step 3: Implement Pydantic boundary models and immutable specification set**

Reject duplicate rule IDs, duplicate source spans, unknown execution classes, missing files, and active rules without acceptance tests. Populate A1-A10 from the formulas in `docs/methodology/prompt-formalization.md`; preserve all 545 traced rules in the manifest and keep rules without an implementation fixture inactive.

- [ ] **Step 4: Run the specification tests**

Run: `uv run pytest tests/unit/test_specification.py -q`

Expected: PASS with all 545 traced rules present, A1-A10 active as candidate rules, the four initial policies active, and inactive rules excluded from dispatch.

- [ ] **Step 5: Commit**

```bash
git add specifications src/crayfisher/specification.py tests/unit/test_specification.py
git commit -m "feat: compile prompt formalization into specifications"
```

### Task 4: Deterministic Run Key and Artifact Store

**Files:**
- Create: `src/crayfisher/run.py`
- Create: `src/crayfisher/artifacts.py`
- Create: `tests/unit/test_run.py`
- Create: `tests/unit/test_artifacts.py`

**Interfaces:**
- Produces: `RunConfig`, `RunManifest`, `run_key(manifest: RunManifest) -> str`
- Produces: `ArtifactStore.create(root: Path, manifest: RunManifest) -> ArtifactStore`
- Produces: `ArtifactStore.append_jsonl(name: ArtifactName, record: BaseModel) -> None`

- [ ] **Step 1: Write failing determinism and append tests**

```python
def test_run_key_changes_when_spec_version_changes() -> None:
    assert run_key(manifest(spec="v1")) != run_key(manifest(spec="v2"))


def test_artifact_store_rejects_unknown_artifact_name(store: ArtifactStore) -> None:
    with pytest.raises(ValueError, match="artifact"):
        store.append_jsonl("misc", example_observation())
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_run.py tests/unit/test_artifacts.py -q`

Expected: FAIL because the run and artifact modules do not exist.

- [ ] **Step 3: Implement canonical run hashing and atomic artifact writes**

Derive the key from target commit, specification version, model identity, prompt version, enabled policy IDs, and normalized run settings. Write JSON through a temporary file and `Path.replace`; JSONL appends use a per-run lock file.

- [ ] **Step 4: Run core gates**

Run: `uv run ruff check . && uv run basedpyright && uv run pytest tests/unit -q`

Expected: all commands exit 0 in under 30 seconds.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/run.py src/crayfisher/artifacts.py tests/unit/test_run.py tests/unit/test_artifacts.py
git commit -m "feat: add reproducible run artifacts"
```
