# CrayFisher Plan

이 문서는 CrayFisher의 현재 상태, 구현 순서, 연구 검증 단계와 알려진 한계를 추적한다. 구현 여부와 논문에서 주장할 수 있는 결과를 같은 표에서 구분하는 것이 목적이다.

## Current State

### Research Scope

- [x] 연구 주제 정의: agent 구현의 code, configuration, persistent state와 runtime effect에 걸친 취약점 chain 자동 발견·검증
- [x] 기존 CrayFisher의 프롬프트 수식화 자료 확인
- [x] 결정적 코드, candidate generation, LLM semantic judgment의 경계 확인
- [x] `oss-crs`는 저장소 구조와 문서 정보 구조만 참고하고 구현·용어·runtime은 복제하지 않기로 결정
- [x] v1에서 evidence-ledger graph와 quality-diverse search를 제외하고 ordered chain 모델을 사용하기로 결정

### Available Source Material

| Source | Status | Use |
|---|---|---|
| `EuroS&P/RESEARCH_TOPIC.md` | Defined | 연구 범위와 주장 경계 |
| 이전 `docs/crayfisher-prompt-formalization.md` | Located | 54개 프롬프트와 545개 원자 규칙의 추적 원본 |
| 이전 A1-A10 agent-flow prompts | Located | agent-specific primitive 후보 규칙 |
| 이전 validation and policy prompts | Located | 반증 질문과 reportability 조건 |
| 이전 `experiments/c3fd_arm_a/`와 Arm B fixture | Available in previous workspace | 격리 replay, link-drop, hardened negative-control 설계 사례 |

### Implementation Status

| Component | Status | Notes |
|---|---|---|
| Python package and CLI | Not implemented | `uv` 기반 Python 3.12 package로 구축 |
| Typed primitive and chain model | Not implemented | graph 대신 ordered tuple과 explicit bridge 사용 |
| Formalization manifest | Not implemented | 모든 규칙의 source, class, implementation, test를 추적 |
| Target profiling | Not implemented | Python·TypeScript agent repository 우선 지원 |
| Primitive discovery | Not implemented | deterministic observations와 semantic candidate를 분리 |
| Chain composition | Not implemented | `Post(p_i) ⊨ Pre(p_{i+1})`와 edge evidence 검사 |
| Recon/Defender/Judgment contracts | Not implemented | prose handoff를 strict JSON schema로 교체 |
| Replay validation | Not implemented | positive/negative/positive control과 link-drop 사용 |
| Reporting | Not implemented | witnessed chain만 confirmed finding으로 출력 |
| Evaluation harness | Not implemented | baseline, ablation, cost, reproducibility 측정 |

## Near-Term: Research Core v0.1

### Milestone 1: Executable Specification and Artifacts

| Item | Status | Acceptance |
|---|---|---|
| Strict Python project | Planned | Ruff, Basedpyright, Pytest가 모두 통과 |
| Typed evidence/primitive/bridge/chain records | Planned | illegal state가 schema parse 단계에서 거부됨 |
| Prompt-formalization manifest | Planned | 원자 규칙마다 source와 execution class가 존재 |
| Deterministic run directory | Planned | 같은 target/spec/config가 같은 run key를 생성 |
| Machine-readable stage artifacts | Planned | 각 단계가 JSON/JSONL artifact를 남김 |

**Why it matters:** 이 단계가 없으면 수식은 논문 장식이고, 실제 실행은 다시 prompt convention에 의존한다.

### Milestone 2: Agent-Aware Primitive Discovery

| Item | Status | Acceptance |
|---|---|---|
| Target profiler | Planned | agent/non-agent/hybrid와 language를 구분 |
| Static observation adapters | Planned | tool, credential, storage, sandbox, LLM call 위치를 정규화 |
| Evidence context builder | Planned | 고정 line window가 아니라 enclosing function과 caller evidence를 제공 |
| Recon semantic contract | Planned | primitive candidate를 strict schema로 반환 |
| Initial policy set | Planned | agent authorization, tool-result injection, sandbox escape, exploit chain |

### Milestone 3: Chain Validation

| Item | Status | Acceptance |
|---|---|---|
| Defender challenge | Planned | broken guard, unreachable path, intended-public behavior를 반증 |
| Exact capability composition | Planned | `SUPPORTED` primitive의 exact post/pre match를 결정적으로 연결 |
| Semantic bridge review | Planned | LLM bridge는 evidence와 challenge 없이 지원 상태가 될 수 없음 |
| Judgment contract | Planned | `SUPPORTED`, `REBUTTED`, `UNKNOWN`만 출력 |
| Chain evidence gate | Planned | N links와 N−1 bridges 중 하나라도 evidence가 없으면 chain이 깨짐 |

### Milestone 4: Replay and Reporting

| Item | Status | Acceptance |
|---|---|---|
| Isolated replay protocol | Planned | host·real credential을 사용하지 않음 |
| Positive/negative/positive control | Planned | terminal effect가 chain input에 의존함을 확인 |
| Link-drop controls | Planned | 필수 link 제거 시 terminal effect가 사라짐 |
| Report generator | Planned | witnessed chain만 security finding으로 출력 |
| No-finding and malformed artifacts | Planned | exact filename과 schema가 고정됨 |

## Mid-Term: EuroS&P Evaluation

| Item | Status | Purpose |
|---|---|---|
| Historical vulnerable/fixed corpus | Planned | recall과 false positive를 측정 |
| Real agent application campaign | Planned | cross-stage chain의 현실성을 검증 |
| Baseline comparison | Planned | Agent Audit, generic code agent, no-chain variant와 비교 |
| Ablation study | Planned | chain composition, Defender, replay의 기여도 분리 |
| Repeated runs | Planned | model stochasticity와 result stability 측정 |
| Responsible disclosure ledger | Planned | 신규성·중복·벤더 확인 상태를 분리 |

## Long-Term

| Item | Status | Purpose |
|---|---|---|
| Go and Rust adapters | Design goal | agent runtime와 SDK 분석 범위 확장 |
| Incremental diff campaigns | Design goal | release·PR 기반 지속 분석 |
| Additional policy compilation | Design goal | 초기 4개 policy 이후 21개 policy 확장 |
| Alternative search strategies | Design goal | ordered-chain baseline 이후에만 비교 |

## Known Limitations

1. 프롬프트 수식화만으로 semantic judgment의 정확성이 증명되지는 않는다.
2. v0.1은 Python과 TypeScript agent repository를 우선 대상으로 한다.
3. exact capability composition 밖의 bridge는 semantic review와 실행 증거가 필요하다.
4. `SUPPORTED`는 취약점 확정이 아니며 `WITNESSED` replay가 있어야 confirmed finding이다.
5. 실제 zero-day와 기존 방법 대비 우수성은 evaluation campaign 이전에 주장하지 않는다.
6. 이전 프롬프트의 confidence 가중치는 근거가 불명확하므로 v0.1에서 사용하지 않는다.

## Implementation Plans

- [Core specification and artifacts](docs/superpowers/plans/2026-08-19-crayfisher-core.md)
- [Agent pipeline and chain composition](docs/superpowers/plans/2026-08-19-crayfisher-agent-pipeline.md)
- [Replay, reporting, and evaluation](docs/superpowers/plans/2026-08-19-crayfisher-replay-evaluation.md)

## Documentation

문서 인덱스는 [docs/README.md](docs/README.md)에 있다. 구현 상태가 변하면 먼저 이 계획의 status와 known limitations를 갱신하고, 사용자 동작이 바뀌면 해당 canonical document를 함께 수정한다.
