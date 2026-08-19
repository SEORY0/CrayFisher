# CrayFisher Documentation

CrayFisher 문서는 연구 주장, 실행 아키텍처, 프롬프트 수식화, 구현 계획과 실험 재현을 분리한다. 같은 개념을 여러 문서에서 다시 정의하지 않고 아래 canonical document로 연결한다.

## Research Scope

| Document | Purpose |
|---|---|
| [EuroS&P Research Topic](../EuroS&P/RESEARCH_TOPIC.md) | 연구 문제와 핵심 주장 |
| [Project Plan](../PLAN.md) | 현재 구현 상태, milestone, 알려진 한계 |

## Architecture and Methodology

| Document | Purpose |
|---|---|
| [Architecture](design/architecture.md) | component, lifecycle, typed interface와 artifact flow |
| [Prompt Formalization Mapping](methodology/prompt-formalization.md) | 기존 프롬프트 수식을 실행 계약으로 옮기는 경계 |

## Implementation Plans

| Plan | Deliverable |
|---|---|
| [Core](superpowers/plans/2026-08-19-crayfisher-core.md) | package, domain model, specification, run artifacts |
| [Agent Pipeline](superpowers/plans/2026-08-19-crayfisher-agent-pipeline.md) | profile, discovery, semantic contracts, chain validation |
| [Replay and Evaluation](superpowers/plans/2026-08-19-crayfisher-replay-evaluation.md) | runtime witness, reports, ablation, CI |

## Planned Canonical References

구현 과정에서 다음 문서를 해당 milestone과 같은 commit에 추가한다.

| Path | Content |
|---|---|
| `docs/getting-started.md` | 설치와 첫 fixture scan |
| `docs/config/run-config.md` | target, model, policy, budget schema |
| `docs/config/artifacts.md` | stage별 JSON/JSONL schema |
| `docs/development-guide.md` | adapter, policy, replay fixture 추가 방법 |
| `docs/evaluation/protocol.md` | corpus, baseline, ablation, metric, statistical protocol |
| `docs/reproducibility.md` | frozen commit, model, prompt/spec version과 exact commands |

## Key Concepts

CrayFisher v1의 lifecycle은 다음 여섯 단계다.

1. **Profile** — target과 agent-specific surface를 고정한다.
2. **Discover** — code/configuration에서 vulnerability primitive 후보를 만든다.
3. **Challenge** — Defender가 각 primitive와 semantic bridge candidate를 반증하고 Judgment가 tri-state review를 만든다.
4. **Compose** — 지원된 primitive의 postcondition과 다음 precondition을 연결한다.
5. **Replay** — 격리 환경에서 terminal effect와 negative control을 실행한다.
6. **Report** — witnessed chain과 미해결 evidence gap을 분리해 출력한다.
