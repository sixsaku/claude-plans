# 실행 계획: PM 브리핑 체계 재설계 — 노션 → GitHub 기반
**작성일**: 2026-04-20 | **상태**: 승인 대기 | **CC**: 12

## 목표
노션 의존성 완전 제거. GitHub plans 폴더 + git log 기반 AI 브리핑 자동화.

## 벤치마킹 근거
- Stevens 패턴 (geoffreylitt.com): SQLite + cron + LLM + Telegram = 단순 아키텍처가 베스트
- DailyBot 패턴: Git 커밋 로그 + LLM 요약 → 알림
- GitHub Docs: plans/MD = 단일 진실 소스, Telegram = 뷰 레이어

## 목표 아키텍처
```
[plans/active/*.md] + [git log from repos]
        ↓
  pm_briefing.py (Claude Haiku)
        ↓
  Telegram 브리핑
        ↓
  plans/daily/YYYY-MM-DD.md → GitHub push
```

## plans 폴더 구조
```
.claude/knowledge/plans/
  active/    ← 진행 중 계획
  daily/     ← 매일 브리핑 자동 저장
  weekly/    ← 주간 브리핑
  archive/   ← 완료된 계획
```

## 서브태스크 (총 13개)

### Phase 0 — 설계
- [ ] 1. plans 폴더 구조 생성 (active/daily/weekly/archive)
- [ ] 2. Haiku 브리핑 프롬프트 설계

### Phase 1 — 스크립트
- [ ] 3. scripts/pm_briefing.py — daily 브리핑 (plans 읽기 + git log + Haiku 요약 + Telegram)
- [ ] 4. scripts/pm_weekly.py — weekly 브리핑
- [ ] 5. scripts/plan_push.py — plans 파일 GitHub 자동 push + Telegram URL 알림

### Phase 2 — jobs.json 업데이트
- [ ] 6. daily-pm-watchdog → pm_briefing.py
- [ ] 7. weekly-pm-briefing → pm_weekly.py

### Phase 3 — plan 자동 저장 파이프라인
- [ ] 8. wing-plan 스킬: 계획 완료 시 plans/active/ 자동 저장
- [ ] 9. plan_push.py 훅 연결

### Phase 4 — 노션 정리
- [ ] 10. jobs.json 노션 참조 제거
- [ ] 11. CLAUDE.md 노션 베이스캠프 페이지 ID 참조 제거

### Phase 5 — 검증
- [ ] 12. pm_briefing.py 수동 실행 확인
- [ ] 13. pm_weekly.py 수동 실행 확인

## 리스크
| 단계 | 리스크 | 대응 |
|------|--------|------|
| 3 | Haiku API 비용 | 입력 토큰 2000자 제한 |
| 3 | repo 목록 하드코딩 | 설정 파일로 관리 |
| 8 | 스킬 동작 변경 | 저장 실패 시 silent fail |
| 11 | CLAUDE.md 수정 | 노션 관련 줄만 최소 제거 |
## [APPROVER_DIRECTIVE] — ultraplan 승인자 관점 분석 요청

이 계획을 최종 승인자(Plan Approver) 관점에서 다음 5가지 축으로 분석해주세요:

1. **논리 비약 검출**: "A이므로 B" 형태의 검증되지 않은 전제가 있는가?
2. **전제 조건 취약점**: 이 계획이 성립하려면 무엇이 전제되어야 하는가? 그 전제가 무너질 수 있는가?
3. **누락 리스크**: 계획에서 다루지 않은 실패 시나리오는 무엇인가?
4. **성공 기준 명확성**: AC(완료 기준)가 객관적으로 측정 가능한가? 주관적 표현이 있는가?
5. **대안 경로**: 현재 접근법보다 더 단순하거나 더 안전한 대안이 있는가?

분석 후 "승인" 또는 "수정 필요 + 구체적 수정 지점" 형태로 최종 판정을 내려주세요.
