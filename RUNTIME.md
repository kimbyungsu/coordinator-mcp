# Coordinator MCP — Runtime Architecture v0.1

> 이 문서는 자동 턴 전환 런타임의 설계 방향을 결정하기 위한 문서다.
> DESIGN.md가 "무엇을 협업하는가"라면, 이 문서는 "그 협업을 누가 어떻게 실행시키는가"다.

---

## 1. 제품 원칙 (금지선)

이 섹션은 협상 불가 원칙이다. 구현 난이도나 편의성을 이유로 아래 원칙을 우회하는 설계는 채택하지 않는다.

```
[자동화 금지선]
- 사람이 턴을 수동으로 전환하는 구조는 제품 모드로 채택 불가
- 반자동(에이전트에게 "이제 네 차례야"를 사람이 말해주는 구조)은 디버그 전용
- 구현 편의를 이유로 사람 중계 노동이 복귀하는 설계는 ESCALATED 대상
- "초기 버전이니까 일단 수동으로" 논리는 제품 정체성 훼손으로 간주

[자동화 최소 요건 — MVP 통과 기준]
- Implementer가 작업 완료 → Verifier 차례로 자동 전환
- Verifier 판정 완료 → Implementer 차례로 자동 복귀 (FIX_REQUIRED 시)
- ESCALATED 발생 시에만 사람 개입
- 사람은 세션 시작과 ESCALATED 처리 외에는 루프에 개입하지 않는다

[설계 후퇴 감지 규칙]
제안이 다음 중 하나에 해당하면 후퇴안으로 분류한다:
- "우선 수동으로 시작하고 나중에 자동화"
- "사용자가 다음 에이전트에게 알려주면 됨"
- "초기엔 터미널 두 개 켜두고 교대로 실행"
- "일단 이렇게 해두고 나중에 개선"
```

---

## 2. 핵심 문제

MCP는 요청-응답 프로토콜이다. 에이전트가 서버를 호출하는 구조이며, 서버가 먼저 에이전트를 호출할 수 없다.

따라서 자동 턴 전환을 위해서는 MCP 서버 외에 **Turn Dispatcher**가 별도로 필요하다.

```
현재 구조 (불완전):
[Implementer] ──call──→ [Coordinator MCP]
[Coordinator MCP] ──?──→ [Verifier]  ← 이 경로가 없음

목표 구조:
[Implementer] ──call──→ [Coordinator MCP]
[Turn Dispatcher] ──감지──→ [Coordinator MCP 상태 변화]
[Turn Dispatcher] ──dispatch──→ [Verifier]
[Verifier] ──call──→ [Coordinator MCP]
[Turn Dispatcher] ──감지──→ ...반복
```

Turn Dispatcher가 이 공백을 메운다.

---

## 3. Runtime 후보

### Runtime-A: API Orchestrated

**구조:**
```
[사용자: 세션 시작]
        ↓
[Runner/Dispatcher]
  ├── Claude API 직접 호출 → Implementer 역할 수행
  ├── 응답 수신 → Coordinator MCP에 기록
  ├── OpenAI/Anthropic API 호출 → Verifier 역할 수행
  ├── 판정 수신 → Coordinator MCP에 기록
  └── 루프 반복
```

**특징:**
- Coordinator 또는 별도 Runner가 LLM API를 직접 호출
- 에이전트는 독립 프로세스가 아니라 API 호출 대상
- 인간 개입 없이 완전 헤드리스 실행 가능

---

### Runtime-B: Agent-Polling Orchestrated

**구조:**
```
[사용자: 세션 시작 + 역할 배정]
        ↓
[Agent A (Implementer) — 자율 폴링 루프]
  coordinator.get_my_task() → 작업 있으면 실행 → 결과 제출 → 반복

[Agent B (Verifier) — 자율 폴링 루프]
  coordinator.get_my_task() → 검증 대상 있으면 실행 → 판정 제출 → 반복
```

**특징:**
- 기존 Claude Code / Codex 에이전트를 그대로 활용
- 에이전트가 autonomous loop mode로 실행되어야 함
- Coordinator는 상태 관리만, 에이전트가 스스로 차례를 확인

**Option B 실현을 위한 전제 조건 (검증 필요):**
```
[ ] 에이전트가 unattended loop 실행을 지원하는가
    (사람 확인 없이 도구 호출을 반복 실행 가능한가)
[ ] 외부 상태(Coordinator MCP 응답)를 보고 스스로 다음 행동을 결정할 수 있는가
[ ] 장시간(수십 분~수 시간) 실행이 세션 종료 없이 안정적인가
[ ] 로컬 파일시스템/터미널 도구 접근이 루프 중에도 유지되는가
[ ] 폴링 간격이 실사용에서 허용 가능한 수준인가
    (너무 빠르면 API 비용, 너무 느리면 UX 저하)

전제 조건 미충족 시: Runtime-A로 전환
```

---

## 4. MVP 평가 기준표

두 옵션을 아래 기준으로 평가한다.
감으로 결정하지 않고, 제품 정체성 기준으로 선택한다.

| 평가 기준 | Runtime-A (API) | Runtime-B (Polling) | 가중치 |
|-----------|----------------|---------------------|--------|
| 자동화 순도 (사람 중계 없는 완전 자동) | ★★★★★ | ★★★★☆ (폴링 지원 여부에 의존) | 높음 |
| 기존 에이전트 재사용성 | ★★☆☆☆ (API 추상화 필요) | ★★★★★ (Claude Code 그대로) | 높음 |
| 로컬 도구 접근 자연스러움 | ★★☆☆☆ (별도 설계 필요) | ★★★★★ (에이전트 도구 그대로) | 높음 |
| 구현 복잡도 | ★★★☆☆ (Runner 구현) | ★★★☆☆ (폴링 루프 설계) | 중간 |
| 세션 안정성 | ★★★★☆ (API 안정성 의존) | ★★★☆☆ (장시간 루프 안정성) | 중간 |
| 턴 전환 레이턴시 | ★★★★★ (즉각 전환) | ★★★☆☆ (폴링 간격만큼 딜레이) | 중간 |
| VSCode 통합 자연스러움 | ★★★☆☆ (별도 연동 필요) | ★★★★★ (기존 환경 그대로) | 중간 |
| 프로젝트 정체성 부합 | ★★★☆☆ | ★★★★★ | 높음 |

**현재 기울기: Runtime-B 우선 검토, 전제 조건 미충족 시 Runtime-A**

---

## 5. Turn Dispatcher 설계 (공통)

두 옵션 모두 Turn Dispatcher가 필요하다. 역할은 동일하고 구현 방식만 다르다.

### Dispatcher 책임
```
1. Coordinator MCP 상태 감시
   - 현재 task의 state 확인
   - 담당 agent_role 확인

2. 차례 감지
   - state == IMPLEMENTING → Implementer 차례
   - state == VERIFYING → Verifier 차례
   - state == NEGOTIATING → 판정에 따라 라우팅
   - state == ESCALATED → 사람에게 알림

3. 차례 디스패치
   - Runtime-A: 해당 LLM API 호출 + 컨텍스트 주입
   - Runtime-B: 해당 에이전트 polling endpoint에 신호 / 에이전트가 자체 감지

4. 응답 수집
   - 결과를 Coordinator MCP에 기록
   - 다음 상태로 전이

5. 장애 처리
   - 에이전트 무응답 → PAUSED 전환 + 재시도
   - 예산 초과 → ESCALATED
   - 세션 복원 필요 → RESTORING 시작
```

### Dispatcher 상태 루프
```
loop:
  state = coordinator.get_session_state()

  if state == ESCALATED or state == DONE or state == PAUSED:
    notify_user()
    wait_for_instruction()
    continue

  next_role = coordinator.get_next_role()
  context = coordinator.build_context_for(next_role)

  if runtime == "A":
    response = call_llm_api(next_role.model, context)
  elif runtime == "B":
    signal_agent(next_role.agent_id)
    response = wait_for_agent_response(next_role.agent_id)

  coordinator.submit_response(next_role, response)
  coordinator.advance_state()
```

---

## 6. 초기 구현 순서

```
Phase 1: Core (Runtime 결정 전 공통 작업)
  - Coordinator MCP 서버 (DESIGN.md 기반)
  - SQLite 스키마
  - Turn Dispatcher 인터페이스 정의

Phase 2: Runtime 선택
  Option B 전제 조건 검증
  → 충족: Runtime-B 구현
  → 미충족: Runtime-A 구현

Phase 3: 실행 환경
  - CLI runner (coordinator start / status / resume)
  - 설정 파일 (.coordinator/project.json + local.json)

Phase 4: UX
  - VSCode 확장 (조종석 역할)
  - 세션 시각화
  - ESCALATED 개입 UI
```

---

## 7. 미결 항목 (RUNTIME.md v0.2에서 결정 필요)

1. **Option B 전제 조건 실제 검증**: Claude Code의 autonomous loop 지원 여부
2. **폴링 간격 설계**: Option B 선택 시 적정 폴링 주기
3. **컨텍스트 주입 크기 제한**: API 호출 시 context window 관리 방식
4. **에이전트 신원 확인**: Dispatcher가 에이전트를 어떻게 인증하는가
5. **Runner 프로세스 관리**: Dispatcher가 crash하면 어떻게 복구하는가

---

*v0.1 — 2026-04-16*
*다음 단계: Option B 전제 조건 검증 → Runtime 확정 → Phase 1 구현 시작*
