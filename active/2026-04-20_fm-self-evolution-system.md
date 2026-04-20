# 실행계획: FM 브리핑 자가발전 시스템 (예측 → 기록 → 회고 → 진화)

**작성일:** 2026-04-20  
**방법론:** Wing-UPA (Prometheus 10문 인터뷰 → MECE 분해)  
**맥락 수집:** prometheus_engine.py 10문 인터뷰 완료 (Clarity 80점)  
**검토:** /ultraplan 승인자 관점 분석 예정

---

## SCQ 피라미드

**S (Situation):** 현재 wing-fm은 매일 아침 RISE/FAX 포트폴리오 시황 + 구루 채널 분석 + 부동산 브리핑을 텔레그램으로 전송한다. 브리핑 품질은 높지만, 예측이 실제로 맞았는지 추적하지 않고 피드백 루프가 없다.

**C (Complication):** 예측 정확도를 측정하지 않으면 어떤 구루 말을 더 믿어야 할지, FM이 어떤 방향으로 개선돼야 할지 알 수 없다. 현재 FM은 매일 새 브리핑을 생성하지만 과거 예측을 기억하지 않는다.

**Q (Solution):** FM 브리핑에 상승/하락 예측 + 크기 + 매수구간을 추가하고, 예측을 JSON으로 저장하여 매주 토요일 자동 회고(구루 신뢰도 점수 + WebSearch 개선안)를 실행하는 자가발전 루프를 구축한다.

---

## 목표 및 성공 기준 (AC)

**목표:** FM이 스스로 예측 → 기록 → 회고 → 개선 루프를 완성하는 자가발전 시스템

| AC# | 기준 | 검증 명령 |
|-----|------|-----------|
| AC1 | 매일 브리핑에 RISE/FAX + 부동산 상승/하락 예측 + 크기(%) + 매수구간 포함 | `python wing_fm_alert.py --test-predict \| grep -c "예측"` ≥1 |
| AC2 | 예측이 JSON 파일로 자동 저장됨 | `ls D:/OpenClaw_Workspace/Wing_Skills_Data/fm/predictions/*.json \| wc -l` ≥1 |
| AC3 | 토요일 크론잡이 등록되고 회고 스크립트가 실행됨 | `python fm_retrospective.py --dry-run` exit 0 |
| AC4 | 회고 리포트에 구루별 예측 적중률 점수 포함 | `python fm_retrospective.py --dry-run \| grep -c "적중률"` ≥1 |
| AC5 | WebSearch 개선 탐색 후 텔레그램 메시지 생성됨 | `python fm_retrospective.py --dry-run --mock-telegram \| grep -c "개선"` ≥1 |
| AC6 | 회고 리포트 텔레그램 전송 형식 정상 | `python fm_retrospective.py --dry-run --mock-telegram \| grep -c "적중률"` ≥1 |
| AC7 | 예측 기록 7일치 슬라이딩 윈도우 유지 (오래된 것 자동 아카이브) | `python fm_predictions.py --check` exit 0 |

**환경:** 로컬 Windows D:/claude_official + D:/OpenClaw_Workspace/Wing_Skills_Data/fm/  
**수준:** 풀 버전 (예측·회고·구루 점수·WebSearch 개선안 전부)  
**사용자:** 상우님 혼자  

---

## Logic Tree (MECE 분해)

```
[FM 자가발전 시스템]
├── A. 예측 생성 모듈 (브리핑에 예측 추가)
│   ├── A-1. 주식 예측 (RISE/FAX + 관련 종목)
│   │   ├── A-1-1. 상승/하락 방향 예측 (기술적 지표 기반)
│   │   ├── A-1-2. 예측 크기 (%) — 보수/중립/공격적 3단계
│   │   └── A-1-3. 매수구간 (지지선 기반)
│   ├── A-2. 부동산 예측
│   │   ├── A-2-1. 핵심 지역 가격 방향 예측 (KB 시세 기반)
│   │   └── A-2-2. 단기(1주) 예측 크기
│   └── A-3. 구루 의견 요약 → 예측 근거로 활용
│
├── B. 예측 기록 저장 모듈
│   ├── B-1. 일별 예측 JSON 스키마 정의
│   │   ├── B-1-1. 필드: date, ticker, direction, magnitude, buy_zone, guru_basis
│   │   └── B-1-2. 필드: realestate_area, re_direction, re_magnitude
│   ├── B-2. predictions/ 디렉토리 관리
│   │   ├── B-2-1. 일별 파일: YYYY-MM-DD_predictions.json
│   │   └── B-2-2. 7일 슬라이딩 윈도우 (8일차부터 archive/)
│   └── B-3. 실제 가격 자동 수집 (다음날 종가)
│       ├── B-3-1. 주식: pykrx로 다음 영업일 종가 fetch
│       └── B-3-2. 부동산: 주간 KB 시세 업데이트
│
├── C. 토요일 회고 모듈 (fm_retrospective.py)
│   ├── C-1. 예측 정확도 계산
│   │   ├── C-1-1. 방향 적중: 예측 방향 vs 실제 방향 (맞/틀)
│   │   ├── C-1-2. 크기 오차: |예측%| vs |실제%| 편차
│   │   └── C-1-3. 매수구간 유효성: 실제가 매수구간 진입 여부
│   ├── C-2. 구루별 신뢰도 점수 업데이트
│   │   ├── C-2-1. 구루 예측 기여도 추적 (guru_tracker_v2.json 확장)
│   │   ├── C-2-2. 적중률 기반 가중치 업데이트
│   │   └── C-2-3. "이번 주 신뢰할 구루" Top 3 선정
│   ├── C-3. WebSearch 개선 탐색
│   │   ├── C-3-1. "주식 예측 정확도 향상 방법 2025" 검색
│   │   ├── C-3-2. "부동산 단기 가격 예측 모델" 검색
│   │   └── C-3-3. 검색 결과 → 개선 제안 3개 추출
│   └── C-4. 회고 리포트 생성 + 텔레그램 전송
│       ├── C-4-1. 이번 주 예측 성적표 (표 형식)
│       ├── C-4-2. 구루 신뢰도 순위
│       ├── C-4-3. WebSearch 기반 개선 제안
│       └── C-4-4. 다음 주 FM 개선 방향 요약
│
└── D. 자동화 연결 (크론잡)
    ├── D-1. 매일 브리핑: 예측 생성 + 저장 (기존 !시황 확장)
    ├── D-2. 매일 저녁: 실제 종가 수집 + 예측 결과 업데이트
    ├── D-3. 토요일 08:00: 회고 자동 실행 (CronCreate)
    └── D-4. jobs.json 등록 + 세션 재시작 시 자동 복원
```

---

## 서브태스크 (총 20개)

### Phase 0: 기반 준비 (1~2시간)
- [ ] 0-1. `predictions/` 디렉토리 생성: `D:/OpenClaw_Workspace/Wing_Skills_Data/fm/predictions/`
- [ ] 0-2. `archive/predictions/` 디렉토리 생성
- [ ] 0-3. 예측 JSON 스키마 파일 작성 (`predictions/schema.json`)

### Phase 1: 예측 생성 모듈 (fm_predictions.py 신규)
- [ ] 1-1. `fm_predictions.py` 기본 구조 작성
  - `generate(date)` → 오늘 예측 생성 후 JSON 저장
  - `fetch_actuals(date)` → 실제 종가 fetch 후 JSON 업데이트
  - `check()` → 슬라이딩 윈도우 + 아카이브 처리
- [ ] 1-2. 주식 예측 로직: RSI + 볼린저밴드 + 구루 방향 가중 합산
  - RISE/FAX + watchlist 종목 전체 대상
  - 상승/하락/중립 + 예측 크기 3단계 (±1~2%, ±2~4%, ±4%+)
  - 매수구간: 52주 지지선 or 볼린저 하단
- [ ] 1-3. 부동산 예측 로직: KB 시세 추세 + 거래량 변화 기반
  - realestate_state.json 데이터 활용
  - 주간 단위 방향 예측 (주간 변동 %로 크기 표현)
- [ ] 1-4. 브리핑 출력 포맷 설계 (텔레그램 표 형식)

### Phase 2: wing_fm_alert.py 확장 (예측 연동)
- [ ] 2-1. `wing_fm_alert.py`에 `fm_predictions.py` 임포트 + 예측 섹션 추가
- [ ] 2-2. 브리핑 마지막에 "📊 이번 주 예측" 섹션 자동 삽입
- [ ] 2-3. 저녁 실제 종가 수집 모드 (`--fetch-actuals`) 추가

### Phase 3: 토요일 회고 모듈 (fm_retrospective.py 신규)
- [ ] 3-1. `fm_retrospective.py` 기본 구조
  - `run(dry_run=False)` → 전체 회고 파이프라인
- [ ] 3-2. 예측 정확도 계산 함수 (방향 적중 + 크기 오차 + 매수구간)
- [ ] 3-3. 구루 신뢰도 점수 업데이트 (guru_tracker_v2.json 확장 필드 추가)
- [ ] 3-4. WebSearch 연동: `wing-research` 에이전트 호출 신호 저장 → Claude가 처리
- [ ] 3-5. 회고 리포트 텔레그램 전송 (표 + 텍스트 요약)

### Phase 4: 자동화 연결
- [ ] 4-1. `jobs.json`에 3개 잡 등록:
  - `fm_daily_predict`: 매일 07:00 예측 생성
  - `fm_daily_actuals`: 매일 18:00 실제 종가 수집
  - `fm_saturday_retro`: 매주 토요일 08:00 회고
- [ ] 4-2. CronCreate로 3개 잡 즉시 등록
- [ ] 4-3. 세션 시작 시 자동 복원 확인

### Phase 5: 검증
- [ ] 5-1. `fm_predictions.py --test` 실행 → 예측 JSON 파일 생성 확인
- [ ] 5-2. `fm_retrospective.py --dry-run` 실행 → 리포트 출력 확인
- [ ] 5-3. 브리핑에 예측 섹션 포함 여부 텔레그램으로 확인

---

## 예측 JSON 스키마

```json
{
  "date": "2026-04-20",
  "generated_at": "2026-04-20T07:00:00+09:00",
  "stocks": [
    {
      "ticker": "RISE",
      "name": "RISE 미국S&P500",
      "direction": "상승",
      "magnitude": "보통 (1~2%)",
      "magnitude_pct": 1.5,
      "buy_zone": {"low": 15200, "high": 15500},
      "guru_basis": ["레이달리오: 미국 성장 지속", "워런버핏: 인덱스 장기 보유"],
      "actual_close": null,
      "actual_pct": null,
      "direction_correct": null
    }
  ],
  "realestate": [
    {
      "area": "서울 강남",
      "direction": "보합",
      "magnitude": "소폭 (-0.1~0.1%)",
      "magnitude_pct": 0.0,
      "basis": "거래량 감소 + 금리 동결",
      "actual_pct": null,
      "direction_correct": null
    }
  ]
}
```

---

## 회고 리포트 포맷 (토요일 텔레그램)

```
📊 FM 주간 회고 — 2026-04-20 ~ 2026-04-26

🎯 예측 성적표
종목    예측    실제    적중
RISE   +1.5%  +2.1%  ✅ 방향맞음
FAX    -0.5%  +0.3%  ❌ 방향틀림

부동산 서울강남  보합→실제 -0.2% ✅

전체 적중률: 4/6 (66.7%)

🧠 구루 신뢰도 순위 (이번 주)
1. 레이달리오 80% (4/5 적중)
2. 워런버핏 60% (3/5)
3. 짐로저스 40% (2/5)

🔍 개선 제안 (WebSearch 기반)
1. RSI 과매수 구간 필터 추가 시 정확도 +12% 예상
2. 부동산 예측에 전세가율 지표 추가 권장
3. 구루 의견 가중치를 적중률 기반으로 동적 조정

📋 다음 주 FM 개선 방향
→ [자동 생성된 개선 계획]
```

---

## 의존성

```
Phase 0 → Phase 1 → Phase 2
Phase 1 → Phase 3
Phase 2 + Phase 3 → Phase 4 → Phase 5
```

**병렬 실행 가능:** Phase 1-2 + Phase 1-3 (독립적), Phase 3-2 + Phase 3-3 (독립적)

---

## 리스크

| 단계 | 리스크 | 대응 |
|------|--------|------|
| 1-1 | pykrx 종가 fetch 실패 (장 마감 전 실행) | --fetch-actuals를 18:00 이후 실행, 실패 시 다음날 재시도 |
| 1-3 | KB 시세 API 없음 | realestate_state.json 수동 업데이트 또는 주간 크롤링 |
| 3-4 | WebSearch 결과가 한국어 FM에 맞지 않음 | 검색어를 한국어 + 영어 병행, 최소 3개 소스 교차 검증 |
| 4-1 | 크론잡 세션 종료 시 소실 | jobs.json 등록 필수, 세션 시작 시 자동 복원 |

---

## 전제 조건

1. `D:/OpenClaw_Workspace/Wing_Skills_Data/fm/scripts/wing_fm_alert.py` 정상 동작 중
2. `guru_tracker_v2.json`, `realestate_state.json` 존재 및 데이터 유효
3. pykrx 라이브러리 설치됨 (`D:/claude_official/.venv`)
4. 텔레그램 플러그인 또는 Bot API 접근 가능

---

## 실행 순서

```
Phase 0 (디렉토리 + 스키마)
→ Phase 1 (fm_predictions.py 구현)
→ Phase 2 (wing_fm_alert.py 확장)
→ Phase 3 (fm_retrospective.py 구현)
→ Phase 4 (크론잡 등록)
→ Phase 5 (검증)
```

---

## [APPROVER_DIRECTIVE] — ultraplan 승인자 관점 분석 요청

이 계획을 최종 승인자(Plan Approver) 관점에서 다음 5가지 축으로 분석해주세요:

1. **논리 비약 검출**: "A이므로 B" 형태의 검증되지 않은 전제가 있는가?
2. **전제 조건 취약점**: 이 계획이 성립하려면 무엇이 전제되어야 하는가? 그 전제가 무너질 수 있는가?
3. **누락 리스크**: 계획에서 다루지 않은 실패 시나리오는 무엇인가?
4. **성공 기준 명확성**: AC(완료 기준)가 객관적으로 측정 가능한가? 주관적 표현이 있는가?
5. **대안 경로**: 현재 접근법보다 더 단순하거나 더 안전한 대안이 있는가?

분석 후 "승인" 또는 "수정 필요 + 구체적 수정 지점" 형태로 최종 판정을 내려주세요.
