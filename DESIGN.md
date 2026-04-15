# Multi-Agent Coordination MCP — 설계 문서 v0.1

---

## 1. 개요

### 목적
두 개 이상의 AI 에이전트가 동일한 프로젝트를 자율적으로 협업할 수 있도록 중재하는 MCP.
사람은 초기 설정과 교착 상황에서만 개입한다. 그 외 모든 턴은 에이전트들이 MCP 버스를 통해 자율 진행한다.

### 핵심 전제
- 에이전트들은 직접 통신이 불가능하다. 이 MCP가 유일한 통신 버스다.
- 에이전트가 하나라도 미연결이면 세션은 시작되지 않는다.
- 에이전트 종류는 사용자가 자유롭게 지정한다 (Claude Code, Codex, Gemini CLI 등 MCP를 지원하는 모든 에이전트).

### Memento와의 관계
```
Memento MCP                      Coordinator MCP (이 문서)
──────────────────────────────────────────────────────────
장기 기억층                       단기 작업층
세션 종료 후에도 유지              작업 완료 후 소멸 가능
프로젝트 규칙, 패턴, 결론          현재 턴, 쟁점, 요약, 락
다음 세션에서 컨텍스트 복원용      지금 이 작업의 실시간 버스
        ↑
        결론층만 올라감 (중간 반박, 미완성 판단 저장 금지)
```

---

## 2. 시스템 아키텍처

### 2.1 3층 메모리 구조
```
┌─────────────────────────────────────┐
│  Layer 3: Memento MCP (장기 기억)    │  ← 최종 합의, 검증된 결론, 반복 패턴
├─────────────────────────────────────┤
│  Layer 2: Task State (작업 상태)     │  ← 현재 task, 턴 번호, 역할, 진행 단계
├─────────────────────────────────────┤
│  Layer 1: Message Bus (즉시 버스)    │  ← 구현 요약, 판정, 반박, 쟁점 (TTL)
└─────────────────────────────────────┘
```

### 2.2 에이전트 연결 구조
```
[사용자]
   │ 초기 설정 (모드, 역할 배정, 최대 턴)
   ▼
[Coordinator MCP]
   ├── read/write ──→ [Agent A: Implementer 역할]
   └── read/write ──→ [Agent B: Verifier 역할]
                           (토론 모드에서는 A=Proposer, B=Critic)
```

### 2.3 에이전트 선택 메커니즘
세션 시작 시 사용자가 다음을 지정한다:
```json
{
  "session_config": {
    "mode": "PROJECT | SINGLE | DISCUSSION",
    "scope": "PROJECT 모드에서는 전체 프로젝트 완료까지",
    "agents": {
      "implementer": "claude-code",
      "verifier": "codex"
    },
    "max_negotiation_turns": 3,
    "project_brief": "프로젝트 목적과 핵심 방향성 (필수)",
    "task_scope": "이번 작업 범위 (파일, 모듈 단위로 명시)"
  }
}
```

Coordinator가 세션 시작 직후 수행하는 첫 동작:
1. agents에 명시된 모든 에이전트가 MCP에 연결되어 응답하는지 확인
2. 하나라도 미연결 → 즉시 중단 및 사용자 보고
3. 모두 연결 확인 → session_id 발급 → 상태 INITIALIZING → 해당 모드 진입

---

## 3. 모드 정의

### 3.1 Mode 1: 프로젝트 완주 모드 (PROJECT)

**목적**: 프로젝트 전체가 완료될 때까지 구현-검증 루프를 자율 반복한다.

**특징**:
- 초기 방향성과 설계 품질이 결과물의 전부를 좌우한다
- 사람 개입 없이 끝까지 밀고 나간다
- 장기 실행이므로 중단 방지 메커니즘이 필수

**흐름**:
```
[사용자: 프로젝트 방향성 + task_scope 지정]
        ↓
[Implementer: 작업 수행]
        ↓
[Implementer → Coordinator: 구현 요약 전달]
        ↓
[Verifier: 보수적 검증 + 피드백]
        ↓
[협의 루프 시작]
  Implementer: 피드백 수용 or 반박
  Verifier: 반박에 대한 재검토
  ... 사용자 지정 최대 턴까지 ...
  최대 턴 초과 → 강제 타협안 1턴 추가 → 양쪽 서명
        ↓
[협의 완료 → Implementer 다음 작업 시작]
        ↓
[반복 — 프로젝트 완료 선언까지]
        ↓
[Coordinator: 전체 요약 → Memento 저장 → 사용자 보고]
```

**프로젝트 완료 판정**:
- Coordinator가 project_brief 기준으로 판정
- 또는 Verifier가 "이 task가 프로젝트의 마지막 task"를 PASS로 통과할 때
- 사용자가 명시적으로 "프로젝트 완료" 선언

### 3.2 Mode 2: 단건 이행 모드 (SINGLE)

**목적**: 사용자의 지시 1건만 이행하고 종료한다.

**흐름**: Mode 1과 동일하되, DONE 후 루프 없이 즉시 세션 종료.

```
[사용자: 단건 지시]
        ↓
[구현 → 검증 → 협의 → 구현 → ... → PASS]
        ↓
[DONE: 결과 요약 → 사용자 보고 → 세션 종료]
```

### 3.3 Mode 3: 토론 모드 (DISCUSSION)

**목적**: 구현 없이 방향성과 설계 문서를 산출한다.
주로 프로젝트 초기 설계 또는 풀리지 않는 문제의 방향 재정립에 사용.

**흐름**:
```
[Proposer: 초안 작성]
        ↓
[Critic: 반박/보완]
        ↓
[Proposer: 변경분만 업데이트]
        ↓
[최대 3턴 반복]
        ↓
[합의 도달] → 설계 문서 산출 → "구현으로 전환하시겠습니까?" → 사용자 확인
[합의 실패] → 1턴 추가 타협안 작성 → 사용자 보고
```

**출력물**: 설계 문서 (합의안 or 타협안)
**전환**: 사용자 확인 후 Mode 1 또는 Mode 2로 자동 전환 가능

---

## 4. 턴 구조 및 주입 규칙

### 4.1 분업 모드 턴 단계
```
[Turn N]
  Phase A: 구현
    → Coordinator가 Implementer에게 주입:
       - 현재 task_scope
       - project_brief (매 턴 고정 주입)
       - 이전 협의에서 결정된 수정 사항
       - 미해결 의무 목록 (Verifier가 누적 추적한 항목)
       - HARD RULE 리마인더 (매 턴 고정 주입)

  Phase B: 구현 요약 전달
    → Implementer → Coordinator → Verifier

  Phase C: 검증
    → Coordinator가 Verifier에게 주입:
       - 구현 요약
       - 누적 미해결 의무 목록
       - project_brief (매 턴 고정 주입)
       - HARD RULE 리마인더 (매 턴 고정 주입)
       - 이전 FIX_REQUIRED 이력 (이슈 ID 포함)

  Phase D: 협의 (피드백 ↔ 반박)
    → Verifier 판정이 FIX_REQUIRED일 경우 진입
    → Implementer가 반박 or 수용 의사 표명
    → 최대 max_negotiation_turns턴
    → 초과 시 타협안 1턴 강제 추가

  Phase E: 합의 완료
    → Coordinator: 합의 내용 기록 → 다음 턴 Phase A 시작
```

### 4.2 턴별 고정 주입 항목

**Implementer에게 매 턴 반드시 주입:**
```
[PROJECT BRIEF]
{project_brief 전문}

[CURRENT SCOPE]
작업 범위: {task_scope}
이 범위 외의 파일/코드는 절대 수정 금지

[OPEN OBLIGATIONS]
이전 턴에서 동의했으나 미이행된 항목:
{open_obligations 목록}
이 항목들은 이번 턴에서 반드시 처리 또는 명시적 반박

[HARD RULES — 위반 시 즉시 세션 중단]
(Section 6 전문 주입)
```

**Verifier에게 매 턴 반드시 주입:**
```
[PROJECT BRIEF]
{project_brief 전문}

[CUMULATIVE OBLIGATIONS]
구현 모델이 지금까지 동의/약속한 미이행 항목:
{cumulative_obligations 목록 (이슈 ID 포함)}

[PREVIOUS FIX_REQUIRED HISTORY]
{이전 판정 이력 — 이슈 ID, 내용, 반영 여부}

[HARD RULES — 위반 시 즉시 세션 중단]
(Section 7 전문 주입)
```

---

## 5. Implementer 기본 원칙 (HARD RULES)

> 이 원칙들은 매 구현 턴 시작 시 Coordinator가 Implementer에게 전문 주입한다.
> 위반이 감지되면 Coordinator가 해당 턴을 무효화하고 사용자에게 보고한다.

### RULE 1: 일회성 땜빵 완료 선언 금지 [HARD]
다음 행동은 완료가 아니다:
- 특정 예시 1개 통과 후 "해결됐습니다" / "완료됐습니다" 선언
- 구체어·패턴·예외 1개 추가 후 "감지 개선 완료" / "수정 완료" 표현
- 테스트 케이스 N개 통과 = 전체 클래스 해결처럼 포장
- 일회성 땜빵(구체어 추가, 예외 처리, if-else 분기)을 구조적 해결로 표현
- "이 표현만 막으면 된다"는 전제로 시스템 전반에 영향을 주는 수정 진행

완료 선언 조건: Verifier로부터 PASS 판정을 수신한 시점에만 완료다.

### RULE 2: 프로젝트 방향성 위반 금지 [HARD]
- 매 턴 주입되는 project_brief를 기준으로 구현 방향을 확인한다
- project_brief와 상충하는 구현을 발견하면 구현을 중단하고 Coordinator에게 보고한다
- 방향성 위반 여부가 불확실한 경우에도 구현 전에 Coordinator에게 확인 요청한다

### RULE 3: 확인 없이 구현 시작 금지 [HARD]
- Coordinator의 명시적 IMPLEMENTING 상태 전환 신호 없이 코드 수정을 시작하지 않는다
- 협의 중(Phase D)에는 어떤 코드도 수정하지 않는다
- 합의 완료 신호 수신 후에만 Phase A(구현)를 시작한다

### RULE 4: 검증 모델 전달 요약 누락 금지 [HARD]
구현 완료 후 반드시 아래 형식의 요약을 Coordinator에 제출한다.
요약 없이 검증 단계로 진입하지 않는다.
최종 구현 완료(프로젝트 종료) 직전 턴에는 이 의무가 면제된다.

```json
{
  "turn": "<번호>",
  "role": "implementer",
  "task_summary": "<무엇을 구현했는가 — 1~3문장>",
  "changes": [
    {
      "file": "<파일명>",
      "type": "add | modify | delete",
      "description": "<변경 내용 구체적으로>"
    }
  ],
  "obligation_closures": [
    {
      "obligation_id": "<이슈 ID>",
      "status": "resolved | partially_resolved | deferred",
      "evidence": "<어떻게 해결했는가>"
    }
  ],
  "verification_focus": ["<Verifier가 반드시 확인해야 할 항목>"],
  "side_effects": ["<작업 범위 외에서 발견한 문제 — 수정하지 않음, 기록만>"],
  "self_check": {
    "runs_without_error": true,
    "matches_spec": true,
    "notes": "<자가 점검 메모>"
  }
}
```

### RULE 5: 동의 후 완료 환각 금지 [HARD]
"동의한다", "반영하겠다", "다음 턴에 처리하겠다"는 완료가 아니다.
동의 후 즉시 다음 단계로 진입하면 그 항목은 망각된다.
동의한 항목은 obligation_closures에 반드시 ID로 등록하고 다음 구현 턴에서 처리한다.

### RULE 6: 지정 범위 외 코드 수정 금지 [HARD]
- task_scope에 명시된 파일/모듈 외의 코드를 수정하지 않는다
- 범위 밖의 문제를 발견하면 수정하지 않고 side_effects에 기록한다
- 범위 확장이 필요하다고 판단되면 Coordinator에게 요청하고 사용자 승인 후에만 범위를 변경한다

---

## 6. Verifier 기본 원칙 (HARD RULES)

> 이 원칙들은 매 검증 턴 시작 시 Coordinator가 Verifier에게 전문 주입한다.

### RULE 1: 누적 약속 이행 검증 [HARD]
현재 턴 커밋만 보고 판정하지 않는다. 반드시 아래 3단계 순서로 진행한다:

1. **미해결 의무 재열거**: 이전 턴까지 Implementer가 동의/약속한 항목 중 아직 닫히지 않은 것
2. **이번 커밋이 닫은 의무**: obligation_closures + 실제 변경 파일 기준으로만 인정
3. **아직 열린 의무**: 이번 턴 이후에도 남아있는 항목 → 다음 턴 previous_unresolved에 승계

이 3단계를 거치지 않으면 PASS 판정을 내릴 수 없다.
Implementer가 해당 약속을 이번 턴 요약에 언급하지 않아도, Verifier는 실제 이행 여부를 독립적으로 확인한다.

### RULE 2: 코드 먼저, 판정 나중 [HARD]
- implementation_summary 요약만 읽고 판정하지 않는다
- 반드시 실제 변경된 파일을 확인한 후 판정한다
- 확인 없이 추상적 판단("잘 된 것 같습니다")을 내리지 않는다
- 개념적 의견과 코드 기반 검증을 명확히 구분한다

### RULE 3: 쓰기 금지
- 코드를 직접 수정하지 않는다
- read + critique만 허용된다
- 수정 제안은 required_fix 항목으로 제시하되 직접 구현하지 않는다

### RULE 4: 대안 경로 검토 의무
- Implementer의 결론을 채점하듯 받지 않는다
- 제시된 원인 외 대안 경로, 더 작은 수정 지점, 새로 생길 부작용까지 확인한다
- 새 설계 제안이 나오면 현재 구현된 입력원/주입 순서/우선순위 구조를 먼저 대조한다
- 이미 있는 기능을 무시한 채 추상 설계만 덧씌우지 않는다

### RULE 5: 판정 형식 고정 [HARD]
감상문, 권장사항 나열, "더 고려해보세요" 형태의 출력을 금지한다.
반드시 아래 형식으로만 출력한다:

```json
{
  "turn": "<번호>",
  "role": "verifier",
  "verdict": "PASS | FIX_REQUIRED | BLOCKER",
  "confidence": 0.0,
  "obligations_checked": [
    {
      "obligation_id": "<이슈 ID>",
      "status": "resolved | unresolved | partially_resolved",
      "evidence": "<실제 코드에서 확인한 내용>"
    }
  ],
  "accepted_points": ["<정상 동작 확인된 항목>"],
  "issues": [
    {
      "id": "<V-001 형식>",
      "severity": "high | medium | low",
      "blocking": true,
      "category": "logic | edge_case | performance | security | spec_mismatch | scope_violation",
      "reason": "<무엇이 왜 문제인가>",
      "required_fix": "<구체적으로 무엇을 어떻게 수정해야 하는가>",
      "alternative_considered": "<검토한 대안 경로>"
    }
  ],
  "previous_unresolved": ["<이전 턴에서 미해결된 이슈 ID>"],
  "retest_focus": ["<다음 턴에서 반드시 재확인할 항목>"]
}
```

판정 기준:
- **PASS**: 모든 high severity blocking 항목 해결 + 기능이 project_brief 기준 명세대로 동작
- **FIX_REQUIRED**: 수정 가능한 문제 존재, 재구현 후 재검증 필요
- **BLOCKER**: 현재 방향으로 진행 불가, Coordinator에게 ESCALATED 요청

---

## 7. 협의 루프 (Phase D) 규칙

### 흐름
```
Verifier: FIX_REQUIRED 판정 발행
        ↓
Implementer: 각 이슈에 대해 수용(ACCEPT) or 반박(REBUT)
        ↓
Verifier: 반박에 대한 재검토
        ↓
... 최대 max_negotiation_turns 턴 ...
        ↓
초과 시: 타협안 1턴 강제 추가
        (양쪽이 각자 양보 가능한 최소 합의안 작성)
        ↓
합의 완료: Coordinator가 agreed_actions 목록 기록
        ↓
다음 구현 Phase A 시작
```

### 반박 형식
```json
{
  "turn": "<번호>",
  "role": "implementer",
  "rebuttal": [
    {
      "issue_id": "V-001",
      "action": "ACCEPT | REBUT | DEFER",
      "reason": "<수용 이유 or 반박 근거>",
      "proposed_alternative": "<반박 시 대안 제시>",
      "timeline": "this_turn | next_turn | out_of_scope"
    }
  ]
}
```

### 타협안 형식 (강제 추가 턴)
```json
{
  "turn": "<번호>",
  "type": "forced_compromise",
  "agreed_actions": [
    {
      "issue_id": "V-001",
      "resolution": "<최종 합의된 처리 방식>",
      "signed_by": ["implementer", "verifier"]
    }
  ],
  "deferred_issues": [
    {
      "issue_id": "V-002",
      "reason": "<연기 이유>",
      "target_turn": "<처리 예정 턴>"
    }
  ]
}
```

---

## 8. 상태 머신

```
INITIALIZING
  → 에이전트 연결 확인 중
  → 실패: AGENT_UNAVAILABLE (세션 종료)
  → 성공: 모드에 따라 분기

[DISCUSSION 모드]
DISCUSSING
  → 합의 도달: CONSENSUS_REACHED → 사용자 확인 → READY_TO_IMPLEMENT
  → 타협 도달: COMPROMISE_REACHED → 사용자 확인 → READY_TO_IMPLEMENT
  → 합의 실패: ESCALATED (사용자 보고)

READY_TO_IMPLEMENT
  → 사용자 확인 (Mode 1/2 지정)
  → IMPLEMENTING

[DIVISION 모드]
IMPLEMENTING
  → 구현 요약 전달 완료: VERIFYING

VERIFYING
  → PASS: DONE (Mode 2) or NEXT_TASK (Mode 1)
  → FIX_REQUIRED: NEGOTIATING
  → BLOCKER: ESCALATED

NEGOTIATING
  → 합의 완료: IMPLEMENTING
  → 협의 실패(타협 후): IMPLEMENTING (타협안 기반)
  → 교착(아래 조건): ESCALATED

NEXT_TASK (Mode 1 전용)
  → 프로젝트 완료 선언: DONE
  → 다음 task 있음: IMPLEMENTING

DONE
  → Memento 저장 → 세션 종료 → 사용자 보고

ESCALATED
  → 사용자에게 상황 보고 → 사용자 지시 대기
  → 사용자 지시에 따라 어느 상태로든 재진입 가능
```

---

## 9. 교착 탈출 메커니즘

### 트리거 조건 (하나라도 충족 시 ESCALATED)
1. 동일 이슈 ID가 `previous_unresolved`에 **3턴 연속** 등장
2. Verifier가 `BLOCKER` 판정 발행
3. 협의 루프에서 `forced_compromise` 후에도 `signed_by` 양쪽 서명 불가
4. confidence < 0.4 인 PASS 판정 (신뢰도 부족 경고)

### ESCALATED 보고 형식
```
[교착 상태 감지]
원인: {trigger 조건}
현재 상태: {마지막 구현 요약}
미해결 이슈: {이슈 ID + 내용 목록}
진행 턴 수: {현재까지 총 턴}
권장 행동: {3옵션으로 방향 재정립 | 수동 중재 | 세션 종료}
```

---

## 10. 장기 프로젝트 중단 방지 (Mode 1 전용)

### 문제
Mode 1은 장시간 실행된다. 에이전트 세션 만료, 컨텍스트 한도 초과, 네트워크 오류 등으로 중단될 수 있다.

### 대응 전략

**체크포인트 자동 저장**
- 매 PASS 판정 후 Coordinator가 자동으로 체크포인트 저장
- 저장 내용: 완료된 task 목록, 현재 project_brief, 미해결 의무 목록, 마지막 구현 요약
- 세션 재시작 시 마지막 체크포인트에서 자동 복원

**컨텍스트 압축**
- 누적 턴이 임계값(예: 20턴) 초과 시 Coordinator가 자동 압축 실행
- 압축 대상: 오래된 협의 내용, 해결된 이슈 상세 내용
- 보존 대상: 미해결 의무 목록, project_brief, 마지막 3턴 전문

**Memento 중간 동기화**
- 매 N턴(사용자 지정, 기본 5턴)마다 중간 결론을 Memento에 저장
- 저장 내용: 확정된 구현 패턴, 반복된 오류 유형, 검증 기준
- 세션 재시작 시 Memento 컨텍스트로 맥락 복원

**에이전트 재연결 처리**
- 에이전트가 응답 없으면 Coordinator가 N초 후 재시도 (최대 3회)
- 3회 실패 시 PAUSED 상태로 전환 + 사용자 알림
- PAUSED 상태에서 에이전트 재연결 시 마지막 체크포인트에서 재개

---

## 11. 토론 모드 (Mode 3) 턴 구조

### 턴 형식 — Proposer
```json
{
  "turn": "<번호>",
  "role": "proposer",
  "position_summary": "<현재 입장 한 줄 요약>",
  "changes_this_turn": ["<이번 턴에서 수정한 점만 — 전체 재작성 금지>"],
  "unresolved": ["<아직 합의 안 된 쟁점>"],
  "questions_for_critic": ["<질문 1>", "<질문 2>"],
  "confidence": 0.0
}
```

### 턴 형식 — Critic
```json
{
  "turn": "<번호>",
  "role": "critic",
  "agrees": ["<동의하는 부분>"],
  "disagrees": [
    {
      "point": "<비동의 항목>",
      "reason": "<왜 문제인가>",
      "severity": "high | medium | low",
      "minimum_fix": "<최소 수정 제안 — 구체적으로>"
    }
  ],
  "open_questions": ["<쟁점>"],
  "consensus_ready": false,
  "confidence": 0.0
}
```

### 토론 종료 조건
**정상 합의**:
- proposer.confidence >= 0.85
- critic.consensus_ready == true
- 양쪽 unresolved / open_questions 모두 비어 있음

**타협 종료**:
- 최대 3턴 후 합의 미달 → 1턴 추가 (타협안 작성 턴)
- 타협안 턴에서도 미합의 → ESCALATED

**산출물**:
```json
{
  "type": "design_document | compromise_document",
  "summary": "<합의/타협된 내용 전문>",
  "open_issues": ["<미합의로 남은 항목>"],
  "recommended_next": "MODE_1 | MODE_2 | USER_DECISION"
}
```

---

## 12. Memento 연동 규칙

### 저장 대상 (세션 종료 또는 중간 동기화 시)
| 항목 | 저장 조건 |
|------|-----------|
| 최종 합의안 / 타협안 | 토론 모드 정상 종료 시 |
| 검증 통과한 구현 요약 | PASS 판정 직후 |
| 반복 발생한 이슈 패턴 | 동일 category 이슈 3회 이상 |
| 프로젝트 방향성 결정 | project_brief 변경 시 |
| 검증 기준 합의 | Verifier/Implementer 간 합의된 판정 기준 |

### 저장 금지 항목
- 중간 반박문
- 협의 중 일시적 가설
- 미통과(FIX_REQUIRED) 구현 내용
- 감정적 critique
- 타협안 작성 과정의 중간 초안

---

## 13. Coordinator 동작 요약

### 세션 시작
1. 에이전트 연결 확인 → 실패 시 즉시 중단
2. session_id 발급
3. 각 에이전트에 역할 + 초기 주입 전송
4. 상태: INITIALIZING → 모드별 첫 상태

### 매 턴
1. 현재 상태 확인
2. 해당 상태의 담당 에이전트에게 주입 컨텍스트 구성
   - project_brief (고정)
   - HARD RULES (고정)
   - 누적 의무 목록 (동적)
   - 이전 판정 이력 (동적)
3. 에이전트 응답 수신 → 형식 검증
4. 형식 불일치 시 재요청 (최대 2회) → 실패 시 ESCALATED
5. 상태 전이 판정 → 다음 턴 준비

### 교착 감지
- 매 턴 종료 후 교착 조건 체크
- 조건 충족 시 ESCALATED 전환 + 사용자 보고

### 세션 종료
- DONE: 체크포인트 최종 저장 → Memento 동기화 → 사용자 보고
- ESCALATED: 현황 보고 → PAUSED 대기

---

## 14. 미결 설계 항목 (v0.2에서 결정 필요)

1. **에이전트 등록 방식**: 에이전트가 MCP에 자신의 agent_id를 어떻게 등록하는가
2. **폴링 vs 웹소켓**: 에이전트가 자기 차례를 어떻게 감지하는가
3. **컨텍스트 압축 알고리즘**: 자동 압축 시 무엇을 남기고 무엇을 버리는가
4. **프로젝트 완료 판정 기준**: Coordinator가 project_brief를 어떻게 파싱해서 완료를 판정하는가
5. **SQLite 스키마**: 실제 테이블 정의 및 TTL 구현 방식
6. **MCP 도구 목록**: 에이전트가 호출하는 실제 도구 이름 및 파라미터

---

*v0.1 — 2026-04-15*
*다음 단계: 사용자 검토 → v0.2 (SQLite 스키마 + MCP 도구 목록 + docker-compose)*
