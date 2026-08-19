# CrayFisher Agent Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement target profiling, evidence context, strict Recon/Defender/Judgment contracts, ordered primitive composition, and fail-closed pipeline orchestration.

**Architecture:** Deterministic adapters generate observations and bounded source context. Semantic agents may propose primitives or bridges only through strict schemas. Challenge records decide which primitives and bridges are supported; deterministic ordered composition consumes only those supported records.

**Tech Stack:** Python 3.12, Pydantic v2, httpx, Semgrep CLI adapter, Typer, Pytest

**Spec:** `docs/design/architecture.md`

## Global Constraints

- This plan starts after `2026-08-19-crayfisher-core.md` is complete.
- Semgrep results are candidates, never final evidence.
- Model responses are recorded verbatim after secret redaction.
- Invalid, missing, duplicate, or mismatched response IDs become `UNKNOWN`.
- Chain composition uses ordered tuples, not a graph database.

---

### Task 1: Target Profile and Observation Contracts

**Files:**
- Create: `src/crayfisher/profiling.py`
- Create: `src/crayfisher/observations.py`
- Create: `tests/fixtures/agent-python/`
- Create: `tests/fixtures/agent-typescript/`
- Create: `tests/unit/test_profiling.py`
- Create: `tests/unit/test_observations.py`

**Interfaces:**
- Produces: `profile_target(root: Path) -> TargetProfile`
- Produces: `ObservationAdapter.collect(profile: TargetProfile) -> tuple[Observation, ...]`
- Produces: `TargetKind = AGENT | NON_AGENT | HYBRID`

- [ ] **Step 1: Write failing profile tests**

```python
def test_agent_target_prioritizes_agent_flow() -> None:
    profile = profile_target(FIXTURES / "agent-python")
    assert profile.kind is TargetKind.AGENT
    assert profile.primary_stages[0] is Stage.AGENT_FLOW


def test_hybrid_preserves_non_agent_scope() -> None:
    profile = profile_target(FIXTURES / "agent-typescript")
    assert profile.kind is TargetKind.HYBRID
    assert profile.web_scope
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_profiling.py tests/unit/test_observations.py -q`

Expected: FAIL because profile and observation types are missing.

- [ ] **Step 3: Implement deterministic package/file/config inspection**

Inspect `pyproject.toml`, requirements files, `package.json`, imports, tool registration, workflow files, credential helpers, storage calls, LLM call sites, process/file/network sinks, and sandbox configuration. Preserve every matched location as `SourceRef`; do not assign a security verdict.

- [ ] **Step 4: Run focused tests**

Run: `uv run pytest tests/unit/test_profiling.py tests/unit/test_observations.py -q`

Expected: PASS for agent, non-agent, and hybrid fixtures.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/profiling.py src/crayfisher/observations.py tests/fixtures tests/unit/test_profiling.py tests/unit/test_observations.py
git commit -m "feat: profile agent security surfaces"
```

### Task 2: Function and Caller Evidence Context

**Files:**
- Create: `src/crayfisher/context.py`
- Create: `tests/unit/test_context.py`

**Interfaces:**
- Produces: `build_context(root: Path, refs: tuple[SourceRef, ...], budget: ContextBudget) -> EvidenceContext`
- Consumes: observations from Task 1

- [ ] **Step 1: Write failing enclosing-function and caller tests**

```python
def test_context_includes_enclosing_function_and_direct_caller() -> None:
    context = build_context(FIXTURE, (sink_ref,), ContextBudget(max_bytes=16_000))
    assert context.enclosing_functions
    assert context.direct_callers
```

- [ ] **Step 2: Verify the test fails**

Run: `uv run pytest tests/unit/test_context.py -q`

Expected: FAIL because `build_context` is missing.

- [ ] **Step 3: Implement bounded Python AST and TypeScript structural context**

Use Python `ast` for function and direct caller extraction. Normalize TypeScript context through Semgrep JSON metavariable spans and enclosing blocks. Emit explicit `MISSING_FUNCTION` and `MISSING_CALLER` records instead of silently dropping unavailable context.

- [ ] **Step 4: Run the context tests**

Run: `uv run pytest tests/unit/test_context.py -q`

Expected: PASS, including budget truncation with a recorded reason.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/context.py tests/unit/test_context.py
git commit -m "feat: build bounded evidence context"
```

### Task 3: Strict Semantic Transport and Recorded Replay

**Files:**
- Create: `src/crayfisher/semantic.py`
- Create: `prompts/recon.md`
- Create: `prompts/defender.md`
- Create: `prompts/judgment.md`
- Create: `tests/fixtures/semantic-responses/`
- Create: `tests/unit/test_semantic.py`

**Interfaces:**
- Produces: `SemanticTransport.complete(request: SemanticRequest) -> SemanticResponse`
- Produces: `RecordedTransport(responses: Mapping[str, Path])`
- Produces: `HttpTransport(client: httpx.Client, endpoint: HttpEndpoint)`

- [ ] **Step 1: Write failing response-routing tests**

```python
def test_mismatched_request_id_is_unknown() -> None:
    result = parse_response(request("req-1"), payload(request_id="req-2"))
    assert result.verdict is ReviewVerdict.UNKNOWN
    assert "request_id_mismatch" in result.reasons


def test_duplicate_claim_is_unknown() -> None:
    result = parse_response(request("req-1"), payload_with_two_claims())
    assert result.verdict is ReviewVerdict.UNKNOWN
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_semantic.py -q`

Expected: FAIL because semantic contracts do not exist.

- [ ] **Step 3: Implement discriminated request/response models**

Define request kinds `recon`, `challenge`, `judgment`, require one claim per requested entity, require evidence refs on supported claims, and preserve malformed payloads as invalid-response artifacts. Prompts describe the schema but tests assert only parsed fields and routing behavior.

- [ ] **Step 4: Run semantic tests**

Run: `uv run pytest tests/unit/test_semantic.py -q`

Expected: PASS for supported, rebutted, abstained, malformed, duplicate, and mismatched fixtures.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/semantic.py prompts tests/fixtures/semantic-responses tests/unit/test_semantic.py
git commit -m "feat: add fail-closed semantic contracts"
```

### Task 4: Primitive Discovery, Defender Challenge, and Judgment

**Files:**
- Create: `src/crayfisher/discovery.py`
- Create: `src/crayfisher/challenge.py`
- Create: `src/crayfisher/judgment.py`
- Create: `tests/unit/test_discovery.py`
- Create: `tests/unit/test_challenge.py`
- Create: `tests/unit/test_judgment.py`

**Interfaces:**
- Produces: `discover_primitives(input: DiscoveryInput, transport: SemanticTransport) -> DiscoveryResult`
- `DiscoveryResult` contains `primitives: tuple[Primitive, ...]` and `bridge_candidates: tuple[BridgeCandidate, ...]`
- Produces: `challenge_primitive(primitive: Primitive, context: EvidenceContext, transport: SemanticTransport) -> ReviewResult`
- Produces: `challenge_bridge(bridge: BridgeCandidate, context: EvidenceContext, transport: SemanticTransport) -> ReviewResult`
- Produces: `judge_primitive(primitive: Primitive, challenge: ReviewResult, transport: SemanticTransport) -> ReviewResult`
- Produces: `judge_bridge(bridge: BridgeCandidate, challenge: ReviewResult, transport: SemanticTransport) -> ReviewResult`

- [ ] **Step 1: Write failing candidate/evidence tests**

```python
def test_semgrep_only_result_is_candidate() -> None:
    discovery = discover_primitives(input_with_semgrep_only(), recorded_transport())
    result = judge_primitive(
        discovery.primitives[0],
        challenge_without_direct_evidence(),
        recorded_transport(),
    )
    assert result.verdict is ReviewVerdict.UNKNOWN


def test_guard_counterevidence_rebuts_primitive() -> None:
    result = challenge_primitive(primitive(), context_with_workspace_guard(), recorded_transport())
    assert result.verdict is ReviewVerdict.REBUTTED


def test_judgment_cannot_support_missing_criterion() -> None:
    result = judge_primitive(primitive(), challenge_missing_reachability(), recorded_transport())
    assert result.verdict is ReviewVerdict.UNKNOWN
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_discovery.py tests/unit/test_challenge.py tests/unit/test_judgment.py -q`

Expected: FAIL because discovery, challenge, and judgment functions are missing.

- [ ] **Step 3: Implement stage-specific requests and tri-state criteria**

Evaluate external reachability, default triggerability, attacker control, and meaningful impact independently as `pass`, `fail`, or `unknown`. The first `fail` rebuts the primitive; any security-criterion `unknown` prevents support. Record duplicate status separately as `known`, `not_known`, or `unknown`; it never changes the security verdict, but only `not_known` permits a zero-day claim. For a non-exact post/pre pair, create a semantic bridge candidate only when capability kinds match and caller/data-flow, persistent-key, runtime-reference, or tool-output-to-argument evidence relates the two source contexts. Challenge whether the producer's postcondition satisfies the consumer's precondition using direct evidence from both sides. Judgment receives both the original claim and Defender result; deterministic validation changes any missing security criterion, mismatched entity ID, or unsupported evidence reference to `UNKNOWN`.

- [ ] **Step 4: Run focused tests**

Run: `uv run pytest tests/unit/test_discovery.py tests/unit/test_challenge.py tests/unit/test_judgment.py -q`

Expected: PASS with exact reasons for every unknown or rebuttal.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/discovery.py src/crayfisher/challenge.py src/crayfisher/judgment.py tests/unit/test_discovery.py tests/unit/test_challenge.py tests/unit/test_judgment.py
git commit -m "feat: discover and adjudicate primitives"
```

### Task 5: Ordered Chain Composition

**Files:**
- Create: `src/crayfisher/composition.py`
- Create: `tests/unit/test_composition.py`

**Interfaces:**
- Produces: `compose(primitives: tuple[Primitive, ...], primitive_reviews: Mapping[str, ReviewResult], bridge_reviews: Mapping[tuple[str, str], ReviewResult]) -> tuple[ChainCandidate, ...]`
- Produces: `validate_chain(candidate: ChainCandidate, primitive_reviews: Mapping[str, ReviewResult], bridge_reviews: Mapping[tuple[str, str], ReviewResult]) -> ChainValidation`

- [ ] **Step 1: Write failing exact-match and missing-edge tests**

```python
def test_exact_postcondition_satisfies_next_precondition() -> None:
    chains = compose(
        (leak_id(), resolve_foreign_id()),
        supported_primitive_reviews(),
        {},
    )
    assert len(chains) == 1


def test_n_links_require_n_minus_one_bridge_evidence() -> None:
    validation = validate_chain(
        chain_with_three_links_and_one_bridge(),
        supported_primitive_reviews(),
        one_supported_bridge_review(),
    )
    assert validation.verdict is ReviewVerdict.UNKNOWN
    assert "missing_bridge_evidence" in validation.reasons
```

- [ ] **Step 2: Verify the tests fail**

Run: `uv run pytest tests/unit/test_composition.py -q`

Expected: FAIL because composition does not exist.

- [ ] **Step 3: Implement bounded ordered search**

Discard any primitive with a non-`SUPPORTED` review, sort the remainder by stable ID, and extend chains only when capabilities match exactly or a `SUPPORTED` semantic bridge with direct evidence exists. Prohibit repeated primitive IDs, cap v0.1 chains at six primitives, and preserve uncomposed primitives in artifacts.

- [ ] **Step 4: Run composition tests**

Run: `uv run pytest tests/unit/test_composition.py -q`

Expected: PASS for exact, semantic, cyclic, missing-evidence, and maximum-length cases.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/composition.py tests/unit/test_composition.py
git commit -m "feat: compose evidence-backed chains"
```

### Task 6: Pipeline Orchestration and Scan Command

**Files:**
- Create: `src/crayfisher/pipeline.py`
- Modify: `src/crayfisher/cli.py`
- Create: `tests/integration/test_pipeline.py`

**Interfaces:**
- Produces: `run_pipeline(request: ScanRequest, transport: SemanticTransport) -> RunSummary`
- Produces: CLI `crayfisher scan PATH --config RUN_CONFIG`

- [ ] **Step 1: Write the failing integration test**

```python
def test_pipeline_writes_every_stage_artifact(tmp_path: Path) -> None:
    summary = run_pipeline(scan_request(tmp_path), recorded_transport())
    assert summary.completed_stages == (
        "profile", "discover", "challenge", "compose"
    )
    assert summary.artifacts.profile.exists()
    assert summary.artifacts.chains.exists()
```

- [ ] **Step 2: Verify the test fails**

Run: `uv run pytest tests/integration/test_pipeline.py -q`

Expected: FAIL because the pipeline is missing.

- [ ] **Step 3: Implement explicit stage transitions**

Abort only on target materialization failure or invalid run configuration. Empty discovery writes `no-findings.json`; malformed model output writes `invalid-responses.jsonl`; all other incomplete states remain `UNKNOWN` records.

- [ ] **Step 4: Run all non-replay gates**

Run: `uv run ruff check . && uv run basedpyright && uv run pytest tests/unit tests/integration -q`

Expected: all commands exit 0.

- [ ] **Step 5: Commit**

```bash
git add src/crayfisher/pipeline.py src/crayfisher/cli.py tests/integration/test_pipeline.py
git commit -m "feat: orchestrate CrayFisher analysis"
```
