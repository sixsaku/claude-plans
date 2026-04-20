# 실행계획: /ultraplan 최종 승인자 관점 파이프라인 구현

**작성일:** 2026-04-20  
**방법론:** Wing-UPA (Logic Tree + WBS + MECE)  
**맥락 수집:** prometheus_engine.py 10문 인터뷰 (명확도 90점)  
**검토 예정:** 공식 /ultraplan (클라우드 Extended Thinking)

---

## SCQ 피라미드

**S (Situation):** wing-plan/wing-upa는 prometheus_engine.py로 10문 인터뷰를 수집하고 커스텀 실행계획을 생성한다. 공식 `/ultraplan`은 클라우드 Extended Thinking으로 계획을 심층 분석하고 PC앱/모바일앱 웹 UI로 검토 가능하다.

**C (Complication):** `/ultraplan`이 단순히 계획을 "승인"하는 것을 넘어 **최종 승인자(Plan Approver)** 관점 — 논리 결함, 리스크, 전제 조건, 성공 기준 불명확성 — 으로 다각도 분석을 수행할 수 있는지 불분명하다. 또한 인터뷰 → 커스텀 계획 → /ultraplan 검토 → 승인 → 실행의 완전한 파이프라인이 없다.

**Q (Solution):** `/ultraplan`에 "최종 승인자 관점 분석 지시"를 커스텀 계획 내 삽입하여 다각적 검토를 유도하고, 인터뷰 → 계획 → /ultraplan 검토 → 승인 → 실행의 End-to-End 파이프라인을 구현한다.

---

## 목표 및 성공 기준

**목표:** `/ultraplan`이 커스텀 계획을 최종 승인자 관점에서 분석하도록 유도하는 프롬프트 구조 + 완전한 파이프라인 구현

**성공 기준 (AC — 모두 충족 필요):**
- [ ] AC1: `wing-plan` 또는 `wing-upa` 실행 시 prometheus 10문 인터뷰가 자동 발동됨
- [ ] AC2: 인터뷰 완료 후 커스텀 계획 MD가 자동 생성되고 GitHub에 push됨
- [ ] AC3: 계획 MD에 `## [APPROVER_DIRECTIVE]` 섹션이 포함됨
  - 검증: `grep -l "APPROVER_DIRECTIVE" .claude/knowledge/plans/active/*.md`
- [ ] AC4: CLI에서 `plan_to_local_mode.py` 실행 시 `/ultraplan` 브릿지 안내가 출력됨
  - 검증: `python scripts/plan_to_local_mode.py <plan_path> | grep -c "ultraplan"` (≥1 이면 PASS)
- [ ] AC5: 텔레그램으로 GitHub URL + `/ultraplan` 사용법 안내 전송됨
- [ ] AC6: `/ultraplan` 웹 UI에서 승인자 관점 분석 결과 확인 가능
  - 검증: `ls .claude/knowledge/plans/ultraplan_results/*.md 2>/dev/null | wc -l` (≥1 이면 PASS)
- [ ] AC7: 텔레그램 인터뷰 중 다른 메시지가 오면 `interview_state.json` 타임아웃(5분) 처리됨
- [ ] AC8: `!interview-cancel` 명령으로 인터뷰 중단 및 상태 초기화됨

**환경:** 로컬 Windows D:/claude_official  
**수준:** 풀 버전 (모든 엣지 케이스 처리)  
**사용자:** 상우님 혼자  

---

## Logic Tree (MECE 분해)

```
[ultraplan 최종 승인자 파이프라인]
├── A. /ultraplan 승인자 관점 유도 방법론
│   ├── A-1. /ultraplan 동작 방식 확인 (Extended Thinking 입력 구조)
│   ├── A-2. 승인자 관점 프롬프트 섹션 설계
│   │   ├── A-2-1. 다각 분석 요청 템플릿 (논리결함/리스크/전제/성공기준)
│   │   └── A-2-2. 커스텀 계획 MD에 섹션 삽입 위치 결정
│   └── A-3. 프롬프트 효과 검증 (수동 테스트)
│
├── B. 인터뷰 → 계획 자동 생성 파이프라인
│   ├── B-1. prometheus_engine.py 개선
│   │   ├── B-1-1. --send-question 포맷터 (이미 구현됨 ✅)
│   │   └── B-1-2. 타임아웃/만료 처리 (이미 구현됨 ✅)
│   ├── B-2. intent_gate.py 활성 인터뷰 감지 (이미 구현됨 ✅)
│   ├── B-3. 인터뷰 완료 → 계획 자동 생성 트리거
│   │   ├── B-3-1. prometheus --approve 후 wing-upa/wing-plan 자동 호출
│   │   └── B-3-2. 계획 MD에 인터뷰 답변 요약 자동 삽입
│   └── B-4. 엣지 케이스 처리
│       ├── B-4-1. 인터뷰 중 다른 메시지 → 인터뷰-answer 라우팅 (이미 구현됨 ✅)
│       ├── B-4-2. 5분 타임아웃 → expired 플래그 + 텔레그램 알림
│       ├── B-4-3. !interview-cancel → 상태 초기화 (이미 구현됨 ✅)
│       └── B-4-4. 중복 인터뷰 방지 (진행 중 !plan 재호출 시 기존 인터뷰 재개)
│
├── C. /ultraplan 브릿지 구현
│   ├── C-1. plan_to_local_mode.py 완성 (이미 구현됨 ✅)
│   ├── C-2. plan_push.py → GitHub URL → 텔레그램 전송
│   │   └── C-2-1. /ultraplan 사용 안내 메시지 표준화
│   └── C-3. wing-plan/wing-upa SKILL.md 최종 통합 (이미 구현됨 ✅)
│
└── D. 검증 및 테스트
    ├── D-1. CLI 시나리오 테스트 (wing-plan "테스트 주제" 실행)
    ├── D-2. 텔레그램 시나리오 테스트 (이번 세션이 라이브 테스트)
    ├── D-3. /ultraplan 승인자 분석 결과 수동 검토
    └── D-4. 엣지 케이스 자동화 테스트 (타임아웃, 취소, 중복)
```

---

## 서브태스크 (총 22개)

### Phase 0: 현황 확인 (이미 완료된 것 체크)
- [x] 0-1. prometheus_engine.py --send-question, --answer digit변환, timeout 구현
- [x] 0-2. intent_gate.py 활성 인터뷰 감지 Priority 0 추가
- [x] 0-3. general-tg.md !interview-answer, !interview-cancel Magic Keyword 추가
- [x] 0-4. wing-plan SKILL.md Step 0 추가 (Telegram/CLI 분기)
- [x] 0-5. wing-upa SKILL.md 10문 선택형으로 업데이트
- [x] 0-6. plan_to_local_mode.py 생성

### Phase 1: /ultraplan 승인자 관점 프롬프트 설계 ← 핵심 신규 작업
- [ ] 1-1. 커스텀 계획 MD에 삽입할 "승인자 관점 분석 요청" 섹션 템플릿 작성
  - 논리 비약 검출 요청
  - 전제 조건 검증 요청
  - 누락 리스크 도출 요청
  - 성공 기준 명확성 검증 요청
  - 대안 경로 제안 요청
- [ ] 1-2. plan_generator.py 또는 wing-upa 계획 생성 흐름에 섹션 자동 삽입 로직 추가
- [ ] 1-3. /ultraplan 실제 테스트 (이번 계획 파일로 직접 검증)

### Phase 2: 인터뷰 완료 → 계획 자동 생성 자동화
- [ ] 2-1. !interview-answer completed=true 시 wing-upa 자동 호출 로직 확인/보완
  - general-tg.md의 현재 규칙: "→ wing-plan Step 1 진행" — 실제로 wing-upa가 계획 작성 시작하는지 검증
- [ ] 2-2. 계획 MD 생성 시 prometheus_report.json 답변 요약 자동 포함
- [ ] 2-3. plan_push.py 실행 후 텔레그램 메시지 표준 포맷 확정

### Phase 3: 엣지 케이스 완성
- [ ] 3-1. 타임아웃(5분) 만료 시 텔레그램 "인터뷰가 만료됐습니다. 다시 시작하려면 !plan" 알림
  - 현재: expired 플래그 설정됨 → 알림 발송 로직 미구현
- [ ] 3-2. 중복 !plan 호출 시 기존 인터뷰 재개 처리
  - 현재: 새 인터뷰로 덮어씀 → 진행 중 인터뷰 있으면 "이미 진행 중" 안내 추가
- [ ] 3-3. 인터뷰 완료 후 prometheus_report.json 5분 캐시 — 재실행 시 스킵 처리 확인

### Phase 4: 검증
- [ ] 4-1. 이번 세션 라이브 테스트 결과 기록 (텔레그램 인터뷰 → 계획 생성 흐름)
- [ ] 4-2. CLI에서 wing-plan 실행 → AskUserQuestion 10문 → 계획 생성 테스트
- [ ] 4-3. /ultraplan 웹 UI에서 승인자 관점 분석 결과 스크린샷 캡처
- [ ] 4-4. 타임아웃 시나리오 수동 테스트 (5분 대기 후 메시지 전송)

---

## 의존성

```
Phase 0 (완료) → Phase 1-1 → Phase 1-2 → Phase 1-3
                                          ↘ Phase 2-1 → Phase 2-2 → Phase 2-3
Phase 1-1 → Phase 3-1
Phase 3-1 + Phase 3-2 + Phase 3-3 → Phase 4-1
Phase 4-1 + Phase 4-2 → Phase 4-3 → Phase 4-4
```

**병렬 실행 가능:**
- Phase 1-1 + Phase 3-2 (독립적)
- Phase 4-1 + Phase 4-2 (독립적)

---

## 리스크

| 단계 | 리스크 | 롤백/대응 |
|------|--------|-----------|
| 1-3 | /ultraplan이 승인자 관점 프롬프트를 무시하고 일반 계획 분석만 수행 | 프롬프트 강화 (더 명시적인 지시), 여러 버전 테스트 |
| 2-1 | completed=true 후 wing-upa 자동 호출이 작동 안 함 | general-tg.md 규칙을 Bash 명령으로 명시화 |
| 3-1 | 5분 만료 후 텔레그램 알림이 세션 없을 때 발송 안 됨 | CronCreate로 만료 감지 잡 등록 |
| 1-2 | 계획 MD 자동 생성 시 승인자 섹션이 누락됨 | 계획 생성 함수에 고정 섹션 삽입 |

---

## 전제 조건

1. Claude Code v2.1.91+ 설치됨 (/ultraplan 지원 버전)
2. plan_push.py가 GitHub에 push할 수 있는 인증 상태 유지
3. prometheus_engine.py가 D:/claude_official/.omc/state/에 읽기/쓰기 가능
4. 텔레그램 채널 플러그인이 세션에 연결된 상태

---

## MECE 자가검증

- **중복 없음:** Phase 0 (기구현), Phase 1 (승인자 프롬프트), Phase 2 (자동화), Phase 3 (엣지케이스), Phase 4 (검증) — 각 영역 독립적
- **누락 없음:** 승인자 관점 프롬프트 ✅ / 파이프라인 자동화 ✅ / 엣지케이스 ✅ / 검증 ✅
- **전제 조건 현실성:** /ultraplan은 계획 텍스트 전체를 입력으로 받으므로 프롬프트 삽입 방식은 현실적
- **모호성:** "승인자 관점 분석"이 /ultraplan에서 실제로 어떻게 나타나는지는 테스트 전까지 불확실 → Phase 1-3에서 검증 필수

---

## 실행 순서

```
Phase 1-1 (승인자 프롬프트 템플릿 설계)
→ Phase 1-2 (계획 생성 흐름에 삽입)
→ Phase 2-1 (자동 호출 보완)
→ Phase 3-1, 3-2 (엣지케이스)
→ Phase 1-3 + Phase 4-1 (이번 계획으로 /ultraplan 라이브 테스트)
→ Phase 4-2, 4-3, 4-4 (종합 검증)
```

---

## [APPROVER_DIRECTIVE] — ultraplan 승인자 관점 분석 요청

> **이 계획을 /ultraplan에게 제출할 때 아래 지시를 포함할 것:**

```
이 계획을 최종 승인자(Plan Approver) 관점에서 다음 5가지 축으로 분석해주세요:

1. **논리 비약 검출**: "A이므로 B" 형태의 검증되지 않은 전제가 있는가?
2. **전제 조건 취약점**: 이 계획이 성립하려면 무엇이 전제되어야 하는가? 그 전제가 무너질 수 있는가?
3. **누락 리스크**: 계획에서 다루지 않은 실패 시나리오는 무엇인가?
4. **성공 기준 명확성**: AC(완료 기준)가 객관적으로 측정 가능한가? 주관적 표현이 있는가?
5. **대안 경로**: 현재 접근법보다 더 단순하거나 더 안전한 대안이 있는가?

분석 후 "승인" 또는 "수정 필요 + 구체적 수정 지점" 형태로 최종 판정을 내려주세요.
```

---

## 다음 즉시 실행 항목

1. 이 계획 파일 → `plan_push.py`로 GitHub push → 텔레그램 URL 전송
2. PC 터미널에서: `/ultraplan` 실행 또는 `claude --plan "위 계획 내용"` 후 Ultraplan 선택
3. /ultraplan 웹 UI에서 승인자 관점 분석 확인
4. Phase 1-1 ~ 1-2 구현 (승인자 프롬프트 자동 삽입 로직)
