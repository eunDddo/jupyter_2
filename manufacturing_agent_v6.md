# Manufacturing AI Agent — v6 설명서

> 파일: `manufacturing_agent_v6.ipynb`
> 한 줄 요약: v5(Supervisor 허브)에 **Input Guardrail 핸드오프(2026-06-19)의 T5**를 반영한 버전. 입력 가드레일을 stateless로 되돌려 **제어·승인 명령(4b)을 통과**시키고, 위험 '실행' 부분집합은 신규 **`safety_gate`**(입력 직후)가 차단한다.

---

## 0. v5 → v6 변경 (핸드오프 반영)

핸드오프(`2026-06-19-v5-input-guardrail-handoff.md`)의 결론: v5는 T5를 **정반대로**(제어·승인 4b를 입력 단에서 차단) 구현했고, 그 차단을 풀려면 위험-차단 계층이 하류에 **반드시 함께** 있어야 한다(C3 — 동시 머지). v6은 권장안인 **옵션 (가) `safety_gate` 노드 추가**로 두 가지를 한 번에 적용했다.

| 항목 | v5 | v6 |
|---|---|---|
| 제어·승인 명령(4b) "설비 정지시켜", "가동 승인해줘" | 입력 단에서 `no_control_authority`로 **차단** | 입력 단 **통과(PASS)** — stateless |
| 위험 '실행' 명령 "점검 없이 재가동", "안전장치 우회" | (차단 계층 없음) | 신규 **`safety_gate`가 차단(BLOCK)** |
| 2층 LLM 차단 범위 | gibberish/out_of_scope/no_control_authority | gibberish/out_of_scope **만**(화이트리스트) |
| 그래프 | `input_gate → context_manager → …` | `input_gate → **safety_gate** → context_manager → …` |

> 핸드오프 §3-1~3-6(입력 단 완화) + §4 옵션(가)/부록 B(safety_gate)를 **하나의 변경 단위**로 적용(C3 준수).

---

## 1. 개요

제조 설비(밀링)의 **고장 위험 예측 + 매뉴얼 근거 검색(RAG)**을 수행하는 LangGraph 멀티에이전트 시스템이다. 사용자의 자연어 질문 또는 프론트엔드의 구조화 수치 입력을 받아:

1. **입력 가드레일(input_gate)**로 보안·유효성을 거른 뒤(빈입력/인젝션/gibberish/out_of_scope만 차단),
2. **safety_gate**가 위험 '실행' 명령을 차단하고,
3. **Supervisor**(중앙 오케스트레이터)가 다음에 실행할 에이전트를 결정하고,
4. **Prediction → Evidence** 순으로 산출물을 만들어,
5. **FinalAnswer**가 예측·근거를 통합한 한국어 답변을 생성한다.

오프라인(API 키 없음)에서도 결정론적 폴백(StubLLM, 규칙 라우팅, 정규식 게이트)으로 끝까지 실행된다.

---

## 2. 아키텍처 — Supervisor 중심 허브 (+ safety_gate)

```
User
  │
  ▼
input_gate ──(BLOCK: empty/injection/gibberish/out_of_scope)──┐
  │ (PASS)                                                     │
  ▼                                                            │
safety_gate ──(BLOCK: 위험 실행 명령)───────────────────────────┤
  │ (PASS)                                                     │
  ▼                                                            ▼
context_manager                                          final_answer ─▶ output_gate ─▶ memory_writer ─▶ END
  │                                                            ▲
  ▼                                                            │
supervisor ◀───────────────┐                                  │
  │  (route)               │                                  │
  ├─▶ prediction_agent ─▶ prediction_gate ─┘ (supervisor로 복귀)
  ├─▶ evidence_agent   ─▶ evidence_gate   ─┘ (supervisor로 복귀)
  └─▶ final_answer
```

- **input_gate**: 보안·최소 유효성만(stateless). 제어·승인 명령은 통과.
- **safety_gate**: 입력 직후. 위험 '실행' 부분집합(재가동 강행/안전장치 우회/경고 무시/점검 전 가동)만 정규식으로 차단. BLOCK 시 `final_answer`가 안내 메시지 출력.
- **Supervisor = LLM ReAct 라우터**: 현재 state를 CoT로 추론해 `next_node` 결정, 실패 시 `agent_feedback`로 재시도 전략 전달. 키 없으면 규칙 라우터(`_rule_route`) 폴백.
- **무한 루프 방지**: `MAX_RETRY = 3`, `retry_counts`, `recursion_limit = 40`.

---

## 3. 입력 가드레일 + safety_gate 동작 (핵심 변경)

### input_gate (stateless)
- **1층 정규식(비용 0)**: 빈 입력 → `empty`, 노골적 인젝션 → `injection` 만 즉시 차단.
- **2층 경량 LLM(모호한 경우만)**: `gibberish`/`out_of_scope` **만** 차단(화이트리스트). 키 없음/실패 시 1층-only 폴백(통과).
- **제어·승인 명령(4b)은 통과** — "위험 실행 여부는 하류에서 판단"한다는 원칙. `CONTROL_COMMAND_PATTERNS`/`no_control_authority`는 제거됨.

### safety_gate (신규)
- `FORBIDDEN_PATTERNS`(정규식)로 **위험을 키우는 실행**만 BLOCK:
  - 점검 전 재가동, 안전장치 우회/해제, 경고·알람 무시 강행, 재가동 강행.
- **PASS 대상**(over-block 방지): 안전측 정지("정지시켜"), 자문("올리면 위험해?"), 정상 진단.
- 에이전트가 아니라 **게이트**(추론/LLM 없는 정규식 1계층) — 책임이 한 곳에 모이고 추적 쉬움.
- BLOCK 시 `final_answer_node`가 `SAFETY_BLOCK_MESSAGE`("설비를 직접 제어·재가동…현장 안전 책임자에게…")를 출력.

> `FORBIDDEN_PATTERNS`는 핸드오프 부록 B의 **시작 스켈레톤**이다. 실제 위험 어휘 범위는 Safety 담당이 `build_safety_hints`의 `avoid_actions` 등과 맞춰 확정해야 한다.

---

## 4. 데이터 계약(주요 스키마)

### `PredictionResult` — 단일 통합 스키마
- `final_answer`가 소비: `full_prediction_available`, `partial_risks`(`FailureRisk` 리스트), `used_stale_features`, `limitations`, `summary`
- `rag_service`(build_query **mode B**)가 소비: `failure_types`, `cause_features`
- `prediction_agent`가 양쪽을 모두 채운다(`partial_risks`에서 `failure_types`/`cause_features` 파생).

### 그 외
- `FailureRisk`/`EvidenceHint`/`SafetyHint`: 규칙 기반 예측 산출물.
- `EvidenceBundle`: RAG 결과 + supervisor 연동 관측 필드(`supervisor_intent`/`feedback`/`is_retry`).
- `InputFlags`: 관측용(`is_empty`/`is_injection`/`is_control_command`(항상 False)/`is_manufacturing`).
- `InputDecision`: 가드레일 판정(`blocked`/`reason`/`layer`/`block_message`). `reason` enum에서 `no_control_authority` 제거.
- `MachineFeatureInput`: 프론트 구조화 수치 입력 계약(`extra="forbid"`).
- `SupervisorPlan`: LLM ReAct 라우터 structured output(CoT `reasoning` + `next_node` + `retry_strategy`).
- `ManufacturingState`: `MessagesState` 상속(input_features/input_decision/supervisor_plan/agent_feedback).

---

## 5. 노트북 구성(섹션)

| 섹션 | 내용 |
|---|---|
| 0 | 설치 & 환경 |
| 1 | 설정 & LLM 어댑터(`.env`, `call_llm`, StubLLM 폴백) |
| 2 | `contracts/` — Pydantic 스키마 + `ManufacturingState` |
| 3 | `memory/` — 장기 메모리(SQLite) |
| 4 | ChromaDB RAG 런타임(`vector_search`) |
| 5 | `context/` — Context Engineering |
| 6 | `services/` — `run_prediction` + `rag_service` |
| 7 | `agents/` — `prediction_agent`, `evidence_agent` (+ 데모) |
| 8 | `gates/` — input(가드레일) / **safety** / prediction / evidence / output |
| 9 | `nodes/` — FinalAnswer + MemoryWriter |
| 10 | `context_manager` 진입점 노드 |
| 11 | `graph/` — Supervisor + route_policy + 그래프 조립(safety_gate 배선 포함) |
| 12 | 단기/장기 체크포인터(SqliteSaver) → `app` |
| 13 | 그래프 시각화(선택) |
| 14 | 실행 — 멀티턴 시나리오 |
| 15 | 정리 |
| (테스트) | Input Guardrail DoD + **safety_gate 검증** |

---

## 6. 실행 방법

### 사전 준비
1. `.env`에 `OPENAI_API_KEY`(없으면 StubLLM/규칙 폴백).
2. **`01_embed_documents_chroma.ipynb`를 먼저 실행**해 ChromaDB(`agent_data/chroma`) 임베딩.

### 실행
- 위에서부터 순서대로 실행(Run All).
- 진입 함수: `run_turn(user_message, user_id, thread_id, request_id, input_features=None)`
  - `input_features`: 프론트 구조화 수치 dict(자연어 파싱보다 우선).
  - 같은 `thread_id`면 단기 체크포인터로 맥락 유지.

---

## 7. 검증 결과

`jupyter nbconvert --execute` 전체 실행 기준:

- **에러 0건**.
- 게이트 흐름: `input_gate → safety_gate → prediction_gate → evidence_gate → output_gate` 전부 PASS.
- **Input Guardrail DoD**: empty/injection 차단, **제어·승인(4b) PASS(T5)**, final_answer passthrough.
- **safety_gate 6케이스**: 점검없이 재가동/안전장치 우회/경고 무시 → BLOCK, 정지/자문/진단 → PASS.
- 제어·승인(4b) 입력 가드레일 PASS 확인.

### 핸드오프 체크리스트 대응
- [x] C2 위치 결정: **옵션 (가) safety_gate 노드** 채택.
- [x] 위험-차단 계층 구현(부록 B `safety_gate` + 그래프 배선).
- [x] §3-1~3-6 적용 — 차단 계층과 **동일 변경 단위로 머지**(C3).
- [x] 회귀: empty/injection 차단, 정상 제조 질문/ features-only PASS.
- [x] T5 신규: `설비 정지시켜`·`승인해줘` → 입력 가드레일 PASS.
- [x] 안전 신규: `점검 없이 재가동해` 류 → safety_gate BLOCK.

### 알아둘 점
- **멀티턴 stale 방지**: `run_turn`이 매 턴 agent 산출물을 초기화(v5에서 도입).
- **`Deserializing unregistered type ...` 경고**: pydantic state의 msgpack 직렬화 경고(무해, 에러 아님).

---

## 8. 향후 확장 포인트

- `FORBIDDEN_PATTERNS` 위험 어휘 범위 확정(Safety 담당과 `avoid_actions` 정합).
- safety_gate 배치 대안: 출력 직전(`output_gate` 인접)으로 옮겨 생성 답변까지 검사(더 견고, 배선 증가).
- Explorer 예측(WHAT_IF/민감도) 복원: `prediction_router` + 조건부 엣지 재배선.
- LLM structured-output 안정화: state에 pydantic 대신 dict 저장 검토(msgpack 경고 제거).
