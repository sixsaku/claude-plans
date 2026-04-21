# Wing-FM 퀀트 진화 계획 + Plan 질문 품질 개선
**작성일**: 2026-04-21 | **CC**: 17 | **설계**: wing-upa | **상태**: 승인 대기

---

## SCQ 피라미드

**S (Situation):**
- 현재 wing-fm 예측은 RSI + 볼린저밴드 3규칙 기반 단순 알고리즘
- 예측 시간(07:00)이 아침 브리핑(08:00)·구루 서치(08:15)보다 먼저 실행 → 인텔리전스 미반영
- prometheus_engine 인터뷰 질문이 범용 SW 개발 기준, FM 도메인 전문성 없음
- TradingAgents(2026 벤치마크): 다층 전문 에이전트 구조가 단순 규칙 대비 Sharpe ratio 우위

**C (Complication):**
- 시스템트레이더·퀀트 수준 예측력 목표 대비 현재 시스템은 1차원 기술지표만 사용
- 구루 인텔리전스·거시 환경(환율/금리)·감성 시그널이 예측에 전혀 반영되지 않음
- 예측 근거 미기록 → 회고/개선 불가능한 블랙박스 상태
- Plan 인터뷰 질문의 낮은 도메인 전문성 → 맥락 수집 품질 저하 → 계획 품질 저하

**Q (Solution):**
TradingAgents 아키텍처를 wing-fm에 적용: Technical·Sentiment·Macro 3-레이어 신호 합산
→ 근거 완전 기록 → 자기진화 루프로 주간 적중률 추적·개선.
Prometheus 인터뷰는 도메인별 질문 뱅크 + Socratic 심화 에이전트로 전문성 대폭 향상.

---

## 아키텍처 청사진

### Track 1: wing-fm 퀀트 진화

```
[데이터 수집층] (07:00 ~ 08:20)
  ├── pykrx: OHLCV + RSI + BB + MACD + 모멘텀
  ├── yfinance: 원달러, 미국채10년, S&P500
  └── wing_fm_alert.py: 아침 브리핑 JSON 저장

[분석 에이전트층] (09:00 크론잡 — Claude 직접 실행)
  ├── Technical Agent: 기술지표 5개 → 기술 시그널 + 신뢰도
  ├── Sentiment Agent: 아침 브리핑 + 구루 인텔리전스 데이터 읽기 → 감성 시그널
  └── Macro Agent: 환율/금리 엔진 + WebSearch "외국인 코스피 수급" → 거시 시그널

[합산·의사결정층]
  └── Synthesis: 3-시그널 가중합 → 최종 방향 + 신뢰도 + 근거 기록

[기록층]
  ├── predictions/YYYY-MM-DD_predictions.json (결과)
  └── predictions/YYYY-MM-DD_rationale.md (완전 근거)

[자기진화층] (토요일 회고)
  └── 적중률 추적 → 가중치 조정 → 일요일 개선 반영
```

### Track 2: Prometheus 질문 품질 진화

```
[도메인 질문 뱅크]
  ├── prometheus_questions_fm.json    ← 퀀트/FM 특화 10문
  ├── prometheus_questions_dev.json   ← 개발 프로젝트 특화
  └── prometheus_questions_pm.json    ← PM/보고서 특화

[질문 선택 로직]
  └── prometheus_engine.py --generate <주제>
      → 주제 분류(fm/dev/pm/general) → 도메인 뱅크 로드
      → WebSearch로 최신 트렌드 반영 질문 1~2개 추가

[Socratic 심화 (Phase B)]
  └── 답변 분석 → "가정 드러내기" 후속 질문 자동 생성
      예: "퀀트 전략" 답변 → "어떤 데이터로 백테스트할 예정인가요?"
```

---

## WBS — 서브태스크 (총 18개)

### Phase 0: 타이밍 수정 (즉시, ~30분)
- [ ] **0-1.** jobs.json `fm-daily-predict` 크론 07:00 → 09:30 변경
  - 이유: 아침 브리핑(08:00) + 구루 서치(08:15) 완료 후 예측 실행
- [ ] **0-2.** fm_predictions.py에 브리핑 데이터 읽기 연동 준비
  - `.omc/state/pm_briefing_data.json` 존재 시 로드
  - wing_fm_alert 결과 캐시 경로 정의

### Phase 1: 기술지표 강화 (Technical Agent)
- [ ] **1-1.** `fm_predictions.py` 기술지표 확장
  - RSI(14) ✅ 유지
  - MACD(12,26,9) 추가: signal crossover 방향 판단
  - 모멘텀(5일/20일) 추가: 단기 vs 중기 괴리
  - 거래량 변화율 추가: 평균 대비 1.5× 이상 = 이상 신호
- [ ] **1-2.** 5-시그널 가중 합산 규칙 설계 (RSI 30% + MACD 25% + BB 20% + 모멘텀 15% + 거래량 10%)
- [ ] **1-3.** 신뢰도 점수(0~100) 산출: 시그널 일치도 기반

### Phase 2: 감성 + 거시 에이전트 (Sentiment/Macro Agent)
- [ ] **2-1.** `fm_predictions.py --generate`에 브리핑 데이터 읽기 추가
  - `pm_briefing_data.json` + 구루 인텔리전스 결과 JSON 읽기
  - 테이버(모멘텀 시그널) / 오건영(환율 경계 여부) 값 추출
- [ ] **2-2.** Sentiment 시그널 변환 로직
  - 테이버 🟢 강세 = +1 / 🟡 혼조 = 0 / 🔴 약세 = -1
  - 구루 해외 방어적 스탠스 3인 이상 = -0.5 보정
- [ ] **2-3.** Macro 시그널: 원달러 1,380 이상 → 수급 패널티 -0.3
- [ ] **2-4.** WebSearch 연동: 크론잡 프롬프트에서 Claude가 "외국인 코스피 순매수" 검색 후 시그널 추가

### Phase 3: 근거 기록 시스템
- [ ] **3-1.** 각 종목 예측에 `rationale` 블록 추가:
  ```json
  "rationale": {
    "technical": {"rsi": 78.7, "signal": "과매수→하락", "macd": "데드크로스"},
    "sentiment": {"taiver": "강세+1", "guru_overseas": "방어적-0.5"},
    "macro": {"usdkrw": 1465, "signal": "수급주의-0.3"},
    "weights": {"tech": 0.5, "sentiment": 0.3, "macro": 0.2},
    "confidence": 72,
    "decision_rule": "합산점수 -0.8 → 하락/소폭"
  }
  ```
- [ ] **3-2.** `YYYY-MM-DD_rationale.md` 생성: 전 종목 근거 마크다운 요약
- [ ] **3-3.** 텔레그램 브리핑에 핵심 근거 1줄 추가: `[RSI78↑ + 구루방어 → 하락]`

### Phase 4: 자기진화 루프 강화
- [ ] **4-1.** `fm_retrospective.py` 개선: 지표별 적중률 계산
  - RSI 단독 적중률 vs 3-레이어 적중률 비교
- [ ] **4-2.** 가중치 자동 조정 로직: 최근 4주 적중률 기반
  - 적중 지표 가중치 +5%, 미적중 -5% (상한 60%, 하한 10%)
- [ ] **4-3.** `fm_weights.json` 파일로 가중치 영속 저장

### Phase 5: Prometheus FM 질문 뱅크 (Track 2)
- [ ] **5-1.** `prometheus_questions_fm.json` 생성 — 10문항:
  1. 예측 대상 자산의 핵심 드라이버는 무엇인가? (거시/섹터/종목 중)
  2. 어떤 지표가 이 자산에 가장 선행하는가? (환율/금리/수급/기술)
  3. 현재 시장 국면(상승/하락/횡보)을 어떻게 판단하는가?
  4. 포지션 진입 기준(매수 트리거)은 무엇인가?
  5. 손절 기준과 목표 수익률은 어떻게 설정하는가?
  6. 구루 의견이 상충할 때 어떤 우선순위로 판단하는가?
  7. 이 예측의 가장 큰 리스크 시나리오는?
  8. 백테스트 또는 과거 유사 사례가 있는가?
  9. 포트폴리오 전체 리스크(상관관계) 고려 사항은?
  10. 이 시스템의 예측력을 어떻게 측정하고 개선할 것인가?
- [ ] **5-2.** `prometheus_engine.py --generate` 주제 분류 로직 추가
  - 주제에 "fm/투자/주식/예측/포트폴리오" 포함 시 fm 뱅크 사용
- [ ] **5-3.** WebSearch 트렌드 질문 1개 자동 추가 (--research 자동 트리거)

### Phase 6: Socratic 심화 에이전트 (Claude Code 공식 서브에이전트 패턴)
- [ ] **6-1.** 답변 분석 로직: 답변이 "모호/표면적"이면 심화 질문 생성
  - 판단 기준: 구체적 수치/근거 없음 → 심화
  - 심화 예: "RSI 65" 답 → "RSI 65는 어떤 시간축 기준인가요?"
- [ ] **6-2.** `prometheus_engine.py --answer` 심화 트리거 조건 추가
- [ ] **6-3.** FM 전용 심화 질문 템플릿 10개 작성 (가정 드러내기 방식)

---

## 의존성 맵

```
Phase 0 (타이밍) ← 즉시 가능
    ↓
Phase 1 (기술지표) ← Phase 0 완료 후
Phase 2 (감성/거시) ← Phase 0 완료 후 (병렬)
    ↓
Phase 3 (근거 기록) ← Phase 1 + 2 완료 후
    ↓
Phase 4 (자기진화) ← Phase 3 완료 후

Phase 5 (FM 질문뱅크) ← 독립, 즉시 시작 가능
Phase 6 (Socratic) ← Phase 5 완료 후
```

---

## 리스크

| 단계 | 리스크 | 심각도 | 대응 |
|------|--------|--------|------|
| Phase 2-4 WebSearch | 크론잡에서 WebSearch 응답 느릴 수 있음 | MED | 타임아웃 30s, 실패 시 스킵 + 이전값 유지 |
| Phase 1 MACD | pykrx 데이터 60일치 필요 — 상장 초기 종목 오류 | LOW | `len(df) < 30`이면 기술지표 스킵 처리 |
| Phase 4 가중치 자동조정 | 4주 샘플 부족 시 과적합 | MED | 최소 10일 데이터 없으면 기본 가중치 유지 |
| Phase 6 심화 질문 | 텔레그램 메시지 길이 초과 | LOW | 심화 질문 1개만, 선택지 없이 단문 |

---

## 완료 기준 (Acceptance Criteria)

- [ ] fm-daily-predict 크론이 09:30으로 실행됨 확인
- [ ] 예측 JSON에 `rationale.technical`, `rationale.sentiment`, `rationale.macro` 필드 존재
- [ ] `confidence` 점수 0~100 범위로 정상 출력
- [ ] 텔레그램 브리핑에 근거 1줄 포함: `[RSI78↑ + 구루방어 → 하락/소폭 신뢰도 72%]`
- [ ] `prometheus_questions_fm.json` 10문 존재
- [ ] `--generate fm` 실행 시 FM 뱅크 질문 사용됨 확인
- [ ] fm_retrospective.py가 지표별 적중률 출력 확인

---

## 참고 벤치마크

- **TradingAgents** (2026): 7-에이전트 구조, 단순 기술지표 대비 Sharpe ratio 우위
  - Fundamental / Sentiment / News / Technical / Researcher / Trader / Risk Manager
- **Claude Code 공식**: Subagent 체인으로 도메인 격리 + 병렬 분석 권장
- **QuantAgent** (arXiv 2509.09995): 가격 주도 멀티에이전트, 신호 합산 가중치 동적 조정
