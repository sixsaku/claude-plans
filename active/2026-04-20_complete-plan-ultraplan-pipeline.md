# 실행 계획: 완전한 Plan Pipeline — 인터뷰 → 커스텀 Plan → 공식 Ultraplan
**작성일**: 2026-04-20 | **설계**: wing-upa (UPA 프로토콜) | **상태**: 승인 대기 | **CC**: 16

---

## SCQ 피라미드

**S (Situation):**
- 공식 `/ultraplan`: 클라우드 계획 수립 → 웹 리뷰 UI → 댓글/승인 → 구현. 강력하지만 컨텍스트 수집 없이 콜드 스타트.
- 커스텀 wing-upa: 맥락 수집(5문 주관식) + 구조적 설계도 생성. 웹 리뷰 없음.
- prometheus_engine.py: 10문항 선택형 state machine 완성. 두 시스템과 단절.

**C (Complication):**
두 시스템의 강점이 단절되어 있다. 공식 Ultraplan은 30분 클라우드 사고로 계획을 정교화하지만, 사전 맥락 없이 콜드 스타트한다. 커스텀 plan은 맥락을 잘 수집하지만 웹 리뷰 UI와 연결이 없다. 텔레그램에서는 `/ultraplan` 직접 실행이 불가능하다.

**Q (Solution):**
인터뷰 → 커스텀 plan → 로컬 plan-mode → "refine with Ultraplan" 경로로 연결한다. 텔레그램에서 발생한 계획은 GitHub push + 사용자 PC 알림으로 `/ultraplan` 실행을 안내한다.

---

## 완전한 프로세스 (목표)

```
[1단계] 트리거: "plan X" 요청 (터미널 or 텔레그램)
         ↓
[2단계] 인터뷰 Phase (prometheus_engine.py)
         10개 선택형 질문 → 맥락 수집
         텔레그램: 번호 선택 | CLI: AskUserQuestion
         ↓
[3단계] 커스텀 Plan 생성 (wing-upa 방법론)
         인터뷰 답변 → Logic Tree + WBS + 리스크 → MD 파일
         ↓
[4단계] 로컬 plan-mode 세션에 커스텀 plan 주입
         claude --plan "[커스텀 plan 전체 내용]"
         ↓
[5단계] plan 승인 다이얼로그 → "No, refine with Ultraplan"
         → 클라우드로 전송 (로컬 plan이 컨텍스트로 포함됨)
         ↓
[6단계] 공식 Ultraplan 클라우드 처리 (30분 extended thinking)
         웹 링크 → 브라우저에서 하이라이트 댓글 / 이모지 반응 / 아웃라인 검토
         ↓
[7단계] 승인 → "Implement here" or "Start new session"
         구현 시작
```

### 텔레그램 분기 (4~7단계)
```
[4단계] plan_push.py → GitHub push → GitHub URL
         텔레그램 알림: "계획 완성! 아래 링크 확인 후 터미널에서 /ultraplan 실행하세요"
         GitHub URL 전송
[5단계~] 사용자가 PC 터미널에서 직접 /ultraplan 실행
```

---

## Logic Tree

```
[완전한 Plan Pipeline]
│
├── A. 인터뷰 엔진 (prometheus_engine.py)
│   ├── A-1. --send-question: 번호+선택지 텍스트 포맷터
│   ├── A-2. --answer: 숫자 → suggestion 자동 변환
│   └── A-3. timeout (5분) + !interview-cancel 처리
│
├── B. 커스텀 Plan 생성기
│   ├── B-1. 인터뷰 답변 → wing-upa 방법론 적용
│   ├── B-2. 출력 형식: Ultraplan 친화적 MD (명확한 AC, WBS, 리스크)
│   └── B-3. plan_push.py → GitHub push → URL 수신
│
├── C. 로컬 plan-mode → Ultraplan 브릿지
│   ├── C-1. 커스텀 plan MD → 로컬 plan-mode 세션 주입 스크립트
│   ├── C-2. "refine with Ultraplan" 선택 안내 (자동화 불가 → 사용자 행동)
│   └── C-3. 텔레그램 분기: GitHub URL + /ultraplan 실행 안내 전송
│
├── D. 스킬 업데이트
│   ├── D-1. wing-plan: Step 0 (인터뷰) + Step N (Ultraplan 브릿지) 추가
│   └── D-2. wing-upa: 10문 선택형 전환 + Ultraplan 연계 명시
│
├── E. 라우팅 레이어
│   ├── E-1. intent_gate.py: 활성 인터뷰 감지 최우선
│   └── E-2. Magic Keyword: !interview-answer, !interview-cancel
│
└── F. 검증
    ├── F-1. 텔레그램 → GitHub URL 흐름 e2e
    └── F-2. CLI → /ultraplan 브릿지 e2e
```

---

## WBS — 서브태스크 (총 20개)

### Phase 0 — 백업 (3개, 최우선)
- [ ] **0-1.** `wing-plan/SKILL.md` → `.omc/state/backup_20260420/wing-plan_SKILL.md.bak`
- [ ] **0-2.** `wing-upa/SKILL.md` → `.omc/state/backup_20260420/wing-upa_SKILL.md.bak`
- [ ] **0-3.** `scripts/prometheus_engine.py` → `.omc/state/backup_20260420/prometheus_engine.py.bak`

### Phase 1 — 인터뷰 엔진 강화 (3개) ← 블로커
- [ ] **1-1.** `prometheus_engine.py --send-question` 추가
  - interview_state.json의 current_q 읽어 포맷:
    ```
    ❓ [N/10] {질문}

    1️⃣ {suggestion[0]}
    2️⃣ {suggestion[1]}
    3️⃣ {suggestion[2]}
    ✏️ 직접 입력 — 원하는 내용을 바로 답장
    ```
- [ ] **1-2.** `--answer` 숫자 변환: "1"/"2"/"3" → suggestion 텍스트 자동 변환
- [ ] **1-3.** timeout 처리: `last_activity` 타임스탬프 + 5분 초과 시 expired 반환

### Phase 2 — 커스텀 Plan 출력 형식 표준화 (2개)
- [ ] **2-1.** wing-upa 출력 MD에 Ultraplan 친화적 섹션 추가:
  - `## Acceptance Criteria` (Bash로 검증 가능한 명령어 형태)
  - `## Implementation Steps` (번호 순서형)
  - `## Context` (인터뷰 Q&A 요약 포함)
  → Ultraplan 클라우드가 로컬 plan을 컨텍스트로 받을 때 최대 활용
- [ ] **2-2.** plan_push.py 실행 자동화: plan 파일 생성 후 항상 push + GitHub URL 획득

### Phase 3 — 로컬 plan-mode → Ultraplan 브릿지 (3개)
- [ ] **3-1.** `scripts/plan_to_local_mode.py` 생성:
  ```python
  # 커스텀 plan MD → 로컬 plan-mode 세션 주입
  # claude --plan "다음 계획을 기반으로 plan-mode 실행: {plan_content}"
  # 사용자에게 "plan 승인 다이얼로그에서 'No, refine with Ultraplan' 선택하세요" 안내
  ```
- [ ] **3-2.** 텔레그램 분기 처리: 텔레그램에서 plan 완료 시:
  - GitHub URL 전송 (plan_push.py)
  - "✅ 계획 완성! PC 터미널에서 실행하세요:\n`/ultraplan` 또는 아래 명령어:\n`claude --plan [plan 내용]` → 승인 다이얼로그에서 Ultraplan 선택"
- [ ] **3-3.** `scripts/plan_push.py` 오류 처리 강화: GitHub push 실패 시 텔레그램 에러 보고

### Phase 4 — 라우팅 레이어 업데이트 (2개)
- [ ] **4-1.** `intent_gate.py` 최우선 인터뷰 감지 추가:
  - interview_state.json 존재 + completed=false + timeout 미초과 → `!interview-answer {msg}` 반환
- [ ] **4-2.** `general-tg.md` Magic Keyword 확장:
  - `!interview-answer <답변>`: answer → 다음 Q or approve → plan 생성 시작
  - `!interview-cancel`: state 삭제 + 취소 안내

### Phase 5 — wing-plan SKILL.md 업데이트 (2개)
- [ ] **5-1.** Step 0 (Interview Phase) 추가 (Step 1 앞에):
  ```
  [Telegram] prometheus --generate → send-question → 답변 수집 (intent_gate 자동 라우팅)
  [CLI]      AskUserQuestion 10문 순차 → state 동기화
  완료 후 Step 1 진행
  ```
- [ ] **5-2.** Step N (Ultraplan Bridge) 추가 (마지막):
  ```
  plan MD 생성 완료 →
  [CLI]      plan_to_local_mode.py 실행 → "refine with Ultraplan" 선택 안내
  [Telegram] plan_push.py → GitHub URL + /ultraplan 실행 안내 전송
  ```

### Phase 6 — wing-upa SKILL.md 업데이트 (2개)
- [ ] **6-1.** 1단계 질문 방식 교체: 5문 주관식 → prometheus 10문 선택형 연동
- [ ] **6-2.** 설계도 출력 형식에 Ultraplan 친화적 AC 섹션 추가 (Phase 2-1 기준)

### Phase 7 — CLI AskUserQuestion 경로 (1개)
- [ ] **7-1.** wing-plan SKILL.md CLI 분기: AskUserQuestion 10문 + state 동기화 + 선택지 25자 이내

### Phase 8 — 검증 (2개)
- [ ] **8-1.** 텔레그램 e2e: "todo 앱 plan" → 10문답 → GitHub URL + /ultraplan 실행 안내 확인
- [ ] **8-2.** CLI e2e: "todo 앱 plan" → AskUserQuestion 10문 → plan MD → plan_to_local_mode.py → Ultraplan 진입 확인

---

## 의존성 맵

```
Phase 0 (백업)
    ↓
Phase 1 (인터뷰 엔진) ─────────────────────────────┐
    ↓                                               │
Phase 2 (plan 출력 표준화) → Phase 3 (Ultraplan 브릿지)  
Phase 4 (라우팅)                                    │
Phase 5 (wing-plan) ──────────────────────────────→ Phase 8 (검증)
Phase 6 (wing-upa)  ──────────────────────────────→
Phase 7 (CLI AskUserQ) ───────────────────────────→
```

Phase 1 + Phase 2 가 블로커. 완료 후 3~7 병렬 진행 가능.

---

## 전제 조건

1. Claude Code v2.1.91 이상 설치됨 (Ultraplan 요구 사항)
2. Pro/Max 구독 + Git 저장소 조건 충족됨
3. prometheus_engine.py 기존 인터페이스 하위 호환 유지
4. 텔레그램에서 `/ultraplan` 직접 실행 불가 → GitHub URL + 수동 터미널 실행 안내로 대응

---

## 리스크 분석

| 단계 | 리스크 | 심각도 | 대응 |
|------|--------|--------|------|
| 3-1 plan_to_local_mode.py | plan 내용이 너무 길어 plan-mode 컨텍스트 초과 | MED | plan 내용을 2000자 이내로 요약한 버전도 함께 생성 |
| 4-1 intent_gate 수정 | 기존 라우팅 오작동 | HIGH | Phase 0 백업 + 회귀 테스트 |
| 3-2 텔레그램 분기 | 사용자가 /ultraplan 실행 안 할 수 있음 | LOW | GitHub URL만으로도 검토 가능 (fallback) |
| 1-3 timeout | 5분 안에 답변 못 하면 인터뷰 종료 | LOW | !interview-cancel 재시작 안내 |

---

## 복구 계획

```bash
cp .omc/state/backup_20260420/wing-plan_SKILL.md.bak .claude/skills/wing-plan/SKILL.md
cp .omc/state/backup_20260420/wing-upa_SKILL.md.bak .claude/skills/wing-upa/SKILL.md
cp .omc/state/backup_20260420/prometheus_engine.py.bak scripts/prometheus_engine.py
```

---

## MECE 자가검증

- ✅ 인터뷰 → 커스텀 plan → /ultraplan 전체 파이프라인 커버
- ✅ 텔레그램 분기 (GitHub URL + 터미널 안내) 명확
- ✅ CLI 분기 (AskUserQuestion + plan_to_local_mode.py + "refine with Ultraplan")
- ✅ Ultraplan 친화적 출력 포맷 표준화 포함
- ✅ 백업 → 구현 → 검증 순서 명확
- ✅ 기존 !interview 명령어와 공존 (동일 state machine)
- ⚠️ plan_to_local_mode.py가 "refine with Ultraplan" 선택을 자동화할 수 없음 → 사용자 행동 필요 (현재 Ultraplan API로 직접 주입 방법 미공개)
## [APPROVER_DIRECTIVE] — ultraplan 승인자 관점 분석 요청

이 계획을 최종 승인자(Plan Approver) 관점에서 다음 5가지 축으로 분석해주세요:

1. **논리 비약 검출**: "A이므로 B" 형태의 검증되지 않은 전제가 있는가?
2. **전제 조건 취약점**: 이 계획이 성립하려면 무엇이 전제되어야 하는가? 그 전제가 무너질 수 있는가?
3. **누락 리스크**: 계획에서 다루지 않은 실패 시나리오는 무엇인가?
4. **성공 기준 명확성**: AC(완료 기준)가 객관적으로 측정 가능한가? 주관적 표현이 있는가?
5. **대안 경로**: 현재 접근법보다 더 단순하거나 더 안전한 대안이 있는가?

분석 후 "승인" 또는 "수정 필요 + 구체적 수정 지점" 형태로 최종 판정을 내려주세요.
