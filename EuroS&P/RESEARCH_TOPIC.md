# EuroS&P 연구 주제

## 연구 시스템 정의

CrayFisher는 **agent 시스템의 분산된 권한 전이를 복원하고, 여러 표현 및
실행 단계에 걸친 취약점 chain을 실행 증거로 검증하는 자동 분석
시스템**이다.

## 가제

**CrayFisher: Automated Discovery and Evidence-Backed Validation of
Cross-Stage Authority Chains in Agent Systems**

## 문제 정의

기존 agent 취약점 분석은 주로 단일 source-to-sink 경로, 개별 tool call,
또는 이미 정의된 tool chain을 검사한다. 그러나 실제 agent 시스템의 권한은
소스 코드, 설정, 직렬화된 workflow, tool output, persistent state, runtime
credential resolution과 외부 provider를 거치면서 분산되고 변형된다. 이로
인해 각 단계만 보면 정상처럼 보이지만, 여러 단계를 연결하면 권한 상승,
credential 탈취, sandbox escape 또는 원격 코드 실행으로 이어지는 취약점이
발생한다.

## 핵심 연구 질문

1. agent 시스템의 코드와 설정에서 공격자 통제, 신뢰 경계 및 capability
   변화를 자동으로 복원할 수 있는가?
2. 서로 다른 표현과 실행 단계에 흩어진 취약점 primitive를 실제 exploit
   chain으로 자동 조합할 수 있는가?
3. 모든 chain link와 transition에 코드 위치, 실행 결과 및 negative control을
   요구하여 오탐을 억제하면서 신규 취약점을 발견할 수 있는가?

## 예상 핵심 공헌

1. 소스 코드부터 runtime effect까지 이어지는 agent 권한 전이 분석 단위.
2. Recon → Defender → Judgment를 통한 후보 생성, 반증 및 독립 판정 절차.
3. 모든 link와 edge에 증거 의무를 부과하는 exploit-chain 검증 방식.
4. 실제 agent framework와 application을 대상으로 한 자동 취약점 발견 및
   end-to-end replay artifact.

## 프롬프트 수식화의 역할

`docs/crayfisher-prompt-formalization.md`는 별도의 논문 주제가 아니라
CrayFisher의 분석 상태, 전이 조건, 증거 의무, deterministic code와 LLM
semantic judgment의 경계를 명시하는 방법론 및 재현성 근거로 사용한다.

## 주장 경계

논문의 중심 주장은 단순히 "LLM이 agent의 버그를 찾는다"가 아니다.
CrayFisher가 기존 단일 경로 또는 사전 정의된 tool-chain 분석이 놓치는
**cross-stage authority chain**을 복원하고, 이를 실제 실행과 반증 가능한
증거로 검증한다는 것이 핵심 주장이다.
