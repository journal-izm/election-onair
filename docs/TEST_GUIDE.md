# P1 개표방송 시스템 — 테스트 실행 가이드 (1차 확인)

설계 문서 8종의 '테스트케이스 — 1차 확인' 61개를 **실제로 돌려 보는 순서와 방법**입니다.
번호(TC-…)는 설계 문서와 같습니다. 각 항목의 ☐에 통과·실패를 표시하고, 실패하면 화면 문구나 터미널 오류를 그대로 메모합니다.

- 대상 코드: [nec-server](https://github.com/journal-izm/nec-server) (선관위) · [station-onair](https://github.com/journal-izm/station-onair) (방송국 M·K)
- 명령은 **Windows PowerShell** 기준입니다. macOS·Linux는 `.venv\Scripts\activate` → `source .venv/bin/activate`, `copy` → `cp`로 바꿉니다.
- 자동 = pytest가 확인 / 수동 = 사람이 화면·소리·명령 결과로 확인

---

## 0. 한눈에 보기

| 순서 | 할 일 | 걸리는 시간 |
|---|---|---|
| 1 | [준비](#1-준비): 설치 · .env · 서버 3개 켜기 | 20분 |
| 2 | [자동 테스트](#2-자동-테스트-pytest): `python -m pytest -q` 두 번 | 2분 |
| 3 | [1단계 선관위 서버](#4-1단계--선관위-서버) 수동 테스트 | 15분 |
| 4 | [2단계 방송국 수신·웹](#5-2단계--방송국-수신웹) | 25분 |
| 5 | [3단계 기사 흐름](#6-3단계--기사-흐름) | 25분 |
| 6 | [4단계 송출](#7-4단계--송출) | 30분 |
| 7 | [5단계 마무리](#8-5단계--마무리아바타시연) | 20분 |
| 8 | [결과 기록표](#10-결과-기록표) 채우기 | — |

**단계 규칙:** 한 단계의 테스트를 모두 통과해야 다음 단계로 넘어갑니다. 실패하면 [문제 해결](#9-문제가-생기면)을 먼저 봅니다.

---

## 1. 준비

### 1-1. 받기와 설치 (두 저장소 각각)

```powershell
git clone https://github.com/journal-izm/nec-server.git
git clone https://github.com/journal-izm/station-onair.git

cd nec-server
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
cd ..

cd station-onair
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
cd ..
```

> chromadb 설치가 오래 걸리거나 실패하면 `requirements.txt`에서 그 줄을 지우고 다시 설치해도 됩니다 (없으면 낱말 검색으로 동작).

### 1-2. 설정 파일 (.env)

**nec-server**
```powershell
cd nec-server
copy .env.example .env
notepad .env        # ADMIN_KEY=원하는-관리키  (예: nec-admin-2026)
python seed.py
```
`seed.py`가 **방송국 키 2개**를 한 번만 보여 줍니다 (`★ M 방송국 API 키 …`). 메모장에 복사해 둡니다.
(다시 보고 싶으면 `.env`에 `STATION_KEY_M=`, `STATION_KEY_K=` 값을 직접 정해 적고 `python seed.py`를 다시 실행)

**station-onair**
```powershell
cd station-onair
copy .env.example .env.m
copy .env.example .env.k
notepad .env.m
notepad .env.k
```

| 칸 | .env.m | .env.k |
|---|---|---|
| STATION_CODE | M | K |
| STATION_NAME | M 방송국 | K 방송국 |
| OTHER_STATION_URL | http://localhost:8102 | http://localhost:8101 |
| SECRET_KEY | 아무 긴 문자열 | 다른 긴 문자열 |
| DEFAULT_PASSWORD | 계정 5개의 처음 비밀번호 | 같은 값이어도 됨 |
| NEC_API_KEY | seed가 보여 준 M 키 | seed가 보여 준 K 키 |
| OPENAI_API_KEY | 3단계 전까지는 **비워 둠** | 비워 둠 |

> OPENAI_API_KEY를 비워 두면 기사는 **틀 기사**(수치만 채움), 낭독은 **브라우저 음성**으로 동작합니다. 1~2단계와 3단계 대부분은 키 없이 테스트합니다.

### 1-3. 서버 3개 켜기 (터미널 3개)

```powershell
# 터미널 ① 선관위
cd nec-server; .venv\Scripts\activate
uvicorn app.main:app --port 8000

# 터미널 ② M 방송국
cd station-onair; .venv\Scripts\activate
uvicorn app.main:app --port 8101 --env-file .env.m --timeout-graceful-shutdown 2

# 터미널 ③ K 방송국
cd station-onair; .venv\Scripts\activate
uvicorn app.main:app --port 8102 --env-file .env.k --timeout-graceful-shutdown 2
```

방송국 서버는 처음 켤 때 초기 데이터(계정 5 · 후보 자료 12 · 큐시트 10)를 자동으로 넣습니다.

### 1-4. 자주 쓰는 주소와 계정

| 이름 | 주소 |
|---|---|
| 선관위 관리 화면 (SCR-02-01) | http://localhost:8000/admin |
| 선관위 API 문서 | http://localhost:8000/docs |
| M 실시간 개표 웹 (SCR-06-01) | http://localhost:8101/ |
| M 로그인 (SCR-01-01) | http://localhost:8101/login |
| M 기자 (SCR-04) | http://localhost:8101/reporter |
| M 데스크 (SCR-05) | http://localhost:8101/desk |
| M 수신 모니터 (SCR-03-01) | http://localhost:8101/monitor |
| M 큐시트·송출 제어 (SCR-07-02) | http://localhost:8101/onair/control |
| M 송출 화면 (SCR-07-01) | http://localhost:8101/onair/screen |
| M API 문서 | http://localhost:8101/docs |

| 계정 | 역할 | 비밀번호 |
|---|---|---|
| reporter1 · reporter2 · reporter3 | 기자 | .env의 DEFAULT_PASSWORD |
| desk1 · desk2 | 데스크 | 같음 |

> **브라우저 팁:** 한 브라우저에서는 한 계정만 로그인됩니다. 기자와 데스크를 동시에 보려면 일반 창 + 시크릿 창, 또는 크롬 + 엣지처럼 나눠 씁니다.

### 1-5. 확인 도구

방송국 DB 값을 읽기 쉽게 보여 주는 도구입니다. station-onair 폴더에서 실행합니다.

```powershell
python tools/check.py --env .env.m counts      # 표마다 행 수
python tools/check.py --env .env.m events      # 감지한 이벤트
python tools/check.py --env .env.m article 3   # 기사 3번 자세히
python tools/check.py --env .env.m latency     # 승인 → 음성 준비 시간
python tools/check.py --env .env.m queue       # 방송 대기열
python tools/check.py --env .env.m llm         # gpt-6-luna 연결
python tools/check.py --env .env.m tts "문장"   # TTS 연결
```

### 1-6. 처음부터 다시 하기 (리허설 전 초기화)

1. 선관위 관리 화면에서 **⟲ 초기화**
2. 방송국 서버 두 개를 끄고(Ctrl+C) `python seed.py --reset`을 M·K 각각 실행
   ```powershell
   python seed.py --env .env.m --reset
   python seed.py --env .env.k --reset
   ```
3. 방송국 서버 다시 켜기 → 큐시트 화면에서 **⟲ 초기화**

---

## 2. 자동 테스트 (pytest)

```powershell
cd nec-server;     .venv\Scripts\activate; python -m pytest -q
cd ..\station-onair; .venv\Scripts\activate; python -m pytest -q
```

| 저장소 | 기대 결과 |
|---|---|
| nec-server | `13 passed` |
| station-onair | `25 passed` |

`skipped`가 보이면 fastapi·httpx가 설치되지 않은 것입니다 (`pip install -r requirements.txt` 다시).

**자동 테스트가 맡는 TC**

| TC | 테스트 함수 (파일) |
|---|---|
| TC-SC-01 | `test_tc_sc_01_broadcast_order_with_elected` (station `tests/test_logic.py`) |
| TC-SC-02 | `test_tc_sc_02_same_scenario_every_time`, `test_tc_sc_02_reset_replays_same` (nec) |
| TC-SC-05 | `test_tc_sc_05_skip_interview_when_no_winner` (station) |
| TC-RQ-01 | `test_tc_rq_01_fixed_accounts` (station `tests/test_api.py`) |
| TC-RQ-02 | `test_tc_rq_02_role_access` (station) |
| TC-RQ-03 | `test_tc_rq_03_and_api_06_approve_publish` (station) |
| TC-RQ-05 | `test_tc_rq_05_keys_are_hashed` (nec), `test_tc_rq_05_password_hashed` (station) |
| TC-FN-01 | `test_tc_fn_01_since_seq` (nec) |
| TC-FN-02 · 03 | `test_tc_fn_02_…`, `test_tc_fn_03_win_levels`, `test_full_scenario_events` (station) |
| TC-FN-04 · 05 | `test_tc_fn_04_…`, `test_tc_fn_05_…` (station) |
| TC-FN-06 | `test_tc_rq_03_and_api_06_approve_publish` 안의 두 번째 승인 409 (station) |
| TC-FN-07 · 08 · 09 | `test_tc_fn_07_…`, `test_tc_fn_08_…`, `test_tc_fn_09_…` (station) |
| TC-DB-01 · 02 | `test_tc_db_01_seed`, `test_tc_db_02_snapshots` (nec) |
| TC-DB-03 · 04 | `test_tc_db_03_and_04_results_and_events`, `test_tc_db_03_summary_has_all_and_regions` (station) |
| TC-DB-05 | `test_tc_db_05_reject_then_approve` (station) |
| TC-API-01 ~ 04 | `test_tc_api_01_…` ~ `test_tc_api_04_…` (nec) |
| TC-API-05 · 06 · 07 · 09 | `test_tc_api_05_…`, `test_tc_rq_03_and_api_06_…`, `test_tc_api_07_…`, `test_tc_api_09_…` (station) |
| TC-ENV-03 | `.gitignore` — [8장 TC-ENV-03](#tc-env-03-git에-올리지-않을-것)에서 한 번 더 눈으로 확인 |

> 자동 테스트가 통과해도 화면·소리는 사람이 봐야 합니다. 아래 수동 테스트를 이어서 합니다.

---

## 3. 기준표: 시나리오의 개표 흐름

선관위 시나리오(`hanbit_2026.csv`)는 매번 같은 결과를 냅니다. 아래는 **시뮬레이터를 시작한 뒤** 이벤트가 생기는 시점입니다.
방송국은 5초마다 받으므로 화면에는 **최대 5초 늦게** 나타납니다. 이벤트는 모두 **22건**입니다.

| 1배속 | 5배속 | 이벤트 | 지역 | 내용 |
|---|---|---|---|---|
| 1:00 | 0:12 | 개표율 | 전체 | 10% 돌파 · 1위 류해온 |
| 2:46 | 0:33 | 개표율 | 전체 | 30% 돌파 |
| 3:42 | 0:44 | 1위 변동 | 성운구 | 류해온 → 윤별하 |
| 4:25~4:53 | 0:53~0:59 | 개표율 | 궤도구·심해구·별빛구·전체·성운구·사막구 | 각 50% 돌파 (6건) |
| 5:46 | 1:09 | 1위 변동 | 사막구 | 류해온 → 윤별하 |
| 6:38 | 1:20 | 개표율 | 전체 | 70% 돌파 |
| **6:52** | **1:22** | **1위 변동** | **전체** | **류해온 → 윤별하 (역전)** |
| 8:41 | 1:44 | 개표율 | 전체 | 90% 돌파 |
| **9:04** | **1:49** | **당선 유력** | 전체 | 윤별하 (개표율 93.9%) |
| 9:05 | 1:49 | 개표율 | 궤도구 | 100% |
| 9:20 | 1:52 | 개표율 | 심해구 | 100% |
| **9:24** | **1:53** | **당선 확실** | 전체 | 윤별하 (96.7%) |
| **9:30** | **1:54** | **당선 확정** | 전체 | 윤별하 (97.8%) |
| 9:45~10:00 | 1:57~2:00 | 개표율 | 별빛구·성운구·사막구·전체 | 각 100% (4건) |

최종 결과: **윤별하 44.45% (100,189표) · 류해온 41.53% (93,614표) · 오로라 14.02% (31,609표)**

> **본방송 권장:** LIVE 구간(방송 4:30)이 시작될 때 시뮬레이터를 **1배속으로 시작**하면, 당선 확정이 방송 약 14:00에 나오고 기사 송고·승인까지 마쳐 15:30 전에 인터뷰로 넘어갑니다.

---

## 4. 1단계 — 선관위 서버

### TC-UI-02 개표 시뮬레이터 화면
- **절차**
  1. http://localhost:8000/admin 을 열고 관리 키 칸에 `.env`의 ADMIN_KEY를 넣고 Enter
  2. ▶ 시작 → 10초 기다림 → ⏸ 일시정지 → 5배속 → ▶ 시작 → ⟲ 초기화(확인 창에서 확인)
- **기대 결과**
  - 키를 넣기 전에는 "관리 키를 입력하면 상태가 보입니다"
  - 시작하면 상태 RUNNING(초록), seq·전체 개표율·경과 시간이 1초마다 늘어남
  - 일시정지하면 PAUSED, 숫자가 멈춤 · 5배속을 누르면 버튼에 테두리 표시, 경과 시간이 5배로 빨라짐
  - 초기화하면 READY · 개표율 0.0% · seq 0 · 경과 00:00
- ☐ 통과 ☐ 실패 메모:

### TC-API-01 선관위 API 키 인증
- **절차** (새 PowerShell 창)
  ```powershell
  curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8000/nec/v1/elections/HANBIT-2026
  curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8000/nec/v1/elections/HANBIT-2026 -H "X-API-KEY: wrong"
  curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8000/nec/v1/elections/HANBIT-2026 -H "X-API-KEY: 여기에-M키"
  ```
- **기대 결과:** `401` → `401` → `200`
- ☐ 통과 ☐ 실패 메모:

### TC-API-02 선거 기본정보 형식 (두 저장소의 약속)
- **절차:** `curl.exe -s http://localhost:8000/nec/v1/elections/HANBIT-2026 -H "X-API-KEY: 여기에-M키"`
- **기대 결과**
  - `election`, `regions`, `candidates` 세 덩어리
  - regions 5개: STAR 별빛구 82000 · DEEP 심해구 76000 · DUNE 사막구 64000 · NEBULA 성운구 71000 · ORBIT 궤도구 58000
  - candidates 3명: 1 윤별하 은하공화연합 #FF2BD6 · 2 류해온 심해행성연맹 #00E5FF · 3 오로라 무소속 #A6FF00
  - `election.total_electors` = 351000
- ☐ 통과 ☐ 실패 메모:

### TC-API-03 개표 변경분 형식 (두 저장소의 약속)
- **절차**
  1. 관리 화면에서 5배속 ▶ 시작, 20초 뒤 ⏸ 일시정지
  2. `curl.exe -s "http://localhost:8000/nec/v1/elections/HANBIT-2026/results?since_seq=0" -H "X-API-KEY: 여기에-M키"`
  3. 응답의 `last_seq` 값으로 다시: `...results?since_seq=마지막값`
- **기대 결과**
  - 첫 응답: `last_seq`, `status`(COUNTING), `has_more`, `regions` 목록. 각 줄에 `seq · code · counted_rate · counted_votes · invalid_votes · votes{"1","2","3"}`
  - 개표율은 0~100 사이, 득표+무효 = 개표수
  - 두 번째 응답: `regions`가 빈 목록 `[]`
- ☐ 통과 ☐ 실패 메모:

### TC-FN-01 변경분만 내려주기
- **절차:** 위에서 이어서 `...results?since_seq=10`
- **기대 결과:** 목록의 seq가 모두 11 이상
- ☐ 통과 ☐ 실패 메모:

### TC-API-04 시뮬레이터 제어 API
- **절차**
  1. http://localhost:8000/docs → **POST /nec/v1/sim/control** → Try it out → Body `{"action": "start"}` → Execute (키 없이)
  2. 같은 화면의 `x-admin-key` 칸에 관리 키를 넣고 다시 Execute
  3. `{"action": "speed", "speed": 3}` 실행
  4. `{"action": "reset"}` 실행
- **기대 결과:** ① 403 "관리 키가 맞지 않습니다" ② 200, `status: RUNNING` ③ 422 (배속은 1·2·5만) ④ `status: READY`, `current_seq: 0`
- ☐ 통과 ☐ 실패 메모:

### TC-DB-01 처음 넣는 데이터 (선관위)
- **절차:** TC-API-02 응답으로 확인 (구 5 · 후보 3) + 관리 화면 ④ 방송국 호출 표에 M·K 두 줄
- **기대 결과:** 구 5개 · 후보 3명 · 방송국 2개
- ☐ 통과 ☐ 실패 메모:

### TC-DB-02 개표 스냅숏 쌓기
- **절차:** 관리 화면 5배속 ▶ 시작 → 약 2분 뒤 상태가 **DONE**이 될 때까지 기다림
- **기대 결과:** seq 약 190, 전체 개표율 100.0%, 상태 DONE. (자동 테스트 `test_tc_db_02_snapshots`가 득표 감소·중복 seq 없음을 확인)
- ☐ 통과 ☐ 실패 메모:

### TC-RQ-05 비밀 정보 보호 (선관위)
- **절차:** 터미널 ① 로그를 훑어보고, 다음을 실행
  ```powershell
  python -c "import sqlite3;print(sqlite3.connect('nec.db').execute('select station_code, api_key_hash from station_key').fetchall())"
  ```
- **기대 결과:** 키 원문이 아니라 64자리 해시만 보임. 로그에 X-API-KEY·ADMIN_KEY 값이 찍히지 않음
- ☐ 통과 ☐ 실패 메모:

### TC-SC-02 같은 결과 다시 재현 (선관위 부분)
- **절차:** 초기화 → 5배속 끝까지 → 기준표의 최종 결과와 비교 → 한 번 더 반복
- **기대 결과:** 두 번 모두 마지막 seq 같음, 최종 득표 같음 (윤별하 100,189 · 류해온 93,614 · 오로라 31,609). 방송국까지 합친 확인은 [5단계 TC-SC-02 후반](#tc-sc-02-후반-방송국-이벤트까지-같은지)
- ☐ 통과 ☐ 실패 메모:

### TC-ENV-01 새 PC 설치 · TC-ENV-02 .env 준비
- **절차**
  1. 1-1 설치가 오류 없이 끝났는지 확인 (TC-ENV-01)
  2. nec-server `.env`에서 `ADMIN_KEY=` 값을 지우고 서버 다시 켜기 → 메시지 확인 → 값 되돌리기
  3. station-onair `.env.m`에서 `NEC_API_KEY=` 값을 지우고 M 서버 다시 켜기 → 메시지 확인 → 값 되돌리기
- **기대 결과:** `[설정 오류] .env에 다음 값이 없습니다: ADMIN_KEY` / `…: NEC_API_KEY` 를 보여 주고 서버가 멈춤
- ☐ 통과 ☐ 실패 메모:

---

## 5. 2단계 — 방송국 수신·웹

> 시작 전: 선관위 ⟲ 초기화. M·K 서버가 켜져 있어야 합니다.

### TC-SYS-01 서버 3대 함께 실행
- **절차:** http://localhost:8000/docs · http://localhost:8101/docs · http://localhost:8102/docs 를 차례로 엽니다. station-onair 폴더를 봅니다.
- **기대 결과:** 세 문서 화면이 모두 열림. 제목이 "nec-server", "station-onair · M 방송국", "… K 방송국". 폴더에 `station_m.db`, `station_k.db`가 따로 생김
- ☐ 통과 ☐ 실패 메모:

### TC-RQ-01 고정 계정 5개 · TC-UI-01 로그인 화면
- **절차:** http://localhost:8101/login 에서 ① desk1 + 틀린 비밀번호 ② reporter1 ③ (로그아웃) desk1 ④ nobody
- **기대 결과**
  - ① "아이디 또는 비밀번호가 맞지 않습니다" ② 기자 화면(/reporter)으로 이동 ③ 데스크 화면(/desk)으로 이동 ④ 같은 오류
  - 회원가입 버튼·주소가 없음. reporter1~3, desk1~2 다섯 계정 모두 로그인됨
- ☐ 통과 ☐ 실패 메모:

### TC-DB-01 처음 넣는 데이터 (방송국)
- **절차:** `python tools/check.py --env .env.m counts`
- **기대 결과:** 계정 5 · 후보 3 · 지역 6(전체 포함) · 후보 자료 12 · 큐시트 10
- ☐ 통과 ☐ 실패 메모:

### TC-UI-03 수신 모니터 · TC-SYS-03 선관위 장애 뒤 따라잡기
- **절차**
  1. desk1로 로그인 → http://localhost:8101/monitor
  2. 선관위 관리 화면 1배속 ▶ 시작
  3. 30초 뒤 터미널 ①(선관위)을 Ctrl+C로 끔 → 30초 기다림 → 다시 켬
  4. 1분 더 기다림
- **기대 결과**
  - 켜져 있을 때: "● 정상"(초록), seq가 늘어남, 수신 기록에 OK
  - 끈 동안: "● 실패"(빨강), 실패 횟수가 늘고 "선관위 서버 연결 실패: ConnectError" 같은 문구
  - 다시 켜면 자동으로 "● 정상", 수신 기록에 한 번에 여러 줄을 받은 OK가 보이고 seq가 선관위 관리 화면의 seq와 같아짐
  - ※ 선관위 서버를 껐다 켜도 시뮬레이터는 멈춘 자리에서 이어서 진행됩니다 (seq 유지)
- ☐ 통과 ☐ 실패 메모:

### TC-DB-03 최신 결과 한 벌
- **절차:** `python tools/check.py --env .env.m counts`
- **기대 결과:** 최신 결과 result_latest = **18** (구 5 × 3명 + 전체 3명)
- ☐ 통과 ☐ 실패 메모:

### TC-DB-04 이벤트 중복 막기
- **절차:** 수신 모니터에서 **지금 바로 받기**를 여러 번 누름 → `python tools/check.py --env .env.m events`
- **기대 결과:** 마지막 줄 "중복 열쇠 없음". 같은 구간 돌파 이벤트가 두 번 나오지 않음
- ☐ 통과 ☐ 실패 메모:

### TC-FN-02 개표율·1위 변동 감지 · TC-FN-03 당선 판정 (화면으로)
- **절차:** 선관위를 5배속으로 끝까지 진행 → 수신 모니터의 '감지한 이벤트' 표와 `check.py events`를 [3장 기준표](#3-기준표-시나리오의-개표-흐름)와 비교
- **기대 결과:** 22건, 순서와 지역·후보가 기준표와 같음. 당선 유력 → 확실 → 확정이 각 1번, 모두 윤별하
- ☐ 통과 ☐ 실패 메모:

### TC-UI-06 실시간 개표 웹 · TC-RQ-06 누구나 보는 웹
- **절차**
  1. 선관위 초기화 → 1배속 시작
  2. 시크릿 창에서 http://localhost:8101/ (로그인 없이)
  3. 브라우저 개발자 도구(F12) → 휴대폰 모양 아이콘 → 폭 390px로 봄
- **기대 결과**
  - 새로고침 없이 전체 개표율·막대·후보 득표율이 바뀜. 위에 "HH:MM:SS 갱신"
  - 후보 동그라미·득표율이 네온색(마젠타·시안·라임), 어두운 배경에서 잘 읽힘
  - 3분쯤 지나 1위가 앞선 후보 위에 오도록 순서가 바뀜
  - 390px 폭에서 세 칸이 한 줄씩 쌓이고 글자·숫자가 잘리지 않음
- ☐ 통과 ☐ 실패 메모:

### TC-UI-07 지역별 상세
- **절차:** 개표 웹 지도에서 별빛구 → 뒤로 → 심해구 → 사막구 → 성운구 → 궤도구
- **기대 결과:** 다섯 곳 모두 /region/코드 상세 화면이 열리고, 지도에서 칠해진 색 = 상세 화면 1위 후보 막대 색
- ☐ 통과 ☐ 실패 메모:

### TC-API-07 공개 결과 API
- **절차**
  ```powershell
  curl.exe -s http://localhost:8101/api/public/results
  curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8101/api/public/results/STAR
  curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8101/api/public/results/XYZ
  ```
- **기대 결과:** 첫 줄은 JSON(counted_rate · candidates · regions 5개), 그다음 `200`, 마지막 `404`. 로그인 없이 됨
- ☐ 통과 ☐ 실패 메모:

### TC-API-08 실시간 알림 (SSE)
- **절차:** 새 PowerShell 창에서 연결을 켜 둔 채 선관위를 진행합니다.
  ```powershell
  curl.exe -N http://localhost:8101/api/public/stream
  ```
- **기대 결과:** 처음에 `retry: 3000`, 이어서 5초 간격으로 `event: results.updated` / `data: {"last_seq": …}`. 15초 동안 소식이 없으면 `: ping`. 연결이 끊기지 않음 (3단계에서 승인하면 `event: article.published`도 들어옴)
- ☐ 통과 ☐ 실패 메모:

### TC-SYS-05 SSE 다시 연결
- **절차:** 개표 웹을 켜 둔 채 터미널 ②(M)를 끄고 5초 뒤 다시 켬
- **기대 결과:** 새로고침하지 않아도 3~10초 안에 다시 갱신이 이어짐
- ☐ 통과 ☐ 실패 메모:

### TC-SYS-02 M·K는 서로 독립
- **절차:** K 개표 웹(http://localhost:8102/)을 켜 둔 채 터미널 ②(M)만 끔
- **기대 결과:** K 웹은 계속 갱신. K 수신 모니터도 정상
- ☐ 통과 ☐ 실패 메모:

### TC-DB-07 서버를 껐다 켜도 유지
- **절차:** 개표 중간에 M 서버를 끄고 다시 켬 → `check.py counts`·수신 모니터 확인
- **기대 결과:** 최신 결과·이벤트 수가 그대로이고, seq가 끊긴 곳부터 이어서 받아짐 (같은 이벤트가 다시 생기지 않음)
- ☐ 통과 ☐ 실패 메모:

### TC-ENV-04 강의장 LAN 접속
- **절차**
  1. 서버 PC에서 `ipconfig` → IPv4 주소 확인 (예: 192.168.0.23)
  2. 방송국 서버를 `uvicorn app.main:app --host 0.0.0.0 --port 8101 --env-file .env.m --timeout-graceful-shutdown 2` 로 다시 켬
  3. 같은 와이파이의 다른 PC·휴대폰에서 http://192.168.0.23:8101/
- **기대 결과:** 개표 웹이 열림. 안 열리면 Windows 방화벽에서 Python 허용(개인 네트워크)
- ☐ 통과 ☐ 실패 메모:

---

## 6. 3단계 — 기사 흐름

> 시작 전: [1-6 초기화](#1-6-처음부터-다시-하기-리허설-전-초기화). 기자 창(reporter1)과 데스크 창(desk1)을 따로 띄웁니다.
> 처음에는 OPENAI_API_KEY를 **비운 채** 틀 기사로 테스트하고, TC-ENV-06 · TC-FN-10에서 키를 넣습니다.

### TC-UI-04 기자 화면 · TC-FN-04 기자 자동 배정
- **절차**
  1. reporter1·2·3을 각각 다른 창(일반·시크릿·다른 브라우저)으로 로그인
  2. 선관위 1배속 시작 → 첫 이벤트(약 1분, 10% 돌파)를 기다림
  3. 배정된 기사를 눌러 열고 → 제목을 조금 고쳐 **임시 저장** → 다른 기사를 눌렀다가 다시 열기
  4. **데스크로 송고**
- **기대 결과**
  - 새로고침 없이 오른쪽 아래 "새 초안이 배정됐습니다" 알림
  - 기사 1→기자1, 2→기자2, 3→기자3, 4→기자1 순서로 배정 (한 기자에게 초안·반려가 3건 쌓이면 그 기자는 건너뜀)
  - 임시 저장한 제목이 유지됨. 원고 칸 옆에 "○○자 · 약 ○○초"
  - 송고하면 상태 SUBMITTED, 제목·본문·원고 칸과 버튼이 잠김
- ☐ 통과 ☐ 실패 메모:

### TC-UI-05 데스크 화면
- **절차**
  1. 기사 3건을 서로 다른 시각에 송고
  2. desk1로 http://localhost:8101/desk
  3. 맨 위 기사 선택 → 반려 사유 칸을 비운 채 반려 버튼 확인 → 사유 입력 → 반려
  4. 다음 기사 → **승인 = 출고**
- **기대 결과**
  - 송고 도착 순(먼저 온 것이 위)으로 정렬, '기다림' 칸에 초가 늘어남
  - 사유가 비어 있으면 반려 버튼이 회색(눌리지 않음)
  - 승인하면 "출고했습니다 (대기열 #n)" 알림, 목록에서 사라짐
- ☐ 통과 ☐ 실패 메모:

### TC-SC-04 반려 후 다시 송고 · TC-DB-05 검토 이력
- **절차**
  1. 반려한 기사를 담당 기자 창에서 엽니다 (목록 '할 일'에 다시 보임)
  2. 고쳐서 다시 송고 → 데스크가 승인
  3. `python tools/check.py --env .env.m article 기사번호`
- **기대 결과**
  - 기자 화면 위쪽 빨간 칸에 "반려 (데스크1 HH:MM:SS): 사유"
  - 다시 송고한 기사가 데스크 목록 **맨 뒤**에 들어감
  - check.py 맨 아래 "검토:" 두 줄 — REJECT(사유 포함) → APPROVE
- ☐ 통과 ☐ 실패 메모:

### TC-FN-05 기사 상태 변화
- **절차:** 송고한 기사(SUBMITTED)를 기자가 http://localhost:8101/docs 의 **PUT /api/articles/{id}** 로 수정 시도 (같은 브라우저에서 기자 로그인 상태)
- **기대 결과:** 409 "SUBMITTED 상태에서는 고칠 수 없습니다". 반려된(REJECTED) 기사는 수정·송고됨
- ☐ 통과 ☐ 실패 메모:

### TC-FN-06 동시 승인 막기
- **절차:** desk1(일반 창)·desk2(시크릿 창)에서 같은 송고 기사를 열고, 거의 동시에 **승인** 클릭
- **기대 결과:** 한쪽은 "출고했습니다", 다른 쪽은 "이미 처리된 기사입니다". `check.py article 번호`의 APPROVE가 1줄
- ☐ 통과 ☐ 실패 메모:

### TC-RQ-02 역할별 접근
- **절차:** reporter1 로그인 창의 주소창에 http://localhost:8101/desk , /monitor , /onair/control 입력. desk1 창에서는 http://localhost:8101/docs 의 **POST /api/articles/{id}/submit** 실행
- **기대 결과:** 기자는 세 주소 모두 들어가지 못하고 로그인 화면을 거쳐 기자 화면(/reporter)으로 돌아감. 데스크의 송고 API는 403
- ☐ 통과 ☐ 실패 메모:

### TC-API-05 기사 API 권한
- **절차:** /docs 에서 **PUT /api/articles/{id}** 를 ① 로그아웃 상태 ② 다른 기자 ③ 담당 기자로 실행
- **기대 결과:** 401 → 403 "내게 배정된 기사만 고칠 수 있습니다" → 200
- ☐ 통과 ☐ 실패 메모:

### TC-RQ-03 사람 검수 원칙 · TC-API-06 승인하면 공개 목록에
- **절차**
  1. 초안(DRAFT)·송고(SUBMITTED)·반려(REJECTED) 기사가 각각 하나 이상 있는 상태에서
  2. `curl.exe -s http://localhost:8101/api/public/articles` 와 `python tools/check.py --env .env.m queue`
  3. 송고 기사 하나를 승인하고 2를 다시 실행
- **기대 결과**
  - 공개 목록·대기열에는 APPROVED 기사만 있음
  - 승인한 기사가 공개 목록 **맨 위**에 나타나고 개표 웹 '실시간 기사'에 NEW 표시로 바로 뜸
  - 같은 기사를 /docs에서 다시 승인하면 409
- ☐ 통과 ☐ 실패 메모:

### TC-SYS-04 OpenAI 실패에도 버티기
- **절차:** `.env.m`에 `OPENAI_API_KEY=sk-wrong` 처럼 틀린 키를 넣고 M 서버 재시작 → 개표 진행
- **기대 결과**
  - 수신 모니터 'AI 기사 생성'에 실패 횟수와 최근 오류(AuthenticationError 등)
  - 기사는 **틀 기사**(모델 칸 `template`)로 계속 만들어지고 수신·웹·송출은 멈추지 않음
  - (틀 기사를 끄려면 `LLM_FALLBACK=off` — 그때는 실패만 기록되고 기사가 생기지 않음)
- ☐ 통과 ☐ 실패 메모:

### TC-ENV-06 gpt-6-luna 연결
- **절차:** `.env.m`에 진짜 팀 키를 넣고 `python tools/check.py --env .env.m llm`
- **기대 결과:** `설정 모델: gpt-6-luna · 응답 모델: gpt-6-luna…` 와 인사 한 문장
- ☐ 통과 ☐ 실패 메모:

### TC-FN-10 기사 초안 생성 (gpt-6-luna + RAG)
- **절차:** 진짜 키로 M 서버 재시작 → 초기화 후 개표 진행 → 첫 기사 몇 건을 `check.py article 번호`로 확인
- **기대 결과**
  - 모델 칸이 `gpt-6-luna` (틀 기사가 아님)
  - "원고 ○○자 → 기준 120~180자: 통과"
  - "글에 나온 득표율·개표율 … → 일치" (다르면 그 AI 초안은 버리고 틀 기사로 바뀜)
  - "참고 자료:"에 해당 후보의 이력·공약 조각 이름 (예: 류해온 이력)
  - 제목·본문에 자료에 없는 공약이나 실존 인물·정당·작품 이름이 없음
- ☐ 통과 ☐ 실패 메모:

---

## 7. 4단계 — 송출

> 시작 전: [1-6 초기화](#1-6-처음부터-다시-하기-리허설-전-초기화). 영상이 없으면 각 구간이 제목 화면(SLATE)으로 나옵니다.

### TC-UI-09 큐시트·송출 제어
- **절차**
  1. desk1로 http://localhost:8101/onair/control
  2. `app/static/media/`에 영상 몇 개를 넣고(이름은 그 폴더 README) 하나는 일부러 빼 둠
  3. 별도 창에서 http://localhost:8101/onair/screen → 화면 클릭(소리 허용)
  4. ▶ 송출 시작 → ⏸ 일시정지 → ⏵ 재개 → ⏭ 다음 구간
- **기대 결과**
  - 큐시트 10줄. 영상 칸에 "준비됨"(초록) 또는 "⚠ 없음: intro_2.mp4"(주황)
  - 시작하면 상태 ON_AIR, 현재 구간 "1. 개표 카운트다운 영상", 방송 경과가 늘어남
  - 일시정지하면 송출 화면의 영상·소리가 멈추고, 재개하면 이어짐
  - 다음 구간을 누르면 송출 화면이 바로 다음 구간으로 넘어감
- ☐ 통과 ☐ 실패 메모:

### TC-UI-08 송출 화면 · TC-RQ-07 송출 화면 규격
- **절차:** 송출 화면을 1920×1080 창(전체 화면 F11 또는 OBS)으로 보며 LIVE 구간까지 진행. 승인 기사가 나오게 함
- **기대 결과**
  - 스크롤 없이 화면에 꼭 맞음. 왼쪽 AI 앵커 자리, 오른쪽 차트, 아래 빨간 헤드라인 띠와 검은 득표 띠
  - 영상 구간: 전체 화면 영상(없으면 제목 화면 + "영상 준비 중 · 60초")
  - 기사 낭독 중: 헤드라인 = 기사 제목, 오른쪽 차트 = 그 기사 지역(예: 별빛구) 네온 막대
  - 읽을 기사가 없을 때: 전체 현황 차트 + 배경음악(bgm.mp3가 있으면) (BR-09)
  - 아래 득표 띠는 5초마다 갱신
- ☐ 통과 ☐ 실패 메모:

### TC-API-09 송출 다음 항목
- **절차:** 송출 화면을 **닫은 상태**에서 /docs 의 **GET /api/onair/next** 를 여러 번 실행 → ITEM이 오면 그 queue_id로 **POST /api/onair/queue/{queue_id}/done** → 다시 next
- **기대 결과:** 영상 구간(VIDEO 또는 SLATE) 5번 뒤 LIVE에서 ITEM(기사) 또는 BOARD(대기열이 빔). done 뒤 next는 다음 기사, 기사가 없으면 BOARD
- ☐ 통과 ☐ 실패 메모:

### TC-FN-07 대기열 순서와 건너뛰기
- **절차**
  1. 송출 화면을 닫아 두고 기사 2~3건을 승인(대기 WAITING)
  2. 당선 유력·확실·확정 중 하나를 승인
  3. 같은 지역·같은 유형(예: 전체 개표율) 기사가 대기 중일 때 그보다 새 기사를 승인
  4. `python tools/check.py --env .env.m queue` → 송출 화면 열기
- **기대 결과:** 당선 기사(우선 1)가 먼저 나감. 옛 기사는 상태 SKIPPED. 나머지는 우선순위(1위 변동 3 → 개표율 5) 다음 승인 순
- ☐ 통과 ☐ 실패 메모:

### TC-DB-06 대기열 연결
- **절차:** 기사를 몇 건 승인한 뒤 `check.py queue`
- **기대 결과:** "기사상태"가 모두 APPROVED, 키를 넣었다면 음성 파일 경로 뒤에 "(있음)", 마지막 줄 "문제 0건"
- ☐ 통과 ☐ 실패 메모:

### TC-ENV-07 TTS 연결과 발음
- **절차:** `python tools/check.py --env .env.m tts "기호 1번 윤별하, 2번 류해온, 3번 오로라 후보입니다."` → `app/static/tts/check_tts.wav` 재생
- **기대 결과:** 파일이 재생되고, 길이가 예상과 비슷함. 세 이름이 바르게 들림 ('류해온'이 '유해온'으로 들려도 되는지 팀이 정함)
- ☐ 통과 ☐ 실패 메모:

### TC-RQ-04 출고 지연 시간
- **절차:** 진짜 키로 기사 10건을 승인 → 승인 순간 개표 웹(다른 창)에 뜨는 시간을 눈으로 확인 → `python tools/check.py --env .env.m latency`
- **기대 결과:** 웹에는 3초 안에 뜸. latency 표의 '걸린 초'가 모두 15 이하(통과)
- ☐ 통과 ☐ 실패 메모:

### TC-FN-08 TTS 다시 만들기 (화면으로)
- **절차:** 자동 테스트로 규칙을 확인했으므로, 실제로는 음성 실패 상황만 봅니다. 진짜 키 상태에서 `.env.m`의 `TTS_MODEL=없는모델`로 바꾸고 재시작 → 기사 1건 승인
- **기대 결과:** 데스크 화면에 "⚠ 기사 n 음성 생성 실패 — 방송에서 건너뜀 (웹 게시는 유지)" 알림. 대기열 상태 SKIPPED, 개표 웹에는 기사가 그대로 있음 (끝나면 TTS_MODEL 되돌리기)
- ☐ 통과 ☐ 실패 메모:

### TC-SC-03 기사 한 건의 한 바퀴
- **절차:** 송출 화면을 켜 두고 LIVE 중에 이벤트 → 초안 배정 → 기자 송고 → 데스크 승인
- **기대 결과:** 승인 후 3초 안에 개표 웹에 기사, 15초 안에 대기열 WAITING → 앞선 기사가 끝나면 AI 앵커(TTS)가 원고를 읽고, 그동안 오른쪽 차트가 그 지역으로 바뀜
- ☐ 통과 ☐ 실패 메모:

### TC-SYS-06 OBS 캡처
- **절차**
  1. 송출 PC 브라우저에서 desk1 로그인(쿠키가 OBS와 공유되지 않으므로 아래 3 참고)
  2. OBS → 소스 + → 브라우저 → URL `http://localhost:8101/onair/screen`, 너비 1920 · 높이 1080, "OBS를 통해 오디오 제어" 체크
  3. OBS 소스 우클릭 → 상호작용 → 로그인 화면이 나오면 desk1로 로그인 → 송출 화면 클릭
- **기대 결과:** 화면이 꽉 차게 잡히고, 기사 낭독·영상 소리가 OBS 오디오 믹서에 들어감
- ☐ 통과 ☐ 실패 메모:

### TC-SC-01 20분 방송 한 바퀴 (리허설)
- **절차**
  1. [1-6 초기화](#1-6-처음부터-다시-하기-리허설-전-초기화), 선관위 5배속
  2. 큐시트 화면 ▶ 송출 시작, 송출 화면 클릭
  3. 방송 4:30(LIVE 시작)에 선관위 ▶ 시작
  4. 기자 3명·데스크 2명이 기사를 계속 송고·승인
- **기대 결과**
  - 카운트다운 → 오프닝 → 후보 소개 3개 → LIVE(기사 낭독·전체 현황) → **당선 확정 기사** → 당선자 인터뷰 → 이력·공약 → 최종 정리 → 마무리 순서로 멈춤 없이 끝나고 상태 ENDED
  - 큐시트 화면 '당선자'에 윤별하
  - 5배속은 개표만 빨라지고 영상 구간은 실제 길이라 전체 약 12분. 영상 점검이 끝났으면 ⏭ 다음 구간으로 건너뛰며 줄일 수 있음
- ☐ 통과 ☐ 실패 메모:

### TC-SC-05 당선 확정이 늦을 때 (BR-10)
- **절차:** LIVE 중 당선 유력이 나오기 전에 선관위를 ⏸ 일시정지 → LIVE 11분이 지나도록 둠 → 큐시트 ⏭ 다음 구간
- **기대 결과:** 시간이 지나도 LIVE가 이어지고 전체 현황 화면이 멈추지 않음. 다음 구간을 누르면 인터뷰·이력을 건너뛰고 "9. 최종 결과 정리"로 감
- ☐ 통과 ☐ 실패 메모:

---

## 8. 5단계 — 마무리(아바타·시연)

### TC-FN-09 아바타 대기 한도 (연습용 mock)
- **절차**
  1. `.env.m`에 `AVATAR_PROVIDER=mock`, `AVATAR_TIMEOUT_SEC=10`, `AVATAR_MOCK_DELAY_SEC=30` → 재시작
  2. 당선 단계 기사(유력·확실·확정) 1건 승인 → 큐시트 화면 대기열 확인
  3. 이번에는 `app/static/avatar/sample.mp4`를 넣고 `AVATAR_MOCK_DELAY_SEC=5`로 재시작 → 당선 기사 승인
- **기대 결과**
  - 2: 약 10초 동안 PREPARING → 한도가 지나 WAITING, 음성 칸 TTS(키가 없으면 '브라우저 음성'). 송출 화면은 앵커 영상 + 음성으로 읽음
  - 3: 음성 칸 AVATAR, 송출 화면 왼쪽에 sample.mp4가 나옴
  - 실제 아바타 서비스는 `app/workers/avatar.py`에 연결 코드를 추가한 뒤 같은 방법으로 확인
- ☐ 통과 ☐ 실패 메모:

### TC-SC-06 클라우드 시연 모드 · TC-ENV-05 Render 배포
- **절차**
  1. Render(무료)에서 New → Blueprint → nec-server 저장소(render.yaml) → ADMIN_KEY, STATION_KEY_M, STATION_KEY_K 입력
  2. station-onair도 Blueprint → NEC_BASE_URL(선관위 Render 주소), NEC_API_KEY(위 STATION_KEY_M), DEFAULT_PASSWORD, OPENAI_API_KEY 입력
  3. 배포가 끝나면 방송국 주소를 열고, 15분 이상 아무도 접속하지 않은 뒤 다시 열기
- **기대 결과**
  - 처음 접속 때 1분 안에 깨어남
  - 아무도 누르지 않아도 개표가 시작되고, 기사도 자동 송고·승인(DEMO_AUTO_APPROVE)되어 웹에 뜸
  - 개표가 끝나고 30초 뒤 처음부터 다시 반복
  - ※ 무료 서버는 잠들면 데이터가 지워집니다. 발표 영상은 미리 녹화해 둡니다.
- ☐ 통과 ☐ 실패 메모:

### TC-ENV-03 Git에 올리지 않을 것
- **절차:** 리허설을 한 번 끝낸 뒤 두 저장소에서 `git status`
- **기대 결과:** `.env`, `.env.m`, `.env.k`, `*.db`, `vectordb/`, `app/static/tts/`, `*.mp4`, `*.wav`가 목록에 **나오지 않음** (bgm.mp3는 올려도 됨)
- ☐ 통과 ☐ 실패 메모:

### TC-SC-02 후반: 방송국 이벤트까지 같은지
- **절차:** [1-6 초기화](#1-6-처음부터-다시-하기-리허설-전-초기화) → 5배속 끝까지 → `check.py events > run1.txt` → 다시 초기화 → 끝까지 → `check.py events > run2.txt` → `fc run1.txt run2.txt`
- **기대 결과:** 이벤트 번호를 빼면 seq·이벤트·지역·후보·개표율이 모두 같음 ("차이점이 없습니다" 또는 번호 차이만)
- ☐ 통과 ☐ 실패 메모:

---

## 9. 문제가 생기면

| 증상 | 원인과 해결 |
|---|---|
| `ModuleNotFoundError: fastapi` | 가상환경이 꺼져 있음 → `.venv\Scripts\activate` 후 다시 |
| 서버가 `[설정 오류] .env에 다음 값이 없습니다` 하고 멈춤 | 말한 값을 .env(.env.m/.env.k)에 채움 |
| 수신 모니터 "선관위 응답 401" | NEC_API_KEY가 선관위 seed 키와 다름 → 다시 복사 (앞뒤 공백 주의) |
| 수신 모니터 "연결 실패" | 선관위 서버(8000)가 꺼져 있거나 NEC_BASE_URL 주소가 틀림 |
| 개표 웹 숫자가 그대로 | 선관위 시뮬레이터를 시작했는지 확인 (관리 화면 RUNNING) |
| 기사가 안 생김 | 이벤트가 생겼는지 `check.py events` → 있으면 서버 터미널의 article 오류 확인 |
| 기자 화면에 알림이 안 옴 | 다른 기자에게 배정된 것일 수 있음 → `check.py article 번호`의 담당 확인 |
| 송출 화면에서 소리가 안 남 | 화면을 한 번 클릭했는지, 브라우저 탭 음소거가 아닌지. 키가 없으면 브라우저 음성(한국어 음성이 설치돼 있어야 함) |
| 송출 화면이 로그인 화면으로 바뀜 | 데스크로 로그인하지 않았음 (OBS는 상호작용으로 로그인) |
| `database is locked` | 같은 DB 파일을 쓰는 서버가 두 개 켜져 있음 → M·K의 STATION_CODE가 다른지 확인 |
| 방송국 서버가 Ctrl+C로 안 꺼짐 | 열린 화면(개표 웹·송출 화면)의 실시간 연결이 끝나길 기다리는 중 → **Ctrl+C를 한 번 더** 누르거나 터미널 휴지통 아이콘. 켤 때 `--timeout-graceful-shutdown 2`를 붙이면 2초 뒤 꺼짐 |
| 처음부터 다시 하고 싶음 | [1-6 초기화](#1-6-처음부터-다시-하기-리허설-전-초기화) |

오류가 나면 **터미널의 마지막 20줄**을 복사해 두면 원인을 찾기 쉽습니다.

---

## 10. 결과 기록표

리허설마다 날짜를 적고 ○(통과) / ×(실패) / -(안 함)을 표시합니다.

| 단계 | TC | 확인할 것 | 방식 | 1차 | 2차 | 메모 |
|---|---|---|---|---|---|---|
| 1 | TC-API-01 | 선관위 API 키 인증 | 자동 | | | |
| 1 | TC-API-02 | 선거 기본정보 형식 (계약) | 자동 | | | |
| 1 | TC-API-03 | 개표 변경분 형식 (계약) | 자동 | | | |
| 1 | TC-API-04 | 시뮬레이터 제어 API | 자동 | | | |
| 1 | TC-DB-01 | 처음 넣는 데이터 (seed) | 자동 | | | |
| 1 | TC-DB-02 | 개표 스냅숏 쌓기 | 자동 | | | |
| 1 | TC-ENV-01 | 새 PC 설치 | 수동 | | | |
| 1 | TC-ENV-02 | .env 준비 | 수동 | | | |
| 1 | TC-ENV-03 | Git에 올리지 않을 것 | 자동 | | | |
| 1 | TC-FN-01 | 변경분만 내려주기 | 자동 | | | |
| 1 | TC-RQ-05 | 비밀 정보 보호 | 자동 | | | |
| 1 | TC-UI-02 | 개표 시뮬레이터 | 수동 | | | |
| 2 | TC-API-07 | 공개 결과 API | 자동 | | | |
| 2 | TC-API-08 | 실시간 알림 (SSE) | 수동 | | | |
| 2 | TC-DB-03 | 최신 결과 한 벌 | 자동 | | | |
| 2 | TC-DB-04 | 이벤트 중복 막기 | 자동 | | | |
| 2 | TC-DB-07 | 서버를 껐다 켜도 유지 | 수동 | | | |
| 2 | TC-ENV-04 | 강의장 LAN 접속 | 수동 | | | |
| 2 | TC-FN-02 | 개표율·1위 변동 감지 | 자동 | | | |
| 2 | TC-FN-03 | 당선 판정 | 자동 | | | |
| 2 | TC-RQ-01 | 고정 계정 5개 | 자동 | | | |
| 2 | TC-RQ-06 | 누구나 보는 웹 | 수동 | | | |
| 2 | TC-SC-02 | 같은 결과 다시 재현 | 수동 | | | |
| 2 | TC-SYS-01 | 서버 3대 함께 실행 | 수동 | | | |
| 2 | TC-SYS-02 | M·K는 서로 독립 | 수동 | | | |
| 2 | TC-SYS-03 | 선관위 장애 뒤 따라잡기 | 수동 | | | |
| 2 | TC-SYS-05 | SSE 다시 연결 | 수동 | | | |
| 2 | TC-UI-01 | 로그인 화면 | 수동 | | | |
| 2 | TC-UI-03 | 수신 모니터 | 수동 | | | |
| 2 | TC-UI-06 | 실시간 개표 웹 | 수동 | | | |
| 2 | TC-UI-07 | 지역별 상세 | 수동 | | | |
| 3 | TC-API-05 | 기사 API 권한 (BR-02) | 자동 | | | |
| 3 | TC-API-06 | 승인하면 공개 목록에 | 자동 | | | |
| 3 | TC-DB-05 | 검토 이력 | 자동 | | | |
| 3 | TC-ENV-06 | gpt-6-luna 연결 | 수동 | | | |
| 3 | TC-FN-04 | 기자 자동 배정 (BR-05) | 자동 | | | |
| 3 | TC-FN-05 | 기사 상태 변화 (BR-03) | 자동 | | | |
| 3 | TC-FN-06 | 동시 승인 막기 (BR-04) | 자동 | | | |
| 3 | TC-FN-10 | 기사 초안 생성 (gpt-6-luna + RAG) | 수동 | | | |
| 3 | TC-RQ-02 | 역할별 접근 | 자동 | | | |
| 3 | TC-RQ-03 | 사람 검수 원칙 (BR-01) | 자동 | | | |
| 3 | TC-SC-04 | 반려 후 다시 송고 | 수동 | | | |
| 3 | TC-SYS-04 | OpenAI 실패에도 버티기 | 수동 | | | |
| 3 | TC-UI-04 | 기자 화면 | 수동 | | | |
| 3 | TC-UI-05 | 데스크 화면 | 수동 | | | |
| 4 | TC-API-09 | 송출 다음 항목 | 자동 | | | |
| 4 | TC-DB-06 | 대기열 연결 | 자동 | | | |
| 4 | TC-ENV-07 | TTS 연결과 발음 | 수동 | | | |
| 4 | TC-FN-07 | 대기열 순서와 건너뛰기 (BR-06·07) | 자동 | | | |
| 4 | TC-FN-08 | TTS 다시 만들기 (BR-08) | 자동 | | | |
| 4 | TC-RQ-04 | 출고 지연 시간 | 수동 | | | |
| 4 | TC-RQ-07 | 송출 화면 규격 | 수동 | | | |
| 4 | TC-SC-01 | 20분 방송 한 바퀴 (5배속 리허설) | 수동 | | | |
| 4 | TC-SC-03 | 기사 한 건의 한 바퀴 | 수동 | | | |
| 4 | TC-SC-05 | 당선 확정이 늦을 때 (BR-10) | 수동 | | | |
| 4 | TC-SYS-06 | OBS 캡처 | 수동 | | | |
| 4 | TC-UI-08 | 송출 화면 | 수동 | | | |
| 4 | TC-UI-09 | 큐시트·송출 제어 | 수동 | | | |
| 5 | TC-ENV-05 | Render 배포 | 수동 | | | |
| 5 | TC-FN-09 | 아바타 대기 한도 (BR-11) | 자동 | | | |
| 5 | TC-SC-06 | 클라우드 시연 모드 | 수동 | | | |

자동 테스트 항목도 수동 절차가 있는 것은 리허설 때 화면으로 한 번 더 확인합니다.
