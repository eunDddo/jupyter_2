# Manufacturing AI Agent — v5 설명서

> 파일: `manufacturing_agent_v5.ipynb`
> 한 줄 요약: 파트별로 모은 통합본(`manufacturing_agent_v4_sup_final.ipynb`)에서 충돌하던 의사결정 지점을 정리해, **가장 깔끔하고 실제로 끝까지 실행되는** 단일 멀티에이전트 파이프라인으로 만든 버전.

---

## 1. 개요

제조 설비(밀링)의 **고장 위험 예측 + 매뉴얼 근거 검색(RAG)**을 수행하는 LangGraph 멀티에이전트 시스템이다. 사용자의 자연어 질문 또는 프론트엔드의 구조화 수치 입력을 받아:

1. **입력 가드레일**로 보안·유효성을 거른 뒤,
2. **Supervisor**(중앙 오케스트레이터)가 다음에 실행할 에이전트를 결정하고,
3. **Prediction → Evidence** 순으로 산출물을 만들어,
4. **FinalAnswer**가 예측·근거를 통합한 한국어 답변을 생성한다.

오프라인(API 키 없음)에서도 결정론적 폴백(StubLLM, 규칙 라우팅)으로 끝까지 실행되도록 설계했다.

---

## 2. 아키텍처 — Supervisor 중심 허브

```
User
  │
  ▼
input_gate ──(BLOCK)──────────────┐
  │ (PASS)                         │
  ▼                                │
context_manager                    │
  │                                ▼
  ▼                            final_answer ─▶ output_gate ─▶ memory_writer ─▶ END
supervisor ◀───────────────┐      ▲
  │  (route)               │      │
  ├─▶ prediction_agent ─▶ prediction_gate ─┘ (supervisor로 복귀)
  ├─▶ evidence_agent   ─▶ evidence_gate   ─┘ (supervisor로 복귀)
  └─▶ final_answer
```

- **모든 gate는 supervisor로 복귀**한다. supervisor가 gate report와 retry 횟수를 보고 다음 노드를 다시 결정한다(반응형 라우팅).
- **Supervisor = LLM ReAct 라우터**: 현재 state를 CoT(Chain of Thought)로 추론해 `next_node`를 고른다. 실패(FAIL/RETRY/INSUFFICIENT) 시 재진입 에이전트에 **실패 사유/전략(`agent_feedback`)**을 실어 보내고, 에이전트는 이를 읽어 접근을 바꾼다.
- **폴백**: LLM 키가 없거나 호출 실패/무효 출력이면 결정론적 규칙 라우터(`_rule_route`)가 동일한 흐름으로 동작한다.
- **무한 루프 방지**: `MAX_RETRY = 3`, agent 실행마다 `retry_counts` 증가, `recursion_limit = 40`. 한도 초과 시 `final_answer`로 강제 종료.

---

## 3. 통합 시 내린 핵심 결정

`*_v4_sup_final` 통합본은 여러 사람이 작업한 파트를 모은 것이라 `#====` 사이 중복 클래스와 `v1/v2/v3` 대안 셀이 서로 충돌했다. v5는 다음과 같이 정리했다.

| 충돌 지점 | 채택 | 이유 |
|---|---|---|
| **아키텍처** | Supervisor 허브 (LLM ReAct + 규칙 폴백) | `_sup` 파트의 핵심 설계. gate→supervisor 복귀로 재시도 피드백이 동작 |
| **Safety** | 전면 제거 | supervisor/graph가 `safety_agent`/`safety_gate`를 참조하는데 **정의가 없어 그래프가 깨졌음** |
| **Prediction** | 단일 `prediction_agent` + 규칙 기반 `run_prediction` | 허브 그래프는 단일 노드를 배선. Explorer 멀티노드(WHAT_IF/EXPLORATORY)는 미배선이라 제외 |
| **PredictionResult** | 단일 통합 스키마 | 3개 버전이 제각각 다른 필드를 가정 → 한 모델로 통합(아래 4장) |
| **Input Guardrail** | v2 `InputDecision` 2층(정규식+경량 LLM) | v1은 정의 없는 `call_llm_json`에 의존. v2가 supervisor/state와 정합 |
| **State** | `MessagesState` 상속 | `input_features`/`input_decision`/`supervisor_plan`/`agent_feedback` 추가 |

### 판단으로 결정한 부분(의도와 다르면 조정 필요)
1. **Explorer 예측(WHAT_IF/민감도 분석) 제외** — 허브 그래프에 배선돼 있지 않아 뺐다. 살리려면 `prediction_router` + 조건부 엣지로 그래프를 다시 배선해야 한다.
2. **제어·승인 명령 차단** — v2 가드레일을 따라 입력 단에서 차단(`no_control_authority`)한다. (v1 시안은 통과시켰음.)

---

## 4. 데이터 계약(주요 스키마)

### `PredictionResult` — 단일 통합 스키마
세 컴포넌트가 서로 다른 필드를 기대했기에 하나로 합쳤다.

- `final_answer`가 소비: `full_prediction_available`, `partial_risks`(`FailureRisk` 객체 리스트), `used_stale_features`, `limitations`, `summary`
- `rag_service`(build_query **mode B**)가 소비: `failure_types`(예: `["OSF","TWF"]`), `cause_features`(예: `["torque","tool_wear"]`)
- `prediction_agent`가 두 쪽을 모두 채운다(`partial_risks`에서 `failure_types`/`cause_features`를 파생).

### 그 외
- `FailureRisk` / `EvidenceHint` / `SafetyHint`: 규칙 기반 예측 산출물.
- `EvidenceBundle`: RAG 결과 + supervisor 연동 관측 필드(`supervisor_intent`, `feedback`, `is_retry`).
- `InputFlags`: 라우팅용 아님, 가드레일 최소 보안 관측용(`is_empty`/`is_injection`/`is_control_command`/`is_manufacturing`).
- `InputDecision`: 가드레일 판정(`blocked`/`reason`/`layer`/`block_message`).
- `MachineFeatureInput`: 프론트엔드 구조화 수치 입력 계약(`extra="forbid"`로 오타/주입 차단).
- `SupervisorPlan`: LLM ReAct 라우터의 structured output(CoT `reasoning` + `next_node` + `retry_strategy`).
- `ManufacturingState`: `MessagesState` 상속 TypedDict(노트북 §2.1).

---

## 5. 노트북 구성(섹션)

| 섹션 | 내용 |
|---|---|
| 0 | 설치 & 환경 |
| 1 | 설정 & LLM 어댑터(`.env` 로드, `call_llm`, StubLLM 폴백) |
| 2 | `contracts/` — Pydantic 스키마 + `ManufacturingState`(MessagesState 상속) |
| 3 | `memory/` — 장기 메모리(SQLite, `ConversationStore`/`RunStore`) |
| 4 | ChromaDB RAG 런타임(`vector_search`) |
| 5 | `context/` — Context Engineering(policy/selector/normalizer/packer) |
| 6 | `services/` — `run_prediction`(규칙 기반) + `rag_service`(Query Builder→Retriever→Ranker→Citation) |
| 7 | `agents/` — `prediction_agent`, `evidence_agent` (+ mode A/B 데모) |
| 8 | `gates/` — input(가드레일) / prediction / evidence / output |
| 9 | `nodes/` — FinalAnswer + MemoryWriter |
| 10 | `context_manager` 진입점 노드 |
| 11 | `graph/` — Supervisor(ReAct) + route_policy + 그래프 조립 |
| 12 | 단기/장기 체크포인터(SqliteSaver) → `app` 컴파일 |
| 13 | 그래프 시각화(선택) |
| 14 | 실행 — 멀티턴 시나리오(자연어 / 구조화 수치) + 메모리·체크포인트 확인 |
| 15 | 정리 |
| (테스트) | Input Guardrail 오프라인 DoD 검증 |

---

## 6. 입력 가드레일(Input Guardrail) 동작

`input_gate`는 라우팅이 아니라 **'서비스 가능 여부'**만 판정한다.

- **1층(정규식, 비용 0)**: 빈 입력 → `empty`, 노골적 인젝션 → `injection`, 제어·승인 명령 → `no_control_authority`. (단, `~하면 위험해?` 같은 자문 질문은 통과.)
- **2층(경량 LLM, 모호한 경우만)**: `gibberish`/`out_of_scope` 판정. 키 없음/실패 시 1층-only 폴백(통과).
- 차단 시 `final_answer`가 차단 메시지를 그대로 출력한다.
- 통과 시 raw query를 `messages`에 실어 supervisor로 넘긴다(라우팅은 supervisor가 결정).

---

## 7. 실행 방법

### 사전 준비
1. `.env`에 `OPENAI_API_KEY` 설정(없으면 StubLLM/규칙 폴백으로 동작).
2. **`01_embed_documents_chroma.ipynb`를 먼저 실행**해 `document/`를 ChromaDB(`agent_data/chroma`)에 임베딩.

### 실행
- 노트북을 위에서부터 순서대로 실행(Run All).
- 핵심 진입 함수:
  ```python
  run_turn(user_message, user_id, thread_id, request_id, input_features=None)
  ```
  - `input_features`: 프론트 구조화 수치 dict(예: `{"type":"M","torque":62.0, ...}`). 자연어 파싱보다 우선.
  - 같은 `thread_id`로 호출하면 단기 체크포인터로 맥락이 이어진다.

### 시나리오
- **시나리오 1**: 자연어 수치 + 진단 질의.
- **시나리오 2**: 구조화 수치 dict + 자연어 질의(구조화 값 우선 반영, 충돌 시 경고).

---

## 8. 검증 결과

`jupyter nbconvert --to notebook --execute` 전체 실행 기준:

- **에러 0건**.
- 게이트 흐름 정상: `input_gate → prediction_gate → evidence_gate → output_gate` 전부 PASS.
- RAG 데모 mode A/B 동작(mode B는 `failure_types` 기반 예측 결합 검색).
- 구조화 수치 입력 우선·충돌 경고 정상.
- 단기 체크포인터 복원 정상.
- Input Guardrail 오프라인 DoD(정규식 케이스 + final_answer passthrough) 통과.

### 알아둘 점
- **멀티턴 stale 결과 방지**: 영속 SqliteSaver가 같은 `thread_id`를 resume할 때 직전 턴의 `prediction_result`/`evidence_bundle`을 그대로 들고 오면 supervisor가 "다 끝남→종료"로 단축돼 stale 답변이 나온다. → `run_turn`이 매 턴 agent 산출물(`prediction_result`/`evidence_bundle`/`final_answer`/`supervisor_plan`/`route`/`intent`/`agent_feedback`/`context_packet`/`input_decision`)을 초기화해 해결했다.
- **`Deserializing unregistered type ...` 경고**: pydantic state 객체를 SqliteSaver가 msgpack으로 직렬화할 때 나오는 **무해한 경고**(에러 아님). 기존 노트북에도 동일하게 존재.

---

## 9. 향후 확장 포인트

- Explorer 예측(WHAT_IF/민감도) 복원: `prediction_router` + 조건부 엣지로 그래프 재배선.
- Safety 서브시스템 재도입(필요 시): `safety_agent`/`safety_gate` 정의 + supervisor 라우팅/digest/그래프 노드 추가.
- LLM structured-output 안정화: msgpack 직렬화 경고 제거를 위해 state에 pydantic 대신 dict 저장으로 전환 검토.
