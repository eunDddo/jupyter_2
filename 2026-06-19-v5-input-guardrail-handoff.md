# v5 Input Guardrail 반영 점검 — 팀장 전달 핸드오프 (2026-06-19)

> 작성: Input Guardrail 담당 / 대상: 팀장(v5 리팩토링 책임)
> 비교: `manufacturing_agent_v2.ipynb`(내 작업본, 기준) ↔ `manufacturing_agent_v5.ipynb`(팀장 통합본)
> 기준 문서: `docs/2026-06-19-input-guardrail-changes.md`
> **이 파일은 Claude Code가 그대로 읽고 빠진 부분을 리팩토링할 수 있도록 작성됨.** 아래 §3이 입력 단 실행 스펙, §4가 안전 의존성 계약/결정, 부록 B가 `safety_gate` 스켈레톤.

---

## 1. 결론 한 줄

T1·T2·T3·T6는 v5에 (구조는 바뀌었지만) **반영됨**. **T5(제어·승인 4b 통과 전환)만 누락**이고, v5는 오히려 **정반대로(4b를 입력 단에서 차단) 구현**되어 있다. 단, T5가 전제하는 **하류 위험-차단 계층이 v5엔 없다**(통합 시 `SafetyAgent` 삭제, `safety_gate`는 미구현). 따라서 §3(입력 단)만 단독 적용하면 안전 구멍이 생기므로, **위험-차단 계층을 한 곳에 두는 결정(§4)과 함께** 적용해야 한다. 권장 경로는 부록 B의 `safety_gate` 노드 1개 추가.

---

## 2. 태스크별 반영 현황

| 태스크 | v2 의도 | v5 상태 | 판정 |
| --- | --- | --- | --- |
| **T1** 1층 정규식 인젝션 보강 | `INJECTION_PATTERNS` 한/영 변형 5개 추가 | v5 `INJECTION_PATTERNS`에 동일 패턴 + `[GUARDRAIL-260619] T1` 마커까지 그대로 존재 (cell, ~L724–732) | ✅ 반영 |
| **T2** 2층 경량 LLM | `call_llm_json`으로 gibberish/out_of_scope 판정, 실패 시 정규식-only 폴백 | `_llm_guardrail()` + `GUARDRAIL_SYS`로 리팩토링(~L1496–1525). 키 없음/실패→None 폴백 동일 | ✅ 반영(이름만 변경) |
| **T3** 리포트가 판단을 *설명*만 | `GateReport`에 block/reason/layer/message | `InputDecision`(blocked/reason/layer/block_message, ~L356–362) + `_guardrail_result()`가 `GateReport` 기록(~L1527). 라우팅은 `status==PASS`로만 분기 | ✅ 반영(구조 분리) |
| ~~**T4**~~ | (철회됨) | v5에 재예측 휴리스틱 흔적 없음 | ✅ 철회 일치 |
| **T5** 제어·승인 4b **통과** | 4b를 PASS시키고 위험 부분집합만 하류 안전 계층이 차단(stateless) | ❌ **정반대**. 4b를 입력 단 1층에서 `no_control_authority`로 **차단**. 게다가 하류 위험-차단 계층 부재(`SafetyAgent` 삭제, `safety_gate` 미구현) | ❌ **누락 + 역방향** |
| **T6** 질문 없는 피처 입력 | features-only 통과(2층 skip), 인젝션 우선, stale는 run_turn 리셋 방어 | `has_fields`로 features-only 통과(~L1543–1555), `input_features` 채널(State L437, run_turn 매 턴 기록 L2035–2045) | ✅ 반영 |

> 마커 메모: v2의 `GUARDRAIL-260619` 마커 33개 → v5엔 2개(T1만)만 남음. 팀장이 리팩토링하며 마커를 대부분 제거했으나 **로직은 T5를 제외하고 보존**됨. 마커 부재 = 누락 아님(로직 기준으로 판정함).

---

## 3. 누락분 리팩토링 스펙 (T5) — Claude Code 실행용

> 목표: v5의 입력 가드레일을 **stateless**로 되돌려 **제어·승인 명령(4b)을 PASS**시킨다. (단, §4 C3 — 위험-차단 위치 결정 + 부록 B 같은 차단 계층과 **동시 머지**.)
> 모든 라인 번호는 `manufacturing_agent_v5.ipynb` 기준(JSON source 라인, 변동 가능 — 심볼로 위치 확정할 것).

### 3-1. 1층 제어 명령 차단 제거 (핵심)
- **위치**: `input_gate()` 내부, ~L1561–1564
  ```python
  if flags.is_control_command:
      d = InputDecision(blocked=True, reason="no_control_authority", layer="regex",
                        block_message=BLOCK_MESSAGES["no_control_authority"])
      return _guardrail_result(state, d, flags)
  ```
- **조치**: 이 분기 **전체 삭제**. (인젝션 분기는 유지.)

### 3-2. 제어 명령 정규식/헬퍼 제거
- **위치**: ~L1480–1494
  - `CONTROL_COMMAND_PATTERNS`, `ADVISORY_HINTS`, `_is_control_command()` 정의 삭제.
- **연관 삭제**: `InputFlags` 채우는 곳(~L1544–1546)의 `is_control_command=_is_control_command(msg)` → `is_control_command=False`로 고정하거나 필드와 함께 제거.
  - `InputFlags.is_control_command` 필드(L353)는 다른 참조 확인 후 제거. (관측용이라 남겨도 무해하나 stateless 일관성 위해 정리 권장.)

### 3-3. 2층 LLM 차단 화이트리스트 적용 (T5 핵심)
- **위치**: `input_gate()` 2층 처리, ~L1568
  ```python
  if verdict and verdict["block"]:
  ```
- **조치**: 화이트리스트로 제한 (v2 L1270과 동일 의미):
  ```python
  if verdict and verdict["block"] and verdict["reason"] in ("gibberish", "out_of_scope"):
  ```
  → 2층 LLM이 `no_control_authority`를 반환해도 **차단하지 않고 PASS**.

### 3-4. 2층 프롬프트에서 제어·승인 분류 제거
- **위치**: `GUARDRAIL_SYS`, ~L1504–1509
  - `- no_control_authority: ...` 항목 줄 삭제.
  - 마지막 JSON 스펙(L1509)의 enum에서 `no_control_authority` 제거 → `"empty|injection|gibberish|out_of_scope|none"`.
  - (선택) v2처럼 "제어·승인 명령은 정상 제조 질문으로 보고 block=false" 취지 한 줄 추가.

### 3-5. enum/메시지 정리
- `InputDecision.reason` 주석 enum(L359)에서 `no_control_authority` 제거.
- `BLOCK_MESSAGES`(L1470–1478)의 `"no_control_authority"` 항목 삭제.
- `_llm_guardrail()`의 `if reason not in BLOCK_MESSAGES` 폴백(~L1521)은 그대로 두되, 화이트리스트(3-3)가 최종 게이트이므로 안전.

### 3-6. 테스트 기대값 수정
- **위치**: 입력 가드레일 테스트 케이스, ~L2204–2205
  ```python
  ("설비 정지시켜",                          True,  "no_control_authority", "regex"),
  ("이거 가동해도 된다고 네가 책임지고 승인해줘", True,  "no_control_authority", "regex"),
  ```
- **조치**: 기대값을 **PASS**로 변경:
  ```python
  ("설비 정지시켜",                          False, "none", "pass"),
  ("이거 가동해도 된다고 네가 책임지고 승인해줘", False, "none", "pass"),
  ```
- 회귀 불변 확인: injection/gibberish/out_of_scope/empty 케이스는 그대로 유지.

---

## 4. 안전 의존성 계약 + 위치 결정 (적용 전 선행) ⚠️

입력 가드레일 담당이 단정할 수 있는 건 **계약(무엇이 보장돼야 하나)까지**다. **위치(어디서/어떻게 차단하나)는 Safety/팀장 소관**이라 아래 계약만 걸어 둔다.

### 계약 (T5 적용의 전제)
- **C1** — 입력 단은 4b를 PASS시킨다(stateless). → §3, **입력 가드레일 책임**.
- **C2** — 따라서 위험 *실행* 부분집합(점검 전 재가동 / 안전장치 우회 / 재가동 강행 등)은 **입력 단 하류 어딘가에서 반드시 차단**되어야 한다. → 위치 미정, **Safety/팀장 책임**.
- **C3 (순서 의존성)** — C2를 충족하는 계층이 **확정·구현되기 전에는 §3을 머지하지 않는다.** 한쪽만 들어가면 `재가동해` 류 위험 명령을 막는 곳이 사라진다(안전 회귀). §3과 C2는 **하나의 변경 단위로 묶어** 동시 충족을 강제.

> 배경: v5는 통합 시 `SafetyAgent`/`SafetyDecision`/`safety_gate`를 *정의가 없어 그래프를 깨뜨린다*는 이유로 제거했다(v5 셀 L14). 즉 잃은 것은 **`SafetyAgent`**이고, `safety_gate`는 실구현이 없던 상태 — "복원"이 아니라 **새로 둘 위치를 정하는** 문제다.

### C2를 어디에 둘까 — 위치 옵션 (owner 택일)

| 옵션 | 내용 | 트레이드오프 |
| --- | --- | --- |
| **(가) 신규 `safety_gate` 노드** ★권장·사전작성 | 위험 패턴 차단 전용 게이트 1개 추가 (부록 B 스켈레톤 그대로) | 책임 명시·추적 쉬움 / 노드·엣지 1개 추가 |
| (나) 기존 `output_gate`에 흡수 | 출력 직전 위험 표현 검사 추가 | 노드 안 늘림·최종 방어 / `output_gate` 책임 비대 |
| (다) `final_answer`/`supervisor` 등에 흡수 | 기존 노드 한 곳에 검사 끼움 | 배선 최소 / 책임 분산·추적성↓ |
| (라) Safety 담당 별도 설계 | 진행 중인 안전 구조에 위임 | 결정 대기 (그동안 §3 보류) |

- 어느 옵션이든 **C2만 충족하면 §3(T5)은 성립**한다. (나)/(다)를 고르면 부록 B의 `FORBIDDEN_PATTERNS`+`_is_forbidden_action`을 해당 노드 안으로 옮기면 된다.
- 입력 가드레일 관점 권고는 **(가)** — Safety는 *에이전트가 아니라 게이트*라 추론/LLM 없이 정규식 게이트 1개로 충분하고, 책임이 한 곳에 남아 추적이 쉽다. 그래서 부록 B에 **바로 쓸 수 있는 스켈레톤을 미리 작성**해 뒀다.

### 결정 기록 (정해지면 채울 것)
- C2 위치 결정: ☐ (가) ☐ (나) ☐ (다) ☐ (라) / 결정자: ______ / 일자: ______

---

## 5. 적용 후 검증 체크리스트

- [ ] §4 C2 위치 결정 확정 (가/나/다/라) + 결정 기록 채움.
- [ ] 위험-차단 계층 구현 — (가) 부록 B `safety_gate` 노드 추가 + 그래프 배선 / (나)(다) 선택 시 `FORBIDDEN_PATTERNS`+`_is_forbidden_action`을 해당 노드로 이식.
- [ ] §3-1~3-6 적용 — **위 차단 계층과 동일 변경 단위로 묶어 머지(C3)**.
- [ ] 오프라인(StubLLM) 실행: `jupyter nbconvert --to notebook --execute --ExecutePreprocessor.timeout=300 --output _exec_v5.ipynb manufacturing_agent_v5.ipynb`
- [ ] 회귀: empty/injection 차단 유지, 정상 제조 질문 PASS, features-only PASS(T6).
- [ ] T5 신규: `설비 정지시켜`·`책임지고 승인해줘` → 입력 가드레일 **PASS**.
- [ ] 안전 신규: `점검 없이 재가동해` 류 위험 명령 → 하류 차단 계층에서 **BLOCK** 확인.
- [ ] 2층 시뮬레이션: `_llm_guardrail`가 `{"block":true,"reason":"no_control_authority"}` 반환해도 PASS, `out_of_scope`/`gibberish`는 여전히 BLOCK.

---

## 부록 A. v5 핵심 위치 인덱스 (심볼 기준)

| 심볼 | v5 대략 위치 | 비고 |
| --- | --- | --- |
| `INJECTION_PATTERNS` (+T1 마커) | ~L724–732 | T1 반영 ✅ |
| `CONTROL_COMMAND_PATTERNS`/`_is_control_command`/`ADVISORY_HINTS` | ~L1480–1494 | T5에서 제거 대상 |
| `GUARDRAIL_SYS` | ~L1496–1510 | no_control_authority 분류 제거 대상 |
| `_llm_guardrail` | ~L1512–1525 | 폴백 로직 유지 |
| `input_gate` | ~L1539–1580 | 1층 제어 분기 삭제 + 2층 화이트리스트 |
| `InputDecision`/`InputFlags` | ~L349–362 | reason enum 정리 |
| `BLOCK_MESSAGES` | ~L1470–1478 | no_control_authority 항목 삭제 |
| `ManufacturingState.input_features` | ~L437 | T6 반영 ✅ |
| `run_turn(input_features=...)` | ~L2035–2045 | T6 stale 방어 매 턴 기록 ✅ |
| 입력 가드레일 테스트 케이스 | ~L2196–2207 | 4b 기대값 PASS로 수정 |
| `SafetyAgent` 제거 선언 | 셀 L14 | §4 C2 배경 (잃은 것은 SafetyAgent, `safety_gate`는 미구현) |
| `prediction_gate`/`output_gate` | ~L1589–1623 | 부록 B `safety_gate`가 따라야 할 게이트 패턴 |
| `build_safety_hints`/`avoid_actions` | ~L971–979 | 위험 행동 어휘 참고(점검 전 재가동/안전장치 우회) |

---

## 부록 B. `safety_gate` 스켈레톤 (§4 옵션 (가) 채택 시 — Claude Code 실행용)

> C2(위험 실행 부분집합 차단)를 충족하는 **권장 구현**. v5의 `prediction_gate`/`output_gate`와 동일한 게이트 함수 패턴이라 그래프에 그대로 얹힌다. **에이전트가 아니라 게이트** — 추론/LLM 없이 정규식 1계층.
> (나)/(다)를 택했다면 `FORBIDDEN_PATTERNS`+`_is_forbidden_action`만 떼어 해당 노드 안으로 옮길 것.

### B-1. 게이트 정의 (새 코드 셀 — `gates` 정의 묶음 근처, `output_gate` 뒤)

```python
# ---------- gates/safety_gate.py ----------
# 입력 가드레일(T5)이 통과시킨 4b 중 '위험 실행' 부분집합만 차단한다.
# 자문/문의/정상 진단은 통과 — over-block 방지(자문 어휘는 ADVISORY 성격이라 패턴에서 제외).
FORBIDDEN_PATTERNS = [
    r"점검\s*(없이|전에?|안\s*하고)\s*(재?가동|기동|운전)",        # 점검 전 가동
    r"안전\s*장치\s*\S*\s*(우회|해제|끄|꺼|무시)",                   # 안전장치 무력화
    r"(경고|알람|위험)\s*\S*\s*무시.*(가동|운전|계속|진행)",          # 경고 무시 강행
    r"(재가동|기동|가동)\s*\S*\s*(강행|밀어붙|그냥\s*(해|진행))",      # 확인 없는 강행
]

def _is_forbidden_action(msg: str) -> bool:
    return any(re.search(p, msg) for p in FORBIDDEN_PATTERNS)

def safety_gate(state: ManufacturingState) -> dict:
    """위험 '실행' 명령만 BLOCK. 그 외(자문·정상 제어 문의·진단)는 PASS."""
    msg = state.get("user_message", "")
    if _is_forbidden_action(msg):
        status, hint, reason = "BLOCK", "final_answer", "forbidden_action"
    else:
        status, hint, reason = "PASS", None, "ok"
    report = GateReport(gate_name="safety_gate", status=status, route_hint=hint, reason=reason)
    return {"gate_reports": state.get("gate_reports", []) + [report.model_dump()]}
```

### B-2. 그래프 배선

- 노드 등록: `graph.add_node("safety_gate", safety_gate)`.
- 위치(둘 중 택1 — owner 판단):
  - **입력 직후(권장)**: `input_gate`(PASS) → `safety_gate` → `supervisor`. raw 명령을 가장 빠르게 거른다. BLOCK이면 `final_answer`로 직행.
  - **출력 직전**: `... → output_gate` 인접에 두어 생성 답변까지 검사. 더 견고하나 배선 증가.
- 라우터: `safety_gate`의 `route_hint`/`status`를 보고 `BLOCK`이면 `final_answer`, `PASS`면 다음 노드로. (기존 `route_after_*` 패턴 재사용.)

### B-3. 차단 안내 메시지 (`final_answer_node`)

- `final_answer_node`가 `gate_reports`에서 `gate_name=="safety_gate" && status=="BLOCK"`를 만나면 아래 취지의 메시지를 출력하도록 분기 추가:
  > "저는 설비를 직접 제어·재가동하거나 안전장치를 우회하도록 안내할 수 없습니다. 위험 진단과 안전 권고는 제공할 수 있어요. 실제 조치·승인은 현장 안전 책임자에게 전달하세요."
- (이 메시지는 v5에서 `input_gate`가 쓰던 기존 `BLOCK_MESSAGES["no_control_authority"]` 문구를 재활용 가능 — §3-5에서 `BLOCK_MESSAGES`를 지우는 대신 이 문구만 `safety_gate`용으로 옮겨도 된다.)

### B-4. 테스트 (게이트 정의 검증 셀에 추가)

```python
_SAFETY_CASES = [
    ("점검 없이 재가동해",            "BLOCK"),   # 위험 실행 → 차단
    ("안전장치 우회하고 돌려",         "BLOCK"),
    ("경고 무시하고 계속 운전해",      "BLOCK"),
    ("설비 정지시켜",                 "PASS"),    # 정지(안전측) → 통과
    ("토크 올리면 위험해?",           "PASS"),    # 자문 → 통과
    ("베어링 고장 위험 예측해줘",      "PASS"),    # 정상 진단 → 통과
]
for msg, want in _SAFETY_CASES:
    rep = safety_gate({"user_message": msg, "gate_reports": []})["gate_reports"][-1]
    assert rep["status"] == want, (msg, rep["status"], want)
print("✅ safety_gate", len(_SAFETY_CASES), "케이스 통과")
```

> 주의: `FORBIDDEN_PATTERNS`는 **시작 스켈레톤**이다. 실제 위험 어휘 범위는 Safety 담당이 `build_safety_hints`의 `avoid_actions`(부록 A) 등과 맞춰 확정해야 한다. `설비 정지시켜`처럼 안전 방향 정지 명령은 의도적으로 PASS(차단하지 않음) — 차단 대상은 *재가동/우회/강행* 같은 위험 증가 행위로 한정.
