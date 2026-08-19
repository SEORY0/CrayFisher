# Prompt Formalization to Executable Contracts

## Source

이 구현 계획은 이전 CrayFisher workspace의 다음 문서를 원자료로 사용한다.

```text
/home/seory0/projects/Ag3nt-Z3r0/CrayFisher/docs/crayfisher-prompt-formalization.md
```

원자료는 54개 prompt/policy/report file을 545개 원자 규칙으로 분해하고, 각 규칙을 `가능`, `후보만`, `LLM`으로 분류한다. 새 저장소는 원문을 그대로 실행 prompt로 복사하지 않고 traceable specification manifest로 옮긴다.

확인한 원자료는 아직 이전 workspace에서 Git에 추적되지 않은 파일이며 SHA-256은 `6b711acec51ea948b5760ec7d7c822e4275c22732a1b2d973df92d6cd3406344`이다. 따라서 위 절대 경로는 provenance 기록일 뿐 배포 의존성이 아니다. 구현 시 필요한 규칙 ID, source span과 분류를 `specifications/manifest.json`에 이관하여 새 저장소만으로 검사할 수 있게 한다.

| Source group | Traced rules |
|---|---:|
| root/agent | 96 |
| meta/recon | 50 |
| static/taint | 74 |
| validation | 118 |
| report/knowledge | 38 |
| policy conditions | 169 |
| **Total** | **545** |

## Normative Symbols

```text
r          frozen repository snapshot
p          vulnerability primitive
C          ordered primitive chain
ev(x)      source/config/runtime evidence supporting x
?          insufficient evidence
```

## Core Contracts Preserved from the Formalization

### Evidence

```text
claim(x) -> exists(file, line, bytes): ev(x)
not ev(x) -> verdict(x) = UNKNOWN
semgrep_match(x) -> candidate(x) and not evidence(x)
```

### Agent Review Order

```text
P = recon(r)
D = {defender(r, p) | p in P}
J = {judgment(r, p, D[p]) | D[p] != REBUTTED}
```

### A1-A10 Agent-Flow Candidates

원자료 `skills/03-taint/ai-agent-flows.md`의 핵심 path는 다음과 같이 보존한다. 각 식은 취약점 verdict가 아니라 candidate 생성 조건이다.

| Rule | Source span | Candidate condition |
|---|---:|---|
| A1 | 35-55 | `external_content -> tool_result -> LLM and not wrapper and dangerous_tool_same_agent` |
| A2 | 59-78 | `external_content -> store_raw -> later_retrieve -> LLM and not sanitize` |
| A3 | 81-99 | `attacker_controls(tool_result_bytes) and tool_result -> messages -> next_LLM_call and not sanitize` |
| A4 | 103-123 | `low_trust_subagent_output -> parent_prompt and not strict_parse` |
| A5 | 127-144 | `lower_trust_conflicting_instruction -> higher_priority_interpretation` |
| A6 | 148-165 | `external_content -> memory -> later_LLM and not wrapper` |
| A7 | 169-187 | `tool_A_output -> tool_B_argument -> dangerous_sink and default_no_approval` |
| A8 | 191-208 | `external_content -> goal_or_task -> follow_up_action and not goal_validation` |
| A9 | 212-228 | `candidate_content -> evaluator -> verdict -> follow_up_action` |
| A10 | 232-249 | `large_external_content and truncation_drops_system_prompt` |

syntax와 data-flow observation은 코드가 만들 수 있지만 wrapper 동등성, sanitizer 충분성, effective authority와 model interpretation은 semantic review가 판단한다.

### Chain Evidence

```text
N primitives -> N primitive citations and N - 1 bridge citations
missing evidence(link_i) or missing evidence(bridge_i) -> break(chain, i)
```

### Validation

```text
security_supported(p) = externally_reachable(p)
                        and default_triggerable(p)
                        and attacker_controls_value(p)
                        and meaningful_impact(p)

disclosure_status(p) in {known, not_known, unknown}
zero_day_claimable(p) = security_supported(p)
                        and disclosure_status(p) = not_known
```

네 security predicate의 control flow는 코드화할 수 있지만 semantic truth는 evidence-backed review로 남긴다. duplicate search 결과는 별도 provenance와 함께 기록하며 security verdict를 바꾸지 않는다.

## Implementation Classification

| Area | Deterministic code | Candidate generation | Strict semantic review |
|---|---|---|---|
| Orchestration | target split, phase order, state, artifact path, schema | agent-irrelevant scope | evidence interpretation |
| Recon | clone/profile/tool invocation | entries, trust promotion, CI and variant candidates | actual reachability and attacker control |
| Static analysis | result normalization, query count | Semgrep and source/sink observations | sanitizer and guard sufficiency |
| Agent flows | A1-A10 rule dispatch | promotion and tool-flow candidates | model interpretation and effective authority |
| Chains | ordered path, exact capability match, citation count | semantic bridge candidates | postcondition satisfies next precondition |
| Validation | criterion order, fail-closed state, CVSS parse | reportability and FP candidates | impact, intended behavior, exploitability |
| Reporting | schema and artifact completeness | report/PoC candidate | root cause and remediation correctness |

## Initial Rule Coverage

v0.1 manifest에는 원자료의 모든 rule ID와 source location을 등록한다. 실행 구현은 다음 normative core부터 시작한다.

| Group | v0.1 behavior |
|---|---|
| Pipeline branch/order | Deterministic |
| Evidence and schema constraints | Deterministic |
| A1-A10 agent flows | Candidate specifications |
| Exploit-chain link/bridge gate | Deterministic plus semantic bridge review |
| Four security criteria | Explicit tri-state review |
| Duplicate status | Separate `known`, `not_known`, `unknown` disclosure label |
| Agent authorization | Initial policy |
| Tool-result injection | Initial policy |
| Sandbox escape | Initial policy |
| Exploit chain | Initial policy with conjunctive reportable conditions |
| Remaining policies | Manifested but inactive until fixtures and tests exist |

## Specification Manifest Entry

각 원자 규칙은 다음 shape를 사용한다.

```json
{
  "rule_id": "agent-flow-a7",
  "source_document": "skills/03-taint/ai-agent-flows.md",
  "source_lines": [169, 187],
  "execution_class": "candidate",
  "stage": "discover",
  "implementation": "specifications/agent-flows.json",
  "acceptance_test": "tests/unit/test_specification.py::test_a7_is_candidate_rule"
}
```

`execution_class`는 `deterministic`, `candidate`, `semantic`, `reference` 중 하나다. `implementation` 또는 `acceptance_test`가 아직 없는 규칙은 manifest에서 `inactive`로 표시하되 삭제하지 않는다.

## Required Resolutions

원자료에서 발견된 충돌은 다음과 같이 고정한다.

| Previous ambiguity | v1 resolution |
|---|---|
| broken trace drop vs partial confidence | preserve as `UNKNOWN`; no numeric partial confidence |
| FP-check scope for intermediate scores | all supported chain candidates receive challenge; no score threshold routing |
| missing explicit criterion-3 result | every criterion is tri-state and criterion 3 is mandatory |
| missing incomplete-fix CVSS vector | no default vector; score only witnessed finding |
| unspecified confidence clamp | additive confidence removed |
| CVSS 0.0 label missing | CVSS parser reports `None` severity for 0.0 |
| malformed/no-finding artifact unspecified | fixed artifact names and schemas |
| tips mix rejection and severity | separate `rejection_rule` and `ranking_hint` |
| known duplicate mixed into validity | separate disclosure status from security verdict |

## Traceability Gate

CI는 다음을 검사한다.

1. manifest의 모든 active rule은 실제 implementation path를 가진다.
2. 모든 active deterministic/candidate/semantic rule은 acceptance test를 가진다.
3. source location과 rule ID가 중복되지 않는다.
4. inactive rule은 runtime dispatch에 포함되지 않는다.
5. prompt/spec version이 바뀌면 frozen replay campaign은 drift로 거부된다.
