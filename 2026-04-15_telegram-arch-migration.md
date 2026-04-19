# 실행 계획: 텔레그램 아키텍처 전면 교체
**작성일:** 2026-04-15  
**목표:** 커스텀 게이트웨이 인프라 → Anthropic 공식 Claude Code Channels 전환  
**성공 기준:** `claude --channels plugin:telegram@claude-plugins-official` 기동 후 텔레그램 메시지가 메인 세션에 도달하고, `mcp__plugin_telegram__reply`로 응답 전송까지 정상 작동

---

## 현재 파일 전수 조사 결과

| 파일 | 목적 | 처리 |
|------|------|------|
| `scripts/tg_gateway.py` | 200ms 폴링 데몬, NAV/NATIVE 처리, SQLite enqueue, _spawn_auto_reply | **Archive** |
| `scripts/tg_queue.py` | SQLite 기반 메시지 큐 (유실 방지) | **Archive** |
| `scripts/atomic_queue.py` | JSONL 원자 쓰기 (gateway↔hook 동기화) | **Archive** |
| `scripts/check_tg_queue.py` | UserPromptSubmit 훅 — JSONL 읽어 세션 주입 | **Archive** |
| `scripts/cleanup_tg_queue.py` | message_queue.jsonl 주기적 정리 | **Archive** |
| `scripts/_check_queue.py` | 큐 상태 디버그 도구 | **Archive** |
| `scripts/gateway_watchdog.py` | gateway 데몬 생존 감시 & 자동 재시작 | **Archive** |
| `scripts/gateway_start.ps1` | gateway 시작 스크립트 | **Archive** |
| `scripts/gateway_stop.ps1` | gateway 종료 스크립트 | **Archive** |
| `.claude/channels/telegram/poller.py` | 공식 플러그인 대신 직접 폴링 | **Archive** |
| `.claude/channels/telegram/message_queue.jsonl` | JSONL 큐 파일 | **Archive** |
| `.claude/hooks/gateway_nav_blocker.ps1` | MCP 중복 응답 차단 (UserPromptSubmit) | **삭제** |
| `scripts/send_tg.py` | MCP 없는 세션용 Bot API 직접 발신 | **유지** (Stop훅 등 보조 도구) |
| `scripts/tg_keyboard.py` | 2-Depth ReplyKeyboard 메뉴 | **유지 + 재연결** |
| `scripts/intent_gate.py` | 자연어 → Magic Keyword 라우터 | **유지 + 재연결** |
| `scripts/dynamic_scanner.py` | 스킬 인벤토리 동적 스캔 | **유지** |
| `.claude/hooks/telegram_notify.ps1` | Stop 훅 — 완료 알림 발송 | **유지** (Bot API 직접 호출, MCP 불필요) |
| `.claude/hooks/check_tg_queue.py` | UserPromptSubmit 훅 | **Archive** (훅 등록도 제거) |

---

## 서브태스크 (총 14개)

### Phase 1 — 안전 보관 (되돌릴 수 있게)
- [ ] **1.** `Archive/legacy_tg_infra/` 디렉토리 생성
- [ ] **2.** 폐기 대상 스크립트 9개 Archive 이동
  - `tg_gateway.py`, `tg_queue.py`, `atomic_queue.py`, `check_tg_queue.py`, `cleanup_tg_queue.py`, `_check_queue.py`, `gateway_watchdog.py`, `gateway_start.ps1`, `gateway_stop.ps1`
- [ ] **3.** `.claude/channels/telegram/` 하위 `poller.py`, `message_queue.jsonl` Archive 이동
- [ ] **4.** `gateway_nav_blocker.ps1` 훅 제거 + settings.json에서 훅 등록 해제
- [ ] **5.** `check_tg_queue.py` UserPromptSubmit 훅 등록 해제 (파일은 Archive)
- [ ] **6.** `jobs.json`에서 `gateway-watchdog` 항목 제거

### Phase 2 — 공식 플러그인 설치
- [ ] **7.** 기존 공식 플러그인 상태 확인 (`/plugin list`)
- [ ] **8.** 플러그인 설치 & 봇 토큰 설정
  ```
  /plugin marketplace add anthropics/claude-plugins-official  (필요시)
  /plugin install telegram@claude-plugins-official
  /telegram:configure <TELEGRAM_BOT_TOKEN>
  ```
- [ ] **9.** 액세스 정책 설정 (`/telegram:access policy allowlist`)

### Phase 3 — 기능 재연결
- [ ] **10.** `general-tg.md` 규칙 정리
  - 제거: gateway IPC 큐 항목, poller 관련 항목, `_spawn_auto_reply` 관련 내용
  - 유지: Magic Keyword Bridge 전체, ReplyKeyboard 네비게이션, IntentGate 라우팅
- [ ] **11.** CLAUDE.md 세션 시작 절차에서 gateway 관련 항목 제거
- [ ] **12.** `jobs.json` 크론 정리 (gateway-watchdog 제거 외 전체 유지)

### Phase 4 — 검증
- [ ] **13.** `claude --channels plugin:telegram@claude-plugins-official` 기동 확인
- [ ] **14.** 기능 검증 체크리스트:
  - [ ] 텔레그램 메시지 → `<channel>` 이벤트 주입 확인
  - [ ] `mcp__plugin_telegram__reply` 응답 전송 확인
  - [ ] `!menu` → `tg_keyboard.py` → ReplyKeyboard 표시 확인
  - [ ] 자연어 → `intent_gate.py` → Magic Keyword 라우팅 확인
  - [ ] Stop 훅 (`telegram_notify.ps1`) 완료 알림 확인

---

## 의존성 맵

```
[1 Archive 디렉토리 생성]
        ↓
[2 스크립트 Archive] → [3 poller Archive] → [4 blocker 훅 제거]
                                           → [5 queue 훅 해제]
                                           → [6 jobs.json 정리]
        ↓ (Phase 1 완료)
[7 플러그인 상태 확인]
        ↓
[8 설치 & 토큰 설정] → [9 액세스 정책]
        ↓ (Phase 2 완료)
[10 rules 정리] → [11 CLAUDE.md 정리] → [12 jobs 정리]
        ↓ (Phase 3 완료)
[13 기동] → [14 검증]
```

**병렬 가능:** 2, 3 / 4, 5, 6 / 10, 11, 12

---

## 리스크 시나리오

| 단계 | 리스크 | 롤백/우회 |
|------|--------|----------|
| 8. 플러그인 설치 | 마켓플레이스 미등록 | `marketplace add anthropics/claude-plugins-official` 후 재시도 |
| 8. 토큰 설정 | 기존 봇 토큰이 공식 플러그인과 충돌 | 동일 봇 토큰 사용 가능 — BotFather 새 봇 불필요 |
| 9. 페어링 | allowlist 설정 전 낯선 사용자 접근 | 즉시 `policy allowlist` 설정으로 차단 |
| 14. 검증 실패 | `<channel>` 이벤트 미도달 | Archive에서 gateway 복원 가능 (삭제 아님) |
| 14. ReplyKeyboard 미작동 | tg_keyboard.py Bot API 직접 호출 방식 — 토큰만 맞으면 작동 | send_tg.py fallback으로 디버그 |

**롤백 전략:** Phase 1에서 삭제 대신 Archive 이동이므로, 14번 검증 실패 시 파일 복원 + 훅 재등록으로 5분 내 원복 가능.

---

## 공식 기동 명령 (전환 후)

```bash
# 기존 (폐기)
python scripts/gateway_start.ps1

# 신규 (공식)
claude --channels plugin:telegram@claude-plugins-official
```

**영구 설정 (매 세션마다 --channels 없이 기동하려면):**  
`settings.json`에 `"channelsEnabled": true` + 플러그인 등록

---

## 전환 후 구조 비교

| 항목 | 현재 | 전환 후 |
|------|------|--------|
| 커스텀 파일 | 10개+ | 2개 (keyboard, intent_gate) |
| 응답 경로 | 2개 (충돌) | 1개 (`mcp__plugin_telegram__reply`) |
| 백그라운드 데몬 | gateway + watchdog | 없음 |
| 오프셋 파일 | 2개 (불일치) | 0개 |
| 훅 충돌 | nav_blocker vs check_tg_queue | 없음 |
| 유지 기능 | — | 메뉴/IntentGate/Magic Keyword 전부 |
