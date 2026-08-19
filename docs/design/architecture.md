# CrayFisher v1 Architecture

## Goal

CrayFisher v1은 LLM agent 구현의 code, configuration, persistent state와 runtime effect에 흩어진 vulnerability primitive를 연결하고, 각 link와 bridge를 반증한 뒤 격리 replay로 확인하는 자동 분석 시스템이다.

## Non-Goals

- generic web pentesting platform을 만들지 않는다.
- OSS-CRS runtime, libCRS, compose, container orchestration을 복제하지 않는다.
- evidence-ledger graph나 quality-diverse search를 v1 핵심으로 사용하지 않는다.
- LLM의 prose 판단을 security verdict로 직접 신뢰하지 않는다.
- replay 이전에 zero-day 또는 confirmed vulnerability를 주장하지 않는다.

## Formal Unit

저장소 snapshot을 `r`, primitive를 `p`, evidence를 `ev`라 한다.

```text
p_i = (Pre_i, Op_i, Post_i, Ev_i)
C   = <p_1, p_2, ..., p_n>
```

- `Pre`: primitive가 실행되기 위해 필요한 attacker capability와 program state
- `Op`: 권한·값·state를 변환하는 code/config/runtime operation
- `Post`: primitive 실행 후 공격자가 획득하는 capability 또는 effect
- `Ev`: source citation, configuration citation 또는 runtime observation

인접 primitive는 다음 조건을 만족해야 한다.

```text
Composable(p_i, p_(i+1)) iff Post_i entails Pre_(i+1)
```

exact capability token match는 deterministic code가 계산한다. exact match가 아니면 semantic bridge candidate가 되고, direct evidence와 Defender challenge가 필요하다.

```text
BridgeCandidate(p_i, p_j) =
  i != j
  and exists(post in Post_i, pre in Pre_j): kind(post) = kind(pre)
  and observed_relation(Ev_i, Ev_j)
```

`observed_relation`은 caller/data-flow, 같은 persistent key, runtime reference resolution 또는 tool-output-to-argument observation 중 하나다. 이 gate를 만족하지 않는 primitive pair를 LLM에 보내지 않으므로 arbitrary all-pairs prompt search는 하지 않는다.

```text
Supported(C) =
  all(link_i.status = SUPPORTED)
  and all(bridge_i.status = SUPPORTED)
  and count(link evidence) = N
  and count(bridge evidence) = N - 1

Confirmed(C) =
  Supported(C)
  and Replay(C).terminal_effect = true
  and all(required link-drop controls remove terminal effect)
```

## Verdict Model

Primitive와 bridge semantic review는 다음 세 verdict만 가진다.

```text
SUPPORTED | REBUTTED | UNKNOWN
```

Chain의 runtime confirmation은 별도 상태다.

```text
UNWITNESSED | WITNESSED
```

`SUPPORTED`와 `WITNESSED`를 합치지 않는다. 정적·semantic evidence가 충분해도 runtime effect가 관찰되지 않으면 confirmed finding이 아니다.

## Lifecycle

```text
profile
  -> discover
  -> challenge
  -> compose
  -> replay
  -> report
```

### Profile

target commit, languages, agent frameworks, entry points, tool registrations, credential stores, persistence, sandbox와 external-provider sites를 기록한다.

### Discover

deterministic adapter가 observations를 만들고 Recon semantic contract가 observations와 source context를 vulnerability primitive 후보로 정규화한다. Semgrep match는 candidate일 뿐 evidence가 아니다.

Discover는 두 번의 bounded pass로 동작한다. 첫 pass가 primitive 후보를 만들고, 두 번째 pass가 exact match로 설명되지 않는 post/pre pair 중 direct source context가 있는 pair만 semantic bridge candidate로 제안한다.

### Challenge

Defender는 reachability, authentication, authorization, sanitizer, intended-public behavior, dead code와 default configuration을 이용해 각 primitive와 semantic bridge candidate를 독립적으로 반증한다. Judgment는 Recon claim과 Defender counterevidence를 함께 받아 `SUPPORTED`, `REBUTTED`, `UNKNOWN` 중 하나를 반환한다. 필수 criterion이 빠지거나 source reference가 맞지 않으면 코드가 결과를 `UNKNOWN`으로 덮어쓴다.

### Compose

`SUPPORTED` primitive만 사용한다. exact post/pre capability match를 결정적으로 연결하고, exact match가 아닌 경우에는 `SUPPORTED` semantic bridge와 direct evidence가 모두 있을 때만 연결한다. 같은 primitive를 한 chain에서 반복하지 않는다.

### Replay

지원된 chain만 격리 fixture 또는 target-specific harness에 전달한다. replay는 positive/negative/positive와 필수 link-drop을 실행하고 raw stdout, stderr, exit code와 observable effect를 보존한다.

### Report

report generator는 final review와 replay artifact를 입력으로 받는다. witnessed chain, unwitnessed supported chain, unknown chain, rebutted candidate와 no-finding을 서로 다른 artifact로 출력하며, 알려진 이슈와의 중복 여부는 security verdict가 아니라 disclosure metadata로 기록한다.

## Boundary Between Code and LLM

| Class | Code owns | LLM owns |
|---|---|---|
| Deterministic | phase order, schema, citation existence, counts, exact capability match, artifact paths, replay outcome | none |
| Candidate | syntax/AST/Semgrep observations, entry and sink candidates, context packaging | semantic interpretation proposal |
| Semantic | strict request/response routing and fail-closed parsing | attacker control, guard sufficiency, impact, bridge precondition satisfaction |

LLM response가 malformed이거나 evidence reference가 없으면 `UNKNOWN`이다. LLM은 자기 응답을 `WITNESSED`로 만들 수 없다.

## Core Types

```python
class ReviewVerdict(StrEnum):
    SUPPORTED = "supported"
    REBUTTED = "rebutted"
    UNKNOWN = "unknown"


class WitnessStatus(StrEnum):
    UNWITNESSED = "unwitnessed"
    WITNESSED = "witnessed"


@dataclass(frozen=True, slots=True)
class SourceRef:
    path: Path
    start_line: int
    end_line: int
    sha256: str


@dataclass(frozen=True, slots=True)
class Capability:
    kind: str
    resource: str
    scope: str


@dataclass(frozen=True, slots=True)
class Primitive:
    primitive_id: str
    preconditions: tuple[Capability, ...]
    operation: str
    postconditions: tuple[Capability, ...]
    evidence: tuple[SourceRef, ...]


@dataclass(frozen=True, slots=True)
class BridgeCandidate:
    producer_id: str
    consumer_id: str
    producer_evidence: tuple[SourceRef, ...]
    consumer_evidence: tuple[SourceRef, ...]


@dataclass(frozen=True, slots=True)
class Bridge:
    producer_id: str
    consumer_id: str
    evidence: tuple[SourceRef, ...]
    verdict: ReviewVerdict


@dataclass(frozen=True, slots=True)
class Chain:
    chain_id: str
    primitive_ids: tuple[str, ...]
    bridges: tuple[Bridge, ...]
```

Pydantic boundary model이 JSON을 parse한 뒤 위 immutable domain type으로 변환한다. 내부 함수는 untyped dictionary를 전달하지 않는다.

## Repository Structure

```text
src/crayfisher/
  cli.py
  model.py
  ids.py
  specification.py
  artifacts.py
  run.py
  profiling.py
  observations.py
  context.py
  semantic.py
  discovery.py
  composition.py
  challenge.py
  replay.py
  reporting.py

specifications/
  manifest.json
  pipeline.json
  agent-flows.json
  policies/

prompts/
  recon.md
  defender.md
  judgment.md

tests/
  fixtures/
  unit/
  integration/
  e2e/

docs/
  config/
  design/
  methodology/
  evaluation/
  superpowers/plans/
```

각 Python file은 한 책임과 250 pure LOC 이하를 유지한다.

## Run Artifacts

```text
runs/<run-key>/
  manifest.json
  profile.json
  observations.jsonl
  primitives.jsonl
  bridges.jsonl
  chains.jsonl
  challenges.jsonl
  replay/
  verdicts.jsonl
  summary.json
  report.md
```

`run-key`는 target commit, specification version, model identity와 run configuration의 canonical hash다. raw model request/response는 redaction 후 별도 artifact로 보존한다.

## Resolved Ambiguities from the Previous Prompts

1. Broken trace는 삭제하지 않고 `UNKNOWN` candidate로 보존한다. partial confidence `0.45`는 사용하지 않는다.
2. additive confidence score를 제거하고 evidence maturity와 explicit verdict를 사용한다.
3. 모든 validation criterion은 `pass`, `fail`, `unknown` 중 하나를 명시한다.
4. CVSS default vector를 사용하지 않는다. witnessed finding에 대해서만 vector와 rationale을 생성하고 parser로 검증한다.
5. malformed response는 `invalid-responses.jsonl`, no finding은 `no-findings.json`에 기록한다.
6. prompt tip은 `rejection_rule`과 `ranking_hint`로 분리한다.
7. policy condition은 `reportable and not excluded and verified`이며 exploit-chain policy의 필수 네 조건은 conjunction으로 유지한다.
8. known-duplicate 검사는 취약점 성립 조건에서 분리한다. `known`, `not_known`, `unknown` disclosure label로 기록하며 `unknown`이면 zero-day라고 주장하지 않는다.

## Evaluation Contract

v1 평가는 다음을 별도로 보고한다.

- primitive precision and recall
- bridge precision
- supported-chain precision
- replay success rate
- unknown and rebuttal rates
- link-drop necessity rate
- tokens, wall time and monetary cost
- repeated-run agreement
- root-cause-deduplicated new findings and disclosure status

Detect, compose, replay와 report success를 하나의 성공률로 합치지 않는다.
