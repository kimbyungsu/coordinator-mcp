# Multi-Agent Coordination MCP — 설계 문서 v0.9

> v0.8 → v0.9 주요 변경
> - Section 0 추가: 설계 철학 — 공통 상황 레이어 (Shared Situation Layer)
>   역할 규칙보다 앞서는 기반 원칙. 독립 에이전트 간 동일 상황 공유가 1번.
>   Shared Situation Object 필드 정의, 역할 렌즈 원칙, 위반 감지 기준 포함.
> - CLAUDE.md: 완료·충분 과신 금지 규칙 추가
>
> v0.7 → v0.8 주요 변경
> - Section 7 RULE 7 추가: Verifier 탐색 범위 3단계 + 탐색 종료 조건 (5개 충족 시 전체 스캔 불필요)
> - Section 15 보강: Verifier→Implementer handoff 대칭 고정 (verdict_id + fix_base_revision, 불변조건 6~9)
>   Coordinator가 양방향 handoff를 독립적으로 snapshot 고정
>
> v0.6.1 → v0.7 주요 변경
> - Section 15: 제출/검증 대상 고정 메커니즘 (submission_id + base_revision + CAS)
> - Section 16: MVP 범위 명시 (v1 = 2-agent 공식 지원, 3-agent 이상 비지원)
> - Section 17: 미결 항목 Batch 1/2/3 재구조화 (기존 Section 15 대체)
>
> v0.6 → v0.6.1 주요 변경
> - Phase A0 active 범위 통일: proposed 제거, accepted/in_progress만 명시
>   (Phase A 시점 proposed 존재 = Coordinator 오류로 정의)
>
> v0.5 → v0.6 주요 변경
> - Phase A0 추가: Implementer 턴 시작 시 상태 정합성 자기점검 (Preflight) [HARD]
>   - open_obligations 재열거, 지난 턴 주장 vs 실제 상태 대조, mismatch 구현 전 명시
> - RULE 4 제출 포맷에 preflight 필드 추가
>
> v0.4.1 → v0.5 주요 변경
> - Verifier HARD RULES에 RULE 6 추가: 과검증 방지 + PASS 허용 의무 + 검증 우선순위
> - Section 8 협의 루프에 재지적 금지 규칙 추가
> - Section 12 토론 모드에 재지적 금지 규칙 추가
> - PROMPT_CONTRACT.md 신규 생성 (주입 원문 분리)
>
> v0.3 → v0.4 주요 변경
> - "active" 집합 개념 정의 추가 (proposed+accepted+in_progress)
> - 8장 협의 종료 조건: high/blocking → resolved/ESCALATED만 (rejected 제거)
> - PASS 판정식: active high/medium/blocking 모두 없어야 함을 명문화
> - verification_cycles 증가 위치 단일화: 5.1은 Section 5.3 참조, 9장은 주석 처리
>
> v0.2 → v0.3 주요 변경
> - severity 정책 통일 (4.2 ↔ 8장 모순 제거): low만 deferred 허용, medium은 해결/rejected만
> - PASS와 deferred low issue 관계 명문화 (task_pass_condition)
> - 복원 패키지 write_permission 동적 계산 명시
> - 복원 패키지 open_obligations / unresolved_issues 구조화 (문자열 배열 → 객체)
> - test_evidence를 선택 → 의무 필드로 강화
> - Coordinator ACK 답변 범위 제한 명문화
> - DISCUSSION 종료 조건에서 confidence 의존도 제거
> - verification cycle 증가 시점 정의
> - Issue draft → canonical ID 변환 시점 명시 (Phase C 종료 후 Phase D 진입 전)
> - draft_id Phase D부터 외부 노출 금지

---

## 0. 설계 철학 — 공통 상황 레이어 (Shared Situation Layer)

> 이 섹션은 모든 역할 규칙(Section 6~14)보다 앞에 온다.
> 아래 원칙을 위반하는 역할 규칙은 이 섹션 기준으로 교정한다.

### 핵심 원칙

이 MCP의 목적은 **성격이 다른 에이전트들을 동일하게 고정된 상황 위에 올려, 각자 독립된 관점으로 부딪히게 만드는 것**이다.

- Implementer / Verifier / Proposer / Critic은 역할 렌즈이지 상하관계가 아니다
- Verifier는 Implementer의 팔다리가 아닌 독립 행위자다
- 어느 에이전트의 자진 신고도 ground truth가 아니다
- Coordinator가 독립적으로 고정한 situation object가 ground truth다

### Shared Situation Object

모든 에이전트는 Coordinator가 고정한 동일한 situation object를 먼저 공유한다.
역할별 규칙은 이 공통 상황을 어떤 관점으로 읽는가의 차이일 뿐이다.

```json
{
  "situation_id": "<고유 ID>",
  "mode": "PROJECT | SINGLE | DISCUSSION",
  "project_brief": "<프로젝트 방향성>",
  "current_task": "<현재 작업 설명>",
  "acceptance_criteria": ["<완료 조건>"],
  "constraints": ["<제약 조건>"],
  "base_revision": "<현재 기준 snapshot>",
  "open_obligations": ["..."],
  "agreed_actions": ["..."],
  "unresolved_points": ["..."],
  "budget_state": {},
  "next_required_action": "<다음 행동>"
}
```

### 역할 렌즈 원칙

| 역할 | 같은 상황을 읽는 관점 |
|------|----------------------|
| Implementer | 어떻게 만들 것인가 |
| Verifier | 무엇이 잘못될 수 있는가 |
| Proposer | 어떤 방향이 맞는가 |
| Critic | 어떤 가정이 틀렸는가 |

### 이 원칙 위반 감지 기준

다음 중 하나라도 해당하면 설계가 이 원칙을 위반한 것이다:

```
[ ] Verifier가 Implementer 제출물만 보고 판단 — situation 독립 접근 불가
[ ] 한 에이전트의 자진 신고가 다른 에이전트의 시작점이 됨
[ ] Coordinator가 situation을 독립 고정하지 않고 자진 신고를 그대로 전달
[ ] 역할 간 정보 비대칭 — 한쪽은 전체 상황, 다른 쪽은 요약만
[ ] 한 역할이 다른 역할의 하위 처리 단계처럼 동작
```

---

## 1. 개요

### 목적
두 개 이상의 AI 에이전트가 동일한 프로젝트를 자율 협업할 수 있도록 중재하는 MCP.
사람은 초기 설정과 ESCALATED 상황에서만 개입한다. 그 외 모든 턴은 에이전트들이 MCP 버스를 통해 자율 진행한다.

### 핵심 전제
- 에이전트들은 직접 통신 불가. 이 MCP가 유일한 통신 버스다.
- 에이전트가 하나라도 미연결이면 세션은 시작되지 않는다.
- 에이전트 종류는 사용자가 자유 지정 (Claude Code, Codex, Gemini CLI 등 MCP 지원 에이전트).
- Coordinator는 상태 관리자다. 설계 의사결정은 Coordinator 권한 밖이다.

### Memento와의 관계
```
Memento MCP (장기 기억)              Coordinator MCP (단기 작업층)
──────────────────────────────────────────────────────────────────
세션 종료 후에도 유지                  작업 완료 후 소멸 가능
프로젝트 규칙, 패턴, 검증된 결론       현재 턴, 쟁점, 요약, 락, 의무 목록
다음 세션 컨텍스트 복원용              지금 이 작업의 실시간 버스
        ↑
        stable=true 판정된 결론만 올라감
        (중간 반박, 미완성 판단, 임시 가설 저장 금지)
```

---

## 2. 시스템 아키텍처

### 2.1 3층 메모리 구조
```
┌──────────────────────────────────────────────┐
│  Layer 3: Memento MCP (장기 기억)              │  ← stable=true 결론만
├──────────────────────────────────────────────┤
│  Layer 2: Task State (작업 상태 + 체크포인트)  │  ← task/turn/role/checkpoint
├──────────────────────────────────────────────┤
│  Layer 1: Message Bus (즉시 버스, TTL)         │  ← 구현요약/판정/반박/쟁점
└──────────────────────────────────────────────┘
```

### 2.2 에이전트 연결 구조
```
[사용자]
   │ 초기 설정 (모드, 역할 배정, 예산, acceptance_criteria)
   ▼
[Coordinator MCP]
   ├── read/write ──→ [Agent A: Implementer (PROJECT/SINGLE) | Proposer (DISCUSSION)]
   └── read/write ──→ [Agent B: Verifier (PROJECT/SINGLE)    | Critic   (DISCUSSION)]
```

### 2.3 에이전트 선택 및 세션 설정
```json
{
  "session_config": {
    "mode": "PROJECT | SINGLE | DISCUSSION",
    "agents": {
      "implementer": "claude-code",
      "verifier": "codex"
    },
    "project_brief": "프로젝트 목적과 핵심 방향성 (필수)",
    "task_scope": {
      "allowed_write_scope": ["src/", "tests/"],
      "adjacent_files_allowed": ["package.json", "tsconfig.json", "*.types.ts"],
      "expansion_requires_approval": true
    },
    "acceptance_criteria": [
      "모든 기존 테스트 통과",
      "신규 기능 X 동작 확인",
      "성능 지표 Y 이상"
    ],
    "task_pass_condition": {
      "allow_deferred_low_on_pass": true,
      "deferred_must_transfer_to_next_task": true,
      "project_done_requires_all_deferred_resolved": true
    },
    "budget": {
      "max_negotiation_turns_per_cycle": 3,
      "max_verification_cycles_per_task": 5,
      "max_total_turns_per_session": 50
    },
    "memento_sync_interval_turns": 10
  }
}
```

**세션 시작 체크리스트 (Coordinator 자동 수행):**
1. 지정된 모든 에이전트 MCP 연결 확인 → 실패 시 즉시 중단
2. acceptance_criteria 존재 여부 확인 (PROJECT 모드에서 필수)
3. session_id 발급, 초기 상태 기록
4. 각 에이전트에 역할 + 초기 컨텍스트 주입

---

## 3. 모드 정의

### 3.1 MODE: PROJECT (프로젝트 완주)

**목적:** 프로젝트 전체가 완료될 때까지 구현-검증 루프를 자율 반복.

**흐름:**
```
[사용자: project_brief + acceptance_criteria + task_scope 설정]
        ↓
[Implementer: 작업 수행]
        ↓
[구현 요약 + diff + test_evidence → Coordinator 제출 → Verifier 전달]
        ↓
[Verifier: 보수적 검증 + 구조화 판정]
        ↓
[협의 루프 (Section 8)]
        ↓
[협의 완료 → Implementer 다음 작업]
        ↓
[반복 — 프로젝트 완료 판정까지]
```

**프로젝트 완료 판정 (Coordinator 체크리스트, 추론 금지):**
- [ ] 모든 task가 CLOSED 상태
- [ ] 마지막 task의 최종 판정이 PASS
- [ ] acceptance_criteria 항목 전부 verified 상태
- [ ] open_obligations 중 active 상태 항목 없음
  > **active** = proposed / accepted / in_progress 상태의 합집합. resolved / deferred / rejected는 active 아님.
- [ ] `project_done_requires_all_deferred_resolved=true`인 경우: deferred 항목도 전부 resolved 또는 rejected
- [ ] 사용자의 명시적 완료 승인 (선택 사항, 설정에 따라)

### 3.2 MODE: SINGLE (단건 이행)

**목적:** 사용자 지시 1건만 이행하고 종료.

**흐름:** PROJECT와 동일하되, PASS 후 루프 없이 즉시 DONE → 세션 종료.

### 3.3 MODE: DISCUSSION (토론)

**목적:** 구현 없이 방향성·설계 문서 산출.
프로젝트 초기 설계 또는 풀리지 않는 문제의 방향 재정립에 사용.

**흐름:**
```
[Proposer: 초안] → [Critic: 반박] → [Proposer: 변경분만]
→ 최대 3턴 → 합의 → 설계 문서 산출
→ 합의 실패 → 타협안 1턴 → DISCUSSION 종료
→ 사용자 확인 후 PROJECT 또는 SINGLE로 전환 가능
```

**타협안은 DISCUSSION 모드에서만 허용된다.**
PROJECT/SINGLE 모드에서 blocking/high severity 이슈에 대한 타협은 구조적으로 불가하다.

---

## 4. Issue Ledger

### 4.1 Issue ID 발급 규칙
- **Issue ID는 Coordinator만 발급한다.**
- Verifier는 `issue_drafts`를 Phase C에서 제출한다.
- Coordinator는 Phase C 응답 수신 직후 각 draft에 canonical `issue_id`를 부여한다.
- **draft_id는 Coordinator 내부 매핑용 식별자일 뿐이며 Phase D부터 외부 노출 금지.**
- Phase D 진입 전에 모든 draft_id → canonical issue_id 변환이 완료되어야 한다.
- 이후 모든 상태 전이와 참조는 canonical issue_id 기준으로만 진행한다.
- ID 형식: `{session_id}-V{순번}` (예: `sess001-V003`)

### 4.2 Obligation 생애주기 상태
```
proposed      → Coordinator가 Verifier draft를 canonical ID로 승격, Ledger 등록
accepted      → Implementer가 ACCEPT 표명
in_progress   → Implementer가 현재 해당 이슈 처리 중
resolved      → Verifier가 실제 코드(diff) 기준으로 이행 확인
deferred      → 양쪽 동의 + Coordinator 승인 (low severity만 허용)
rejected      → Implementer 반박 근거 제시 + Verifier 수용
```

**전이 규칙 (severity 통일 기준):**

| severity | 허용 전이 |
|----------|-----------|
| high / blocking=true | `proposed → accepted → in_progress → resolved` 또는 ESCALATED. deferred 불가. |
| medium | `proposed → accepted → in_progress → resolved` 또는 `→ rejected`. deferred 불가. |
| low | 위 전이 + `→ deferred` 허용 (양쪽 동의 + Coordinator 승인) |

**PASS 시점의 deferred low issue 처리:**
- `allow_deferred_low_on_pass=true`이면: deferred low issue가 남아있어도 task 단위 PASS 가능
- 단, `deferred_must_transfer_to_next_task=true`이면: Coordinator가 해당 issue를 다음 task의 open_obligations로 자동 이관
- `project_done_requires_all_deferred_resolved=true`이면: 프로젝트 DONE 전에 모든 deferred 처리 완료 필요

---

## 5. 턴 구조 및 주입 규칙

### 5.1 분업 모드 턴 단계
```
[Turn N]
  Phase A0: 상태 정합성 자기점검 (Preflight) [HARD]
    구현 시작 전 Implementer가 반드시 수행:
      1. open_obligations 재열거 — active(accepted/in_progress) 항목 명시
         ※ Phase A 시점에 proposed 항목이 존재하면 Coordinator 오류 (Phase D에서 미처리)
      2. 지난 턴 obligation_updates 주장 vs 현재 실제 상태 대조
         - "resolved"로 표시한 항목이 실제로 처리됐는가?
         - mismatch 발견 시 구현 전에 명시
      3. 이번 수정 범위가 open_obligations 중 누락하는 항목 없는지 확인
    mismatch 없으면 한 줄로 통과. mismatch 있으면 구현 전에 드러내고 처리.
    결과는 제출 포맷의 preflight 필드에 포함 (Section 6 RULE 4).

  Phase A: 구현
    Coordinator → Implementer 주입:
      - project_brief (고정)
      - task_scope (고정)
      - IMPLEMENTER HARD RULES 전문 (고정)
      - open_obligations 목록 (in_progress/accepted 상태 항목)
      - 이전 협의 agreed_actions
      - 이관된 deferred 항목 (있는 경우)

  Phase B: 구현 요약 + diff + test_evidence 제출
    Implementer → Coordinator 제출 (Section 6 RULE 4 형식)

  Phase C: 검증
    Coordinator → Verifier 주입:
      - 구현 요약
      - 실제 변경 파일 목록 + diff/patch
      - test_evidence (RULE 4 형식 그대로)
      - open_obligations 전체 (obligation_id + 현재 상태)
      - VERIFIER HARD RULES 전문 (고정)
      - 이전 FIX_REQUIRED 이력 (issue_id 포함)
      - project_brief (고정)

  ── Phase C 종료 후, Phase D 진입 전 ──
    Coordinator 처리:
      1. Verifier issue_drafts 형식 검증
      2. 각 draft에 canonical issue_id 발급
      3. Ledger에 proposed 상태로 등록
      4. draft_id → canonical issue_id 내부 매핑 생성 (Phase D 종료 후 폐기)
      5. Negotiation Packet 구성 (canonical issue_id 기준)
      6. verification_cycles_current_task += 1 (단일 source of truth — Section 5.3)

  Phase D: 협의 (FIX_REQUIRED 시 진입)
    - Coordinator가 Negotiation Packet을 Implementer에게 전달
    - Implementer는 canonical issue_id만 참조 (draft_id 비노출)
    - severity 정책에 따라 처리 (Section 8)
    - 최대 max_negotiation_turns_per_cycle 턴
    - 초과 시 → high/blocking 미해결이면 ESCALATED
                low만 남았으면 deferred 처리 후 Phase A

  Phase E: 합의 완료
    Coordinator:
      - agreed_actions 기록
      - deferred 항목 다음 task로 이관 등록
      - 예산 체크 (Section 5.2)
      - draft_id → canonical issue_id 매핑 폐기
      - 다음 Turn Phase A 시작 또는 DONE/ESCALATED
```

### 5.2 예산 체크 (매 Phase E 종료 시)
```
if verification_cycles_current_task >= max_verification_cycles_per_task:
    → ESCALATED ("task 내 최대 검증 사이클 초과")

if total_turns_session >= max_total_turns_per_session:
    → ESCALATED ("세션 총 턴 예산 초과")
```

### 5.3 verification cycle 정의
```
1 cycle = IMPLEMENTING 시작 → Verifier 판정 완료 (PASS or FIX_REQUIRED or BLOCKER)
즉, Phase A → Phase B → Phase C 완료 시 verification_cycles_current_task += 1
Phase D (협의)는 cycle 카운트에 포함되지 않음
task가 변경될 때 verification_cycles_current_task 리셋
```

---

## 6. Implementer 기본 원칙 (HARD RULES)

> 매 구현 턴 시작 시 Coordinator가 전문 주입한다.
> 위반 감지 시 해당 턴 무효화 + 사용자 보고.

### RULE 1: 일회성 땜빵 완료 선언 금지 [HARD]
다음은 완료가 아니다:
- 예시 1개 통과 후 "완료" 선언
- 구체어/패턴/예외 추가 후 "수정 완료" 표현
- 테스트 N개 통과 = 전체 클래스 해결처럼 포장
- if-else 분기 추가를 구조적 해결로 표현

완료 선언 조건: Verifier로부터 PASS 수신 시점에만 완료다.

### RULE 2: 프로젝트 방향성 위반 금지 [HARD]
- 매 턴 주입되는 project_brief를 기준으로 구현 방향 확인
- project_brief와 상충하는 구현 발견 시 → 구현 중단 → `USER_ESCALATION` 요청
- 방향성/설계 충돌이 불확실한 경우 → DISCUSSION 모드 전환 요청
- **Coordinator에게 방향성 해석을 묻지 않는다.** Coordinator는 상태 관리자이며 설계 결정 권한이 없다.

### RULE 3: 확인 없이 구현 시작 금지 [HARD]
- Coordinator의 IMPLEMENTING 상태 신호 없이 코드 수정 시작 금지
- 협의 중(Phase D)에는 어떤 코드도 수정하지 않는다
- 합의 완료 신호 수신 후에만 Phase A 시작

### RULE 4: 검증 모델 전달 요약 + diff + test_evidence 제출 의무 [HARD]
구현 완료 후 반드시 아래 형식으로 제출한다.
제출 없이 검증 단계 진입 불가.

```json
{
  "turn": "<번호>",
  "role": "implementer",
  "preflight": {
    "open_obligations_relisted": ["<obligation_id>"],
    "last_turn_mismatch": [
      {
        "obligation_id": "<issue_id>",
        "claimed_status": "resolved | in_progress",
        "current_assessment": "confirmed | mismatch | unclear",
        "reason": "<불일치 사유 또는 확인 근거>"
      }
    ],
    "all_clear": true
  },
  "task_summary": "<무엇을 구현했는가 — 1~3문장>",
  "changes": [
    {
      "file": "<파일명>",
      "type": "add | modify | delete",
      "scope": "allowed | adjacent | expansion_requested",
      "description": "<변경 내용 구체적으로>",
      "diff_ref": "<diff 참조 ID 또는 내용>"
    }
  ],
  "test_evidence": {
    "ran": true,
    "commands": ["<실행한 명령어>"],
    "summary": "<테스트 결과 요약>",
    "passed": 0,
    "failed": 0,
    "not_run_reason": "<ran=false인 경우 반드시 기재. 빈 문자열 불가>"
  },
  "obligation_updates": [
    {
      "obligation_id": "<canonical issue_id>",
      "new_status": "resolved | in_progress | deferred_request",
      "evidence": "<어떻게 처리했는가 — 파일명:라인 기준>"
    }
  ],
  "verification_focus": ["<Verifier가 반드시 확인해야 할 항목>"],
  "side_effects": ["<작업 범위 외 발견 문제 — 수정 안 함, 기록만>"],
  "adjacent_files_used": ["<인접 파일 수정 목록>"],
  "self_check": {
    "runs_without_error": true,
    "matches_spec": true,
    "notes": "<자가 점검 메모>"
  }
}
```

**test_evidence.ran=false 허용 조건:**
- `not_run_reason`에 구체적 이유 필수 기재
- Verifier는 ran=false 자체를 이슈로 등록할 수 있음
- "환경 미구성", "시간 부족" 등 비이유는 불인정

### RULE 5: 동의 후 완료 환각 금지 [HARD]
"동의한다", "반영하겠다"는 완료가 아니다.
동의한 항목은 obligation_updates에 `in_progress`로 즉시 등록하고 다음 구현 턴에서 처리한다.
동의 후 곧바로 다음 단계 진입 금지.

### RULE 6: 작업 범위 제어 [HARD]
```
allowed_write_scope:    자유롭게 수정 가능
adjacent_files_allowed: 수정 가능하되 changes[]에 scope="adjacent"로 명시
그 외:                  수정 금지 — side_effects에 기록
                        범위 확장 필요 시 expansion_request 제출 → 사용자 승인 후만 허용
```
범위 확장 요청 형식:
```json
{
  "type": "expansion_request",
  "reason": "<왜 범위 확장이 필요한가>",
  "requested_files": ["<파일명>"],
  "impact": "<확장 시 영향 범위>"
}
```

---

## 7. Verifier 기본 원칙 (HARD RULES)

> 매 검증 턴 시작 시 Coordinator가 전문 주입한다.

### RULE 1: 누적 약속 이행 검증 [HARD]
반드시 아래 3단계 순서로 진행한다:

1. **미해결 의무 재열거**: open_obligations 중 accepted/in_progress 상태인 항목
2. **이번 변경이 닫은 의무**: obligation_updates + 실제 diff 기준으로만 인정 (요약 불인정)
3. **아직 열린 의무**: 다음 turn previous_unresolved에 승계

이 3단계 없이 PASS 판정 불가.

### RULE 2: diff 기반 검증, 요약 기반 판정 금지 [HARD]
- Coordinator가 전달한 diff/patch를 직접 읽은 후 판정
- implementation_summary 요약만으로 판정 금지
- test_evidence.ran=false인 경우: 해당 사실 자체를 이슈 draft로 등록 가능
- "잘 된 것 같습니다" 형태의 추상 판단 금지

### RULE 3: 쓰기 금지
- 코드 직접 수정 금지
- read + critique만 허용
- 수정 제안은 required_fix 항목으로 제시 (직접 구현 금지)

### RULE 4: 대안 경로 검토 의무
- 제시된 원인 외 대안 경로, 더 작은 수정 지점, 부작용 확인
- 새 설계 제안 시 기존 구현과 반드시 대조
- 이미 있는 기능을 무시한 채 추상 설계만 덧씌우지 않는다

### RULE 6: 과검증 방지 + PASS 허용 의무 [HARD]
검증의 목적은 문제 개수를 늘리는 것이 아니다.
현재 diff와 수정안을 기준으로, **다음 턴으로 넘기면 안 되는 실질적 미해결 문제가 있는지** 확인하는 것이다.

**검증 우선순위 (이 순서로만 검토):**
1. 정확성 — 로직 오류, 명세 불일치
2. 누적 의무 이행 — open_obligations 실제 처리 여부
3. acceptance_criteria 충족 여부
4. 회귀 위험 — 기존 기능 파손 가능성
5. 범위 위반 — scope 외 수정
6. 그 외 사소한 항목

**절대 금지:**
- 사소한 문구 차이, 비본질적 형식 문제 지적
- resolved / rejected / deferred 처리된 항목 재지적
- 이전 턴에서 합의된 항목 반복 제기
- 새로운 근거 없이 동일 주장 재제기
- 억지 쟁점 생성

**PASS 허용 의무:**
우선순위 1~5에 해당하는 실질적 미해결 문제가 없으면 PASS를 내야 한다.
PASS를 유보할 이유를 억지로 만들어내지 않는다.

### RULE 5: 판정 형식 고정 + issue_drafts 제출 [HARD]
감상문, 권장사항 나열, "더 고려해보세요" 표현 금지.
**draft_id는 Verifier 내부 임시 식별자다. 반드시 포함해야 하며, Coordinator가 canonical issue_id로 교체 후 Phase D에서 외부 노출 금지.**

```json
{
  "turn": "<번호>",
  "role": "verifier",
  "verdict": "PASS | FIX_REQUIRED | BLOCKER",
  "confidence": 0.0,
  "obligations_checked": [
    {
      "obligation_id": "<canonical issue_id>",
      "verified_status": "resolved | unresolved | partially_resolved",
      "evidence": "<실제 diff에서 확인한 내용 — 파일명:라인>"
    }
  ],
  "accepted_points": ["<정상 동작 확인 항목>"],
  "issue_drafts": [
    {
      "draft_id": "<Verifier 임시 ID — Coordinator가 canonical ID로 교체>",
      "severity": "high | medium | low",
      "blocking": true,
      "category": "logic | edge_case | performance | security | spec_mismatch | scope_violation | test_missing",
      "reason": "<무엇이 왜 문제인가>",
      "required_fix": "<파일명:라인 기준 구체적 수정 사항>",
      "diff_location": "<해당 diff 내 위치>",
      "alternative_considered": "<검토한 대안>"
    }
  ],
  "previous_unresolved": ["<미해결 canonical issue_id 목록>"],
  "retest_focus": ["<다음 턴 재확인 항목>"]
}
```

판정 기준:
- **PASS**: 아래 조건 전부 충족
  - active high obligation 없음 (resolved만 — rejected 불가)
  - active medium obligation 없음 (resolved 또는 rejected만)
  - active blocking obligation 없음 (resolved만 — rejected 불가)
  - acceptance_criteria 항목 충족
  - low deferred는 `allow_deferred_low_on_pass` 정책에 따라 허용
- **FIX_REQUIRED**: 수정 가능한 문제 존재, 재구현 후 재검증 필요
- **BLOCKER**: 현재 방향으로 진행 불가 → Coordinator에 ESCALATED 요청

confidence는 참고 지표로만 사용. 단독 교착 트리거로 사용 불가.

### RULE 7: 검증 탐색 범위 규칙 [HARD]
기본적으로 Coordinator가 제공한 artifact bundle만 검증 대상이다.
매 턴 repo 전체를 탐색하지 않는다.

```
[단계 1: 기본 검증 — 항상 수행]
- artifact bundle의 changed_files + patch
- test_outputs
- open_obligations 관련 파일

[단계 2: 인접 확장 — 아래 조건 중 하나라도 해당 시]
- public interface / API 변경
- config / schema / migration 변경
- shared util / 공통 모듈 변경
- side_effects에 인접 파일 기록 존재
- preflight mismatch 존재

[단계 3: 전체 확장 — 명시적 트리거 있을 때만]
- 제출물과 실제 snapshot diff 불일치 의심
- changed_files 누락 의심
- scope 위반 의심
- 동일 파일에서 반복 불일치 이력 (2회 이상)
- 단계 1~2로 판정 불가한 경우

[탐색 종료 조건 — 5개 모두 충족 시 추가 탐색 없이 판정 가능]
  [ ] changed_files가 artifact bundle의 snapshot diff와 일치
  [ ] patch가 base_revision 대비 완전
  [ ] test output artifact 존재 + ran=true
  [ ] scope 위반 없음 (changed_files 전체 allowed/adjacent)
  [ ] obligation mismatch 없음
→ 5개 충족 시 전체 repo 탐색 없이도 PASS 가능
→ "혹시 빠진 게 있지 않을까"는 이 5개가 충족되면 종료해야 할 불안이다
```

---

## 8. 협의 루프 (Phase D) 규칙

### 진입 조건
Verifier 판정이 FIX_REQUIRED이고 Coordinator가 Negotiation Packet 생성 완료 시.

### Phase C → Phase D 전환 절차 (Coordinator)
```
1. Verifier issue_drafts 형식 검증
2. 각 draft에 canonical issue_id 발급 (형식: {session_id}-V{순번})
3. Ledger에 proposed 상태로 등록
4. 내부 draft_id → canonical issue_id 매핑 생성
5. Negotiation Packet 구성:
   {
     "canonical_issues": [
       {
         "issue_id": "<canonical>",
         "severity": "...",
         "blocking": true/false,
         "reason": "...",
         "required_fix": "..."
       }
     ],
     "deferred_low_allowed": true/false,
     "turns_remaining": <max - current>
   }
6. Negotiation Packet을 Implementer에게 전달
7. draft_id는 이 시점부터 외부에 노출하지 않음
```

### severity별 처리 규칙 (통일)
```
high 또는 blocking=true:
  → resolved 또는 ESCALATED만 허용
  → deferred 불가
  → 타협 불가

medium:
  → resolved 또는 rejected(반박 수용) 허용
  → deferred 불가
  → 타협 불가

low:
  → resolved, rejected, deferred 모두 허용
  → deferred 조건: 양쪽 동의 + Coordinator 승인 + task_pass_condition 정책 준수
```

### 재지적 금지 규칙 (Verifier + Implementer 공통)
협의 루프에서 다음 행동은 금지된다:
- 이미 resolved / rejected / deferred 처리된 issue_id 재제기
- 이전 턴 대비 변화가 없는 주장 반복
- 새 근거(diff, 코드 위치, 테스트 결과) 없이 동일 주장 재제기
- 이미 agreed_actions에 기록된 합의 사항 번복

위반 감지 시 Coordinator가 해당 발언을 무효 처리하고 경고를 발행한다.

### 반박 형식 (Implementer → Coordinator)
```json
{
  "turn": "<번호>",
  "role": "implementer",
  "rebuttal": [
    {
      "issue_id": "<canonical issue_id>",
      "action": "ACCEPT | REBUT | DEFER_REQUEST",
      "reason": "<수용 이유 또는 반박 근거>",
      "proposed_alternative": "<반박 시 대안>",
      "timeline": "this_turn | next_turn"
    }
  ]
}
```

### 협의 종료 조건
```
정상 종료:
  모든 high/blocking → resolved (타협/rejected 불가)
  모든 medium → resolved 또는 rejected (합의)
  remaining issues 전부 deferred (low만)

비정상 종료 (ESCALATED):
  max_negotiation_turns_per_cycle 초과 + high/blocking 미해결 → ESCALATED
  (ESCALATED는 정상 종료 경로가 아님)

DISCUSSION 모드에서만 forced_compromise 허용.
PROJECT/SINGLE 모드에서 forced_compromise 생성 불가.
```

### Phase D 종료 후
```
Coordinator:
  - agreed_actions 기록
  - deferred 항목: next task open_obligations에 이관 등록
  - draft_id → canonical issue_id 내부 매핑 폐기
  - Phase E 진입
```

---

## 9. 상태 머신

```
INITIALIZING
  → 에이전트 연결 확인
  → 실패: AGENT_UNAVAILABLE (세션 종료)
  → 성공: 모드별 첫 상태

───────────────── DISCUSSION 모드 ─────────────────

DISCUSSING
  → 합의: CONSENSUS_REACHED
  → 타협: COMPROMISE_REACHED
  → 실패: ESCALATED

CONSENSUS_REACHED / COMPROMISE_REACHED
  → 사용자 확인 후 PROJECT/SINGLE 모드 진입 가능
  → 또는 세션 종료

───────────────── PROJECT / SINGLE 모드 ─────────────────

IMPLEMENTING
  → 구현 요약 + diff + test_evidence 제출 완료: VERIFYING

VERIFYING
  → Verifier 판정 완료 → verification_cycles_current_task += 1 (Section 5.3 참조)
  → PASS + 완료 체크리스트 미충족 (PROJECT): NEXT_TASK
  → PASS + 완료 체크리스트 충족 (PROJECT): DONE
  → PASS (SINGLE): DONE
  → FIX_REQUIRED: [Phase C→D 전환 절차] → NEGOTIATING
  → BLOCKER: ESCALATED

NEGOTIATING
  → 합의 완료 (모든 high/blocking 해결): IMPLEMENTING
  → 협의 실패 + high/blocking 미해결: ESCALATED
  → 협의 완료 (low만 deferred): IMPLEMENTING

NEXT_TASK (PROJECT 전용)
  → 다음 task: IMPLEMENTING (verification_cycles_current_task 리셋)
  → 완료 체크리스트 전부 충족: DONE

───────────────── 공통 상태 ─────────────────

PAUSED
  → 에이전트 응답 없음 (재시도 N회 실패)
  → 예산 임박 경고
  → 사용자 개입 대기
  → 에이전트 재연결 시: RESTORING

RESTORING
  → Coordinator가 최신 체크포인트 로드
  → 복원 패키지 구성 완료: RESYNCING

RESYNCING
  → 각 에이전트에 복원 패키지 주입
  → 에이전트 ACK 수신 대기
  → 모든 에이전트 ACK 확인: READY_TO_RESUME
  → ACK 타임아웃: PAUSED 재진입

READY_TO_RESUME
  → Coordinator: 직전 중단 상태로 복귀
  → 복귀 전 사용자 알림 (선택)

DONE
  → 체크포인트 최종 저장 → Memento 동기화 → 세션 종료 → 사용자 보고

ESCALATED
  → 사용자 보고 → PAUSED 대기
  → 사용자 지시에 따라 어느 상태로든 재진입 가능
```

---

## 10. 교착 탈출 메커니즘

### 트리거 조건 (하나라도 충족 시 ESCALATED)
1. 동일 issue_id가 `previous_unresolved`에 **3턴 연속** 등장
2. Verifier가 `BLOCKER` 판정 발행
3. `verification_cycles_current_task >= max_verification_cycles_per_task`
4. `total_turns_session >= max_total_turns_per_session`
5. 협의 루프에서 `max_negotiation_turns_per_cycle` 초과 + high/blocking 미해결

> confidence < 0.4는 교착 트리거로 사용하지 않는다.
> confidence는 참고 지표이며, 실제 트리거는 위 5개 조건으로만 판정한다.

### ESCALATED 보고 형식
```
[교착 상태 감지]
원인: {trigger 조건}
현재 상태: {마지막 구현 요약}
미해결 이슈: {issue_id + severity + reason 목록}
진행 턴: {total_turns_session} / 예산: {max_total_turns_per_session}
검증 사이클: {verification_cycles_current_task} / 예산: {max_verification_cycles_per_task}
권장 행동: {DISCUSSION 모드로 방향 재정립 | 수동 중재 | 세션 종료}
```

---

## 11. 장기 프로젝트 중단 방지 + 체크포인트 복원 프로토콜

### 11.1 체크포인트 저장 트리거
- PASS 판정 직후 (자동)
- `memento_sync_interval_turns` 도달 시 (자동)
- PAUSED 진입 시 (자동)
- 사용자 명시적 요청 시 (수동)

### 11.2 체크포인트 저장 내용
```json
{
  "checkpoint_id": "<자동 생성>",
  "session_id": "<session_id>",
  "task_id": "<현재 task_id>",
  "saved_at": "<ISO 8601 타임스탬프>",
  "current_state": "<IMPLEMENTING | VERIFYING | NEGOTIATING | PAUSED>",
  "current_turn": "<번호>",
  "current_phase": "<A | B | C | D | E>",
  "verification_cycles_current_task": 0,
  "total_turns_session": 0,
  "writer_lock": {
    "holder": "<agent_id>",
    "scope": "<허용된 파일 목록>",
    "acquired_at": "<타임스탬프>",
    "expires_at": "<타임스탬프>"
  },
  "open_obligations": [
    {
      "obligation_id": "<canonical issue_id>",
      "status": "proposed | accepted | in_progress | deferred",
      "severity": "high | medium | low",
      "blocking": true,
      "summary": "<한 줄 요약>",
      "assigned_to": "<agent_id>",
      "consecutive_turns_unresolved": 0
    }
  ],
  "unresolved_issues": [
    {
      "issue_id": "<canonical issue_id>",
      "severity": "high | medium | low",
      "blocking": true,
      "reason": "<요약>",
      "required_fix": "<구체적 수정 사항>",
      "consecutive_turns_unresolved": 0
    }
  ],
  "last_verified_changes": "<마지막 PASS 시점의 구현 요약>",
  "recent_agreed_actions": ["<최근 3턴 합의 사항>"],
  "project_brief_snapshot": "<project_brief 전문>",
  "acceptance_criteria_status": [
    {
      "criterion": "<항목>",
      "status": "pending | verified | waived"
    }
  ],
  "completed_tasks": ["<완료된 task 목록>"],
  "budget_remaining": {
    "total_turns_left": 0,
    "cycles_left_current_task": 0
  }
}
```

### 11.3 체크포인트 복원 트리거
다음 상황 발생 시 Coordinator가 RESTORING 상태 진입:
- 에이전트 재연결 감지 (PAUSED → 재연결)
- 세션 재시작 감지
- 에이전트 컨텍스트 손실 감지
- 컨텍스트 압축 실행 후

### 11.4 복원 패키지 주입 형식 (Coordinator → 각 에이전트)

**write_permission은 현재 writer_lock 상태와 에이전트 역할 기준으로 동적 계산된다.**
복원 시점의 current_state와 writer_lock.holder를 확인해 결정한다.
예시의 true/false는 고정값이 아님.

```json
{
  "type": "CHECKPOINT_RESTORE",
  "session_id": "<session_id>",
  "task_id": "<task_id>",
  "restore_summary": "<한 단락 — 지금까지 무슨 일이 있었는가>",
  "current_state": "<복원 시점 상태>",
  "your_role": "<implementer | verifier>",
  "your_write_permission": "<writer_lock.holder == this_agent_id AND current_state == IMPLEMENTING>",
  "open_obligations": [
    {
      "obligation_id": "<canonical issue_id>",
      "status": "proposed | accepted | in_progress | deferred",
      "severity": "high | medium | low",
      "blocking": true,
      "summary": "<한 줄 요약>",
      "assigned_to": "<agent_id>",
      "consecutive_turns_unresolved": 0
    }
  ],
  "unresolved_issues": [
    {
      "issue_id": "<canonical issue_id>",
      "severity": "high | medium | low",
      "blocking": true,
      "reason": "<요약>",
      "required_fix": "<구체적 수정 사항>"
    }
  ],
  "last_verified_changes": "<마지막 PASS 구현 요약>",
  "current_phase": "<A | B | C | D | E>",
  "next_required_action": "<지금 당장 해야 할 것>",
  "project_brief": "<전문>",
  "budget_remaining": {
    "total_turns_left": 0,
    "cycles_left_current_task": 0
  }
}
```

### 11.5 복원 후 재개 규칙
```
1. Coordinator → 각 에이전트에 복원 패키지 주입 (RESYNCING)
2. 각 에이전트 → ACK 반환
3. 모든 에이전트 ACK 수신 → READY_TO_RESUME
4. Coordinator → 직전 상태로 복귀
5. 복귀 전 사용자 알림 (선택)

ACK 없이 직전 상태 복귀 금지.
ACK 타임아웃 시 PAUSED 재진입.
```

**에이전트 ACK 형식:**
```json
{
  "type": "RESTORE_ACK",
  "agent_id": "<agent_id>",
  "session_id": "<session_id>",
  "understood": true,
  "current_state_confirmed": "<복원 패키지에서 받은 current_state>",
  "next_action_confirmed": "<next_required_action 확인>",
  "questions": ["<불명확한 점 — 없으면 빈 배열>"]
}
```

**Coordinator ACK 질문 처리 범위:**
questions가 비어 있지 않을 때 Coordinator가 답변할 수 있는 것:
- 현재 상태 (state, phase, turn)
- 작업 범위 (task_scope, allowed_write_scope)
- Issue Ledger 내용 (issue_id, status, severity)
- 예산 잔량 (turns_left, cycles_left)
- Writer lock 상태

Coordinator가 답변할 수 없는 것 (→ USER_ESCALATION 또는 DISCUSSION 재진입):
- 설계 방향 해석
- 의도/우선순위 충돌 판단
- project_brief의 의미 해석
- acceptance_criteria의 충족 기준 해석

### 11.6 컨텍스트 압축 규칙
누적 턴이 임계값 초과 시:
- **보존**: open_obligations 전체, unresolved_issues 전체, 최근 3턴 전문, project_brief, acceptance_criteria 상태
- **압축**: 해결된 이슈 상세, 오래된 협의 내용, closed obligations 상세
- **압축 후**: RESTORING → RESYNCING → READY_TO_RESUME 사이클 강제 실행

### 11.7 Memento 중간 동기화 기준
`memento_sync_interval_turns` 도달 시 아래 `stable=true` 조건 만족 항목만 동기화:

```
stable=true 조건:
  - PASS 판정을 받은 구현 요약
  - 3회 이상 반복된 이슈 패턴 (같은 category)
  - 양쪽 모두 accepted + Verifier가 resolved로 마킹한 합의 규칙
  - Verifier가 PASS에서 verified로 마킹한 acceptance_criteria

stable=false (동기화 금지):
  - 협의 중인 이슈
  - FIX_REQUIRED 상태의 구현 내용
  - in_progress 상태 의무
  - 임시 가설, 반박문, deferred 항목 (resolved 전)
```

---

## 12. 토론 모드 (DISCUSSION) 턴 구조

### Proposer 출력 형식
```json
{
  "turn": "<번호>",
  "role": "proposer",
  "position_summary": "<현재 입장 한 줄>",
  "changes_this_turn": ["<이번 턴 변경분만 — 전체 재작성 금지>"],
  "unresolved": ["<미합의 쟁점>"],
  "questions_for_critic": ["<질문 1~3개>"],
  "confidence": 0.0
}
```

### Critic 출력 형식
```json
{
  "turn": "<번호>",
  "role": "critic",
  "agrees": ["<동의 항목>"],
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

### 토론 모드 재지적 금지 규칙
- 이전 턴에서 proposer가 반영 완료하거나 합의된 항목은 재제기 금지
- Critic은 변경분(changes_this_turn)에만 반응한다. 이전 턴 전체 입장을 다시 비판하지 않는다
- 새로운 근거 없이 동일 disagreement 항목을 반복 제기 금지
- 이미 동의한 항목을 severity 상향 조정하거나 새 조건을 추가해 번복 금지

위반 감지 시 Coordinator가 해당 발언을 무효 처리한다.

### 토론 종료 조건
**정상 합의 (체크리스트 기반, confidence 값은 참고 지표로만 사용):**
- critic.consensus_ready == true
- proposer.unresolved 비어 있음
- critic.open_questions 비어 있음
- critic.disagrees 중 severity=high 항목 없음

**타협 종료 (3턴 초과 시):**
- 1턴 추가 → 양쪽이 양보 가능한 최소 합의안 작성
- 여전히 미합의 → ESCALATED

**산출물:**
```json
{
  "type": "design_document | compromise_document",
  "summary": "<합의/타협 내용 전문>",
  "open_issues": ["<미합의 항목>"],
  "recommended_next": "PROJECT | SINGLE | USER_DECISION"
}
```

---

## 13. Memento 연동 규칙

| 항목 | 저장 조건 |
|------|-----------|
| 최종 합의안 / 타협안 | DISCUSSION 종료 후 stable=true |
| 검증 통과 구현 요약 | PASS 판정 직후 |
| 반복 이슈 패턴 | 동일 category 3회 이상 |
| 프로젝트 방향성 결정 | project_brief 변경 시 |
| 검증 기준 합의 | Verifier/Implementer 합의된 판정 기준 |

**저장 금지:**
- 중간 반박문
- FIX_REQUIRED 상태의 구현 내용
- 협의 중 임시 가설
- 타협안 초안
- deferred 항목 (resolved 전)

---

## 14. Coordinator 동작 요약

### 역할 범위
- 상태 관리자 + 턴 관리자 + Issue Ledger 관리자 + Writer Lock 관리자
- 설계 의사결정자가 아님
- 방향성/설계 충돌 판정 권한 없음 → DISCUSSION 또는 USER_ESCALATION으로 위임

### 매 턴 처리 순서
1. 현재 상태 + 예산 잔량 확인
2. 담당 에이전트 결정
3. 주입 컨텍스트 구성 (project_brief + HARD RULES + 동적 항목)
4. 에이전트 응답 수신 + 형식 검증
5. Phase C 완료 시: issue_draft → canonical issue_id 발급 + Ledger 등록
6. 교착 조건 체크
7. 상태 전이 판정
8. 체크포인트 저장 조건 확인

### Writer Lock 관리
```
writer_lock = {
  holder: "<현재 구현 권한 보유 agent_id>",
  scope: "<allowed_write_scope + adjacent_files_allowed>",
  acquired_at: "<타임스탬프>",
  expires_at: "<타임스탬프>"
}
```
- IMPLEMENTING 진입 시 Implementer에게 lock 발급
- VERIFYING 진입 시 lock 회수
- RESTORING/RESYNCING 중: lock 상태 체크포인트 기준으로 동결
- READY_TO_RESUME 시: 직전 lock 상태 복원
- lock 없는 에이전트의 write 시도 → 즉시 차단 + 경고

---

## 15. 제출/검증 대상 고정 메커니즘

> 구현 세부 명세(컬럼명, endpoint 등)는 구현 시 확정한다.
> 이 섹션은 흔들리면 안 되는 설계 불변 조건만 정의한다.

### 불변 조건

```
1. 모든 제출(submission)은 고유한 submission_id를 가진다

2. 모든 제출은 제출 시점 워크스페이스 상태를 고정하는
   base_revision (tree hash 또는 snapshot_id)을 포함한다

3. Verifier가 검증하는 대상은 제출 시점에 고정된 snapshot 기준이다
   - live workspace 직접 접근 금지
   - Coordinator가 제공하는 artifact bundle (patch + changed file list + test outputs + base_revision) 기준으로만 검증

4. 모든 상태 전이는 expected_state를 포함한 CAS(compare-and-swap) 방식으로만 실행된다
   - 전이 요청의 expected_state ≠ 현재 실제 state → 전이 거부

5. 이미 처리된 submission_id는 재처리하지 않는다 (멱등성 보장)
```

### Coordinator artifact bundle 구성 (Implementer → Verifier 전달)
```
{
  "submission_id": "<고유 ID>",
  "base_revision": "<제출 시점 snapshot 식별자>",
  "changed_files": ["<파일 목록>"],
  "patch": "<diff 전문 또는 참조>",
  "test_outputs": "<stdout/stderr artifact 참조>",
  "submitted_at": "<타임스탬프>"
}
```

> Coordinator는 Implementer 자진 신고 diff를 그대로 전달하지 않는다.
> 실제 snapshot 비교(git diff 또는 동등한 수단)로 independent하게 생성하여 전달한다.

### Verifier → Implementer handoff 고정 (대칭 원칙)

FIX_REQUIRED / BLOCKER 판정도 동일한 고정 메커니즘이 적용된다.

**추가 불변 조건:**
```
6. FIX_REQUIRED/BLOCKER 판정은 고유한 verdict_id를 가진다

7. verdict는 Verifier가 분석한 base_revision에 결합된다
   - "어느 snapshot 기준으로 이슈를 찾았는가"가 고정됨

8. Coordinator는 다음 구현 사이클의 fix_base_revision을 명시적으로 고정한다
   - fix_base_revision = Verifier가 분석한 base_revision (원칙)
   - workspace가 그 이후 변경됐다면 → RESTORE 또는 새 cycle 시작

9. Implementer는 fix_base_revision과 다른 workspace에서 수정 시작 불가
   - 불일치 감지 시 Coordinator가 차단 + PAUSED 전환
```

> 불변 조건 1~5는 Implementer → Verifier handoff를 고정한다.
> 불변 조건 6~9는 Verifier → Implementer handoff를 대칭으로 고정한다.
> 두 방향 모두 Coordinator가 독립적으로 snapshot을 고정하며, 어느 에이전트의 자진 신고도 ground truth가 아니다.

> 스냅샷 고정(증거 원본성)과 멱등성(중복 전이 방지)은 같은 뿌리에서 나온다.
> submission_id + base_revision + expected_state 세 필드가 하나의 메커니즘으로 두 문제를 해결한다.

---

## 16. MVP 범위

```
[v1 공식 지원]
- 2-agent 구조: Implementer / Verifier 쌍
- 2-agent 구조: Proposer / Critic 쌍 (DISCUSSION 모드)

[v1 명시적 비지원]
- 3-agent 이상 동시 참여
- 동일 역할 병렬 실행 (Implementer 2명 동시 등)
- 동적 역할 추가 (런타임 중 역할 변경)

[비지원 이유]
현재 설계 전체(Issue Ledger, Writer Lock, 상태 기계, preflight)가
2-agent 쌍을 전제로 최적화되어 있다.
3-agent 이상은 별도 설계 검토가 필요하다.
```

---

## 17. 미결 설계 항목

### Batch 1 — 코드 첫 커밋 전 필수
| 항목 | 내용 |
|------|------|
| MCP 도구 목록 | 에이전트가 호출하는 실제 도구 이름 + 파라미터 (MCP_API.md로 분리) |
| SQLite 스키마 | tasks / messages / obligations / locks / checkpoints / submissions 테이블 (SCHEMA.md로 분리) |
| 동적 주입 파라미터 생성 로직 | Coordinator가 {open_obligations}, {diff_ref} 등을 어떻게 조립하는가 |
| submission_id 발급 규칙 | 고유성 보장 방식 (UUID v4 등) |
| base_revision 추출 방식 | git tree hash vs 별도 snapshot 메커니즘 |

### Batch 2 — 첫 사이클 실행 후
| 항목 | 내용 |
|------|------|
| E2E 검증 계획 | 최소 1회 구현→검증→FIX_REQUIRED→재구현→PASS 사이클 (VALIDATION.md) |
| 운영자 UX / 관찰성 | 현재 상태, PAUSED/ESCALATED 원인, resume/cancel/rebaseline 흐름 (UX_FLOW.md) |
| 에이전트 신원 확인 | agent_id 발급 방식, 인증 방법 |
| ACK 타임아웃 값 | 에이전트 응답 대기 최대 시간 |

### Batch 3 — v1.0 전
| 항목 | 내용 |
|------|------|
| 요구사항 변경 프로토콜 | 중간 변경 시 obligations 무효화/이관, rebaseline 흐름 (CHANGE_PROTOCOL.md) |
| 역할별 도구 권한 모델 | Implementer/Verifier별 tool allowlist, shell/destructive command 제한 |
| 폴링 vs 웹소켓 | 에이전트 차례 감지 방식 최종 결정 |
| diff 전달 방식 | MCP 메시지 인라인 vs 파일 경로 참조 |
| 컨텍스트 압축 알고리즘 | 무엇을 남기고 버릴지 판단 기준 |
| docker-compose 구성 | Coordinator MCP 서비스 정의 |

---

*v0.7 — 2026-04-16*
*v0.6.1 → v0.7: Section 15 제출/검증 대상 고정 메커니즘 추가, Section 16 MVP 범위 명시, Section 17 미결 항목 Batch 1/2/3 재구조화*
*v0.5 → v0.6: Phase A0 추가 (Implementer preflight), RULE 4 preflight 필드 추가*
*v0.6 → v0.6.1: Phase A0 active 범위 통일 (accepted/in_progress only)*
*실제 주입 원문은 PROMPT_CONTRACT.md 참조*
