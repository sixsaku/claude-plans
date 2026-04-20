# 실행 계획: wing-plan/UPA 선택형 질문 단계 통합 (범용)
**작성일**: 2026-04-20 | **설계**: wing-upa (UPA 프로토콜) | **상태**: 승인 대기 | **CC**: 14

---

## SCQ 요약

- **S**: prometheus_engine.py에 10문항 state machine 완성. wing-plan은 질문 없음, wing-upa는 5문 주관식.
- **C**: 두 시스템이 단절. CLI(AskUserQuestion)와 Telegram(비동기 reply) I/O 패러다임 불일치.
- **Q**: interview_state.json을 공통 인터페이스로 삼고 렌더링 방식만 컨텍스트별 분기.

---

## 핵심 아키텍처 결정

### AskUserQuestion을 Telegram에서 직접 사용 가능한가?
**불가 (현재).** AskUserQuestion은 터미널 stdin 블로킹 I/O. Telegram은 비동기 네트워크 이벤트. 직접 연결 불가.

**Hook IPC 브릿지(미래 고도화)**: AskUserQuestion hook → Telegram 전송 → `.omc/state/aq_response.json` 폴링 → Claude가 파일에 답변 기록 → hook 반환. 구현 가능하지만 race condition 리스크 있어 **Phase 7 (미래)** 로 분류.

**이번 구현 표준**: `interview_state.json` 기반 비동기 state machine. CLI에서는 AskUserQuestion 병렬 지원.

---

## 최종 사용자 경험 (목표 UX)

### 텔레그램
```
[User]  todo 앱 plan 세워줘

[Claude] 🎯 계획 수립 전 맥락을 수집합니다. 10개 질문에 번호로 답하거나 직접 입력하세요.
         취소: !interview-cancel

❓ [1/10] 이 프로젝트에서 가장 핵심 기능 하나는?

1️⃣ 사용자 인증 및 권한 관리
2️⃣ 할일 CRUD + 마감일 알림
3️⃣ 팀 협업 + 공유 기능
✏️ 직접 입력 — 원하는 내용을 바로 답장

[User]  2

[Claude] ❓ [2/10] 완성 시 사용자가 처음으로 할 수 있게 되는 행동은? ...
```

### CLI (터미널/데스크탑)
```
AskUserQuestion UI — 라디오 버튼 10문 순차 표시 + 자유입력 옵션
각 답변이 prometheus state에 동기화됨
```

---

## Logic Tree

```
[목표: wing-plan/upa 범용 10문 선택형 인터뷰 통합]
│
├── A. 인터뷰 엔진 강화 (prometheus_engine.py)
│   ├── A-1. --send-question: 현재 Q를 Telegram용 번호 텍스트 포맷으로 출력
│   ├── A-2. --answer: 숫자("1"~"3") → suggestion 자동 변환
│   └── A-3. timeout 감지: started_at 기준 5분 초과 시 expired 플래그
│
├── B. 라우팅 레이어 (intent_gate.py)
│   ├── B-1. 최우선: interview_state.json 존재+미완료 → !interview-answer 반환
│   └── B-2. timeout 체크 → expire + 안내
│
├── C. Magic Keyword 확장
│   ├── C-1. !interview-answer <답변>: answer → 다음 Q 전송 or approve
│   └── C-2. !interview-cancel: state 삭제 + 취소 안내
│
├── D. 스킬 업데이트
│   ├── D-1. wing-plan: Step 0 (Interview Phase) 추가
│   └── D-2. wing-upa: 5문 주관식 → 10문 선택형 전환
│
├── E. CLI 전용 (AskUserQuestion)
│   └── E-1. 컨텍스트 감지 → AskUserQuestion 10문 + state 동기화
│
└── F. 검증
    ├── F-1. Telegram e2e
    └── F-2. CLI e2e
```

---

## WBS — 서브태스크 (총 18개)

### Phase 0 — 백업 (3개)
- [ ] **0-1.** `wing-plan/SKILL.md` → `.omc/state/backup_20260420/wing-plan_SKILL.md.bak`
- [ ] **0-2.** `wing-upa/SKILL.md` → `.omc/state/backup_20260420/wing-upa_SKILL.md.bak`
- [ ] **0-3.** `scripts/prometheus_engine.py` → `.omc/state/backup_20260420/prometheus_engine.py.bak`

### Phase 1 — prometheus_engine.py 강화 (3개) ← 핵심 블로커
- [ ] **1-1.** `--send-question` 옵션 추가
  - interview_state.json의 current_q를 읽어 아래 포맷으로 텍스트 반환:
    ```
    ❓ [N/10] {질문}

    1️⃣ {suggestion[0]}
    2️⃣ {suggestion[1]}
    3️⃣ {suggestion[2]}
    ✏️ 직접 입력 — 원하는 내용을 바로 답장
    ```
- [ ] **1-2.** `--answer` 숫자 변환 처리 추가
  - 입력이 "1"/"2"/"3" → 해당 suggestion 텍스트로 자동 변환 후 저장
  - 그 외 → 자유 입력으로 처리
- [ ] **1-3.** timeout 필드 추가
  - `interview_state.json`에 `last_activity` 타임스탬프 기록
  - `--send-question` 호출 시 5분 초과 체크 → `{"expired": true}` 반환

### Phase 2 — intent_gate.py 업데이트 (2개)
- [ ] **2-1.** **최우선 순위** 인터뷰 상태 감지 추가 (룰 기반 패턴 앞에 삽입):
  ```python
  # 인터뷰 활성 감지 — 모든 패턴보다 먼저 체크
  if INTERVIEW_PATH.exists():
      state = json.loads(INTERVIEW_PATH.read_text())
      if not state.get("completed") and not state.get("expired"):
          last = state.get("last_activity", state.get("started_at"))
          if (now - parse(last)).seconds < 300:
              return "!interview-answer " + msg  # 그대로 답변으로 라우팅
          else:
              INTERVIEW_PATH.unlink()  # timeout expire
  ```
- [ ] **2-2.** `!interview-cancel` 처리 추가 (룰 기반 패턴에 등록)

### Phase 3 — Magic Keyword Bridge 확장 (2개, general-tg.md)
- [ ] **3-1.** `!interview-answer <답변>` 키워드 추가:
  - `prometheus_engine.py --answer {current_q} "<답변>"` 실행
  - 결과 `completed: false` → `--send-question` 실행 → Telegram 다음 질문 전송
  - 결과 `completed: true` → `--approve` 실행 → "✅ 맥락 수집 완료! 계획 수립 시작…" 전송 → wing-plan Step 2 진행
- [ ] **3-2.** `!interview-cancel` 키워드 추가:
  - `interview_state.json` 삭제
  - "인터뷰를 취소했습니다. 일반 대화로 복귀합니다." 전송

### Phase 4 — wing-plan SKILL.md 업데이트 (2개)
- [ ] **4-1.** Step 0 (Interview Phase) 삽입 (Step 1 앞에):
  ```markdown
  ### Step 0: 맥락 수집 인터뷰 (자동)
  계획 수립 전 반드시 prometheus_engine.py로 10개 선택형 질문을 실행한다.
  
  [Telegram 컨텍스트 — <channel> 태그 존재 시]
  1. `prometheus_engine.py --generate "주제"` 실행
  2. `prometheus_engine.py --send-question` 결과를 Telegram reply로 전송
  3. 이후 답변은 intent_gate → !interview-answer로 자동 라우팅됨
  4. 10문 완료 + --approve 후 Step 1 진행
  
  [CLI 컨텍스트 — <channel> 태그 없음]
  AskUserQuestion으로 10문 순차 표시 (E-1 참고)
  ```
- [ ] **4-2.** "Step 0 생략 조건" 명시:
  - prometheus_report.json이 이미 존재하고 5분 이내인 경우
  - 사용자가 "질문 없이 바로 plan" 명시한 경우

### Phase 5 — wing-upa SKILL.md 업데이트 (2개)
- [ ] **5-1.** 1단계 질문 방식 교체:
  - 기존: "5가지 날카로운 질문을 직접 작성하여 답을 기다린다"
  - 변경: "prometheus_engine.py --generate로 10개 선택형 질문 실행 (wing-plan Step 0와 동일)"
- [ ] **5-2.** "질문 생략 가능" 조건 유지 (이미 충분한 정보가 있을 때)

### Phase 6 — CLI AskUserQuestion 경로 (2개)
- [ ] **6-1.** wing-plan SKILL.md CLI 분기 상세 작성:
  - 10개 질문을 AskUserQuestion으로 순차 표시
  - 선택지는 25자 이내 (truncation 방지 — Issue #39867 대응)
  - 각 답변 → `prometheus_engine.py --answer N "답변"` 동기화
- [ ] **6-2.** AskUserQuestion 동결 버그 대응 안내 추가:
  - 동결 시: Ctrl+C 후 `!prometheus-ok` (force-pass) 사용 안내

### Phase 7 (미래) — Hook IPC 브릿지 (설계만, 구현 보류)
- [ ] **7-1.** 설계 문서만 작성: AskUserQuestion hook → Telegram 전송 → aq_response.json 폴링 → 응답 반환 구조

### Phase 8 — 검증 (2개)
- [ ] **8-1.** Telegram e2e: "간단한 todo 앱 plan 세워줘" → 10문답 → 계획 생성 확인
- [ ] **8-2.** CLI 터미널: 동일 요청 → AskUserQuestion UI 10문 표시 확인

---

## 의존성 맵

```
Phase 0 (백업) ──┐
                 ↓
Phase 1 (prometheus 강화) ──────────────────────────┐
         ↓                                           │
Phase 2 (intent_gate)    ──→ Phase 3 (keyword)      │
Phase 4 (wing-plan)      ──→ Phase 5 (wing-upa)     ↓
Phase 6 (CLI AskUserQ)                        Phase 8 (검증)
```

Phase 1이 핵심 블로커. 완료 후 2~6은 독립적으로 진행 가능.

---

## 전제 조건 (Assumptions)

1. `prometheus_engine.py`의 기존 state machine은 변경 없이 확장만 한다 (하위 호환 유지).
2. intent_gate.py의 인터뷰 감지는 최우선 순위로 삽입 (기존 라우팅 규칙보다 앞).
3. CLI 환경에서 AskUserQuestion 사용 시 세션이 터미널 모드여야 함 (비대화형 파이프 세션 제외).
4. `!interview-cancel` 을 언제든 보내면 인터뷰 상태가 즉시 초기화됨.

---

## 리스크 분석

| 단계 | 리스크 | 심각도 | 대응 |
|------|--------|--------|------|
| 2-1 intent_gate 수정 | 기존 라우팅 오작동 | HIGH | Phase 0 백업 + 모든 !키워드 회귀 테스트 |
| 3-1 인터뷰 중 다른 메세지 | 엉뚱한 메세지를 답변으로 처리 | MED | timeout 5분 + !interview-cancel 명확 안내 |
| 6-1 AskUserQuestion 동결 | 터미널 무응답 | MED | force-pass fallback 안내 명시 |
| 1-1~1-3 prometheus 확장 | 기존 --generate/--answer 동작 변경 | LOW | 새 옵션만 추가, 기존 코드 건드리지 않음 |

---

## 복구 계획

Phase 0 백업 파일을 원본 경로에 복사하면 즉시 롤백 가능.
```bash
cp .omc/state/backup_20260420/wing-plan_SKILL.md.bak .claude/skills/wing-plan/SKILL.md
cp .omc/state/backup_20260420/wing-upa_SKILL.md.bak .claude/skills/wing-upa/SKILL.md
cp .omc/state/backup_20260420/prometheus_engine.py.bak scripts/prometheus_engine.py
```

---

## MECE 자가검증

- ✅ Telegram 경로 커버됨 (interview_state.json + intent_gate)
- ✅ CLI 경로 커버됨 (AskUserQuestion + state 동기화)
- ✅ 인터뷰 취소/timeout 커버됨
- ✅ 기존 !interview 명령어 공존 (동일 state machine 사용, 충돌 없음)
- ✅ AskUserQuestion Telegram 불가 이유 명문화 + 미래 고도화 경로 제시
- ✅ 백업 → 구현 → 검증 순서 명확
- ⚠️ wing-plan 트리거 조건이 애매할 수 있음 → "질문 생략 조건" 명시로 대응
## [APPROVER_DIRECTIVE] — ultraplan 승인자 관점 분석 요청

이 계획을 최종 승인자(Plan Approver) 관점에서 다음 5가지 축으로 분석해주세요:

1. **논리 비약 검출**: "A이므로 B" 형태의 검증되지 않은 전제가 있는가?
2. **전제 조건 취약점**: 이 계획이 성립하려면 무엇이 전제되어야 하는가? 그 전제가 무너질 수 있는가?
3. **누락 리스크**: 계획에서 다루지 않은 실패 시나리오는 무엇인가?
4. **성공 기준 명확성**: AC(완료 기준)가 객관적으로 측정 가능한가? 주관적 표현이 있는가?
5. **대안 경로**: 현재 접근법보다 더 단순하거나 더 안전한 대안이 있는가?

분석 후 "승인" 또는 "수정 필요 + 구체적 수정 지점" 형태로 최종 판정을 내려주세요.
