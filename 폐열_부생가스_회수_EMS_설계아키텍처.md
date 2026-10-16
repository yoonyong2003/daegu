# 석유화학 발전소 폐열·부생가스 회수 EMS 설계 아키텍처

- 작성일: 2026-10-16
- 대상: 이 EMS를 구현할 팀과 IT·OT 담당
- 기반 문서: `EMS_대시보드_단계별_구현가이드.md` (이하 **구현가이드**), `EMS_도입_필요성_2026_신규동인.md`
- 짝 문서: `폐열_부생가스_회수_EMS_기능정의서.md` (이하 **기능정의서**). 화면과 기능의 ID(FR-xxx)는 그 문서와 같다

## 이 문서를 읽는 법

| 표기 | 뜻 |
| --- | --- |
| 참조 | 구현가이드의 설계를 그대로 가져왔다 |
| 변경 | 구현가이드의 설계를 이 발전소에 맞게 바꿨다 |
| 제안 | 출처가 아니라 이 문서의 판단이다 |
| 가정 | 현장 데이터가 없어서 임시로 둔 값이다 |

코드와 SQL은 **설계 예시**이고 실행해 검증하지 않았다. 구현 전에 사용할 DB 버전의 공식 문서로 문법과 설정을 확인한다 (참조 §3.1의 TimescaleDB 실시간 집계 기본값 변경처럼 버전에 따라 달라지는 것이 있다).

---

## 0. 요약

1. **구조는 4계층**이다. 현장(OT) → 수집·저장 → 서비스(계산) → 화면. 구현가이드 §3.4의 `collector / db / services / web / simulator` 구성을 유지한다.
2. **읽기가 기본, 쓰기는 예외다.** EMS는 DCS에서 측정값을 읽기 전용으로 받는다. 설비로 가는 쓰기는 운전원 승인과 인터락 검증을 거친 별도 경로로만 열고, 6단계 이후에 연다.
3. **집계는 한 곳에서 한다.** 화면마다 따로 계산하지 않고 DB의 집계 뷰와 서비스 한 곳의 결과를 함께 쓴다 (참조 §1.2 규칙 3).
4. **스키마는 더하기만 한다.** 단계가 올라도 앞 단계가 깨지지 않는다 (참조 §1.2 규칙 2).
5. **EMS가 멈춰도 발전소는 멈추지 않는다.** DCS의 기본 제어와 안전계장시스템(SIS)은 EMS와 독립이다.
6. **참조에서 가장 크게 바꾼 것**: 구현가이드의 중심 데이터는 수전 계량(전기)이다. 여기서는 **유량, 온도, 조성, 압력(열·가스)** 이 중심이고, 모든 지표가 **에너지 수지**를 통해 만들어진다.

---

## 1. 설계 원칙

참조 §1.2, §1.3의 원칙을 가져오고, 발전소에 필요한 것을 더했다.

| 번호 | 원칙 | 출처 |
| --- | --- | --- |
| 1 | 한 단계에 한 사람의 한 결정만 더한다 | 참조 |
| 2 | 스키마는 더하기만 한다 | 참조 |
| 3 | 집계는 한 곳에서 한다 | 참조 |
| 4 | 데이터가 없으면 빈 상태를 보여주고, 가짜 값을 채우지 않는다 | 참조 |
| 5 | 화면은 등록표로 켜고 끈다 | 참조 |
| 6 | 끝 조건을 통과한 뒤 다음 단계로 간다 | 참조 |
| 7 | 화면을 복사해 늘리지 않는다. 설비나 스트림은 변수로 바꿔 한 화면을 다시 쓴다 | 참조 |
| 8 | 외부 API는 화면이 직접 부르지 않는다. 수집기가 받아 DB에 넣는다 | 참조 §4.6 |
| 9 | **EMS는 안전 시스템이 아니다.** SIS와 DCS 기본 제어를 대신하거나 거치지 않는다 | 제안 |
| 10 | **쓰기는 별도 경로, 별도 승인, 별도 로그** | 제안 |
| 11 | **통신 두절과 품질 불량은 "마지막 안전 상태"로 복귀한다** | 제안 |
| 12 | 모든 수치는 출처(계측기 → 원시 자료 → 집계 → 보고 수치)를 추적할 수 있다 | 신규동인 §2.5 |

---

## 2. 전체 구조

### 2.1 논리 구조

```
┌──────────────────────────────────────────────────────────────┐
│ 사용자  경영자 · 에너지관리자 · 운전원 · 설비 · 안전환경 · 구매   │
└───────────────▲──────────────────────────────┬───────────────┘
                │ 화면·알림                     │ 수락·거부 입력
┌───────────────┴──────────────────────────────▼───────────────┐
│ web   화면 등록표에 따라 열리는 화면 7개 + 관리자 화면          │
├──────────────────────────────────────────────────────────────┤
│ services  balance · alarm · economics · baseline · forecast   │
│           optimizer · interlock · recommendation              │
├──────────────────────────────────────────────────────────────┤
│ db  원시 시계열 · 집계 뷰 · 설정 · 알람 · 권고 · 감사 로그       │
├──────────────────────────────────────────────────────────────┤
│ collector  historian · opcua/modbus · analyzer · weather ·    │
│            market(연료·유가) · (후반) approved-writer           │
└───────────────▲──────────────────────────────┬───────────────┘
        읽기 전용│                              │ 승인된 쓰기만
═══════ IT·OT 경계 (방화벽 · 단방향 · 분리된 쓰기 경로) ═══════════
┌───────────────┴──────────────────────────────▼───────────────┐
│ OT  센서·계측기 · DCS · SCADA · Historian · SIS(별도·불간섭)    │
│     회수 설비: HRSG · ORC · CHP · 열교환기 · 부생가스 라인       │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 구성요소

| 구성요소 | 역할 | 단계 | 비고 |
| --- | --- | --- | --- |
| **collector/historian** | 기존 Historian·DCS에서 이력과 실시간 값을 읽는다 | 0 | 기존 시스템이 있으면 먼저 붙는다 |
| **collector/opcua, modbus** | 현장 계측기·분석기에서 직접 읽는다 | 0~3 | 추가 계측점은 여기로 |
| **collector/analyzer** | 부생가스 조성(GC) 분석값을 읽는다 | 0~1 | 조성이 없으면 발열량 계산이 안 된다 |
| **collector/weather** | 기온·습도 예보 | 5 | 예측과 기준선 보정에 쓴다 |
| **collector/market** | 유가·연료 가격 | 2 | 단가 테이블로 입력하는 방식도 지원 |
| **collector/approved-writer** | 승인된 권고를 DCS에 쓰는 유일한 경로 | 6 이후 | 기본은 꺼 둔다 |
| **db** | 시계열(TimescaleDB 예시), 집계 뷰, 설정, 로그 | 0~ | 참조 §3.1 |
| **services/balance** | 에너지 수지, 회수율, 연료 대체율 | 0~3 | 모든 지표의 뿌리 |
| **services/alarm** | 규칙 판정, 묶기, 확인과 메모 | 1 | |
| **services/economics** | 연료비, 절감액, 발전 단가, 시나리오 | 2 | |
| **services/baseline** | 기준선 모델, 정규화 | 2~4 | M&V의 기반 |
| **services/forecast** | 발생량·수요 예측 | 5 | 화면과 분리해서 정해진 시각에 돈다 |
| **services/optimizer** | 배분 최적화 | 6 | |
| **services/interlock** | 권고가 안전 규칙을 지키는지 검증 | 6 | 검증 실패 권고는 화면에 안 나온다 |
| **web** | 화면과 입력 흐름 | 0~ | 0단계는 Grafana로 시작 |
| **simulator** | 실제 계측이 없을 때 쓰는 발생·회수 생성기 | 0~ | §10 |

---

## 3. 데이터 흐름

### 3.1 실시간 경로

```
계측기·DCS → collector → (정제·검증·표준화) → meter_reading
                                                  │
                    ┌─────────────────────────────┤
                    ▼                             ▼
             연속 집계 뷰                    서비스(balance, alarm)
         (1분 · 15분 · 1시간)                     │
                    │                             ▼
                    └──────────────►  화면으로 실시간 메시지 전송
```

### 3.2 화면으로 보내는 실시간 메시지 (변경)

구현가이드 §3.1의 메시지 형식을 이 발전소에 맞게 바꿨다. 없는 항목은 빼고 보낸다.

```json
{
  "t": "2026-10-16T10:07:24+09:00",
  "fuel_sub": { "pct": 18.4, "target_pct": 20.0 },
  "gen_mw": 42.0,
  "ext_fuel": { "kind": "HFO", "tph": 3.1 },
  "recovered": { "heat_gj_h": 210.5, "offgas_gj_h": 86.2 },
  "wasted": { "flare_nm3h": 1240, "flue_t_out_c": 214 },
  "units": [
    { "id": "HRSG-1", "state": "run" },
    { "id": "ORC-1", "state": "run" }
  ],
  "quality": { "data_age_s": 3, "balance_err_pct": 2.1 }
}
```

- `fuel_sub`, `recovered`, `wasted`는 0단계부터 채운다.
- 예측(`forecast`)은 5단계, 권고(`reco`)는 6단계부터 채운다.
- 값의 이름과 단위는 예시이고, 실제 태그가 정해지면 맞춘다.

### 3.3 주기 (가정)

| 데이터 | 수집 주기 | 이유 |
| --- | --- | --- |
| 유량·온도·압력(빠른 신호) | 1~5초 | 급변 감지와 알람 |
| 조성 분석(GC) | 1~15분 | 분석기의 한계 |
| 연료 단가·유가 | 일 또는 입력 시 | 느리게 변함 |
| 기상 예보 | 1시간 | |
| 예측 갱신 | 매일 정해진 시각과 매 15분 | 참조 §4.6 |
| 최적화 재계산 | 5~15분, 일간 계획은 24시간 앞까지 | 기능정의서 §4.7 |

**저장량 가늠 (가정)**: 태그 500개를 5초 주기로 저장하면 하루에 500 × 86,400 ÷ 5 = **8,640,000행**이다. 실제 태그 수와 주기가 정해지면 다시 계산한다. 원시 자료는 압축과 보관 정책(§5.5)으로 관리한다.

---

## 4. 데이터 모델

### 4.1 태그 규칙 (변경)

구현가이드의 6칸 규칙을 그대로 쓰고 각 칸의 뜻만 바꿨다.

```
{site}.{building}.{floor}.{zone}.{equipment}.{measurement}
```

| 칸 | 이 발전소에서의 뜻 | 예 |
| --- | --- | --- |
| site | 사업장 | `S1` |
| building | 발전부 | `PW` |
| floor | 층 대신 `0F`로 고정 | `0F` |
| zone | 스트림 또는 설비 구역 | `HRSG`, `OFFGAS`, `FLARE`, `ORC`, `UTL` |
| equipment | 설비 번호 | `HRSG-1`, `HDR-A` |
| measurement | 측정 항목 | `flue_t_out`, `flow`, `lhv`, `press` |

| 예시 태그 | 뜻 |
| --- | --- |
| `S1.PW.0F.HRSG.HRSG-1.flue_t_in` | HRSG-1 배가스 입구 온도 |
| `S1.PW.0F.HRSG.HRSG-1.flue_t_out` | HRSG-1 배가스 출구 온도 |
| `S1.PW.0F.OFFGAS.HDR-A.flow` | 부생가스 헤더 A 유량 |
| `S1.PW.0F.OFFGAS.HDR-A.lhv` | 부생가스 헤더 A 저위발열량 |
| `S1.PW.0F.FLARE.FL-1.flow` | 플레어 유량 |
| `S1.PW.0F.ORC.ORC-1.kw` | ORC 발전 출력 |
| `S1.PW.0F.UTL.FUEL-EXT.flow` | 외부 연료 사용량 |

규칙: 단위는 SI 기준으로 통일하고 변환은 수집 단계에서만 한다. 태그마다 유효 범위, 주기, 교정일, 책임자를 `tag` 테이블에 등록한다.

### 4.2 테이블 (제안, 더하기만 한다)

마이그레이션은 단계별로 파일을 나눈다 (참조 §3.4 `000_base, 010_alarm, 020_tariff` 방식).

**000_base: 자산, 태그, 원시 자료**

```sql
CREATE TABLE asset (
  asset_id   text PRIMARY KEY,           -- 예: HRSG-1
  name       text NOT NULL,
  kind       text NOT NULL,              -- boiler, hrsg, orc, hx, header, flare, ...
  zone       text NOT NULL,
  rated      double precision,
  rated_unit text,
  parent_id  text REFERENCES asset(asset_id)
);

CREATE TABLE tag (
  tag         text PRIMARY KEY,          -- 4.1의 규칙
  asset_id    text REFERENCES asset(asset_id),
  medium      text,                      -- flue_gas, offgas, steam, water, power, fuel
  measurement text NOT NULL,
  unit        text NOT NULL,
  valid_min   double precision,
  valid_max   double precision,
  sample_s    integer,
  calibrated  date,
  owner       text
);

CREATE TABLE meter_reading (
  ts      timestamptz      NOT NULL,
  tag     text             NOT NULL,
  value   double precision,
  quality smallint         DEFAULT 0     -- 0 정상, 1 추정(보정), 2 누락
);
SELECT create_hypertable('meter_reading', 'ts');

CREATE MATERIALIZED VIEW meter_1m
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 minute', ts) AS bucket, tag,
       avg(value) AS avg_value, max(value) AS max_value, count(*) AS n
FROM meter_reading
GROUP BY bucket, tag;

CREATE MATERIALIZED VIEW meter_15m
WITH (timescaledb.continuous) AS
SELECT time_bucket('15 minutes', bucket) AS bucket, tag,
       sum(avg_value * n) / sum(n) AS avg_value, max(max_value) AS max_value
FROM meter_1m
GROUP BY 1, tag;
```

연속 집계는 새 자료가 들어오면 바뀐 구간만 다시 계산한다 (참조 §3.1, [S08]). **실시간 집계는 TimescaleDB 2.13부터 기본으로 꺼져 있다.** 0단계 "지금" 화면처럼 방금 값이 필요한 곳은 이 설정을 켜거나 원시 테이블을 따로 조회한다.

**010_alarm: 알람** (참조 §4.2와 같음)

```sql
CREATE TABLE alarm_rule (
  rule_id text PRIMARY KEY,
  level   text NOT NULL,                 -- 주의, 경고, 위험
  expr    text NOT NULL,
  advice  text                           -- 권장 조치 문구
);

CREATE TABLE alarm_event (
  id         bigserial PRIMARY KEY,
  rule_id    text REFERENCES alarm_rule(rule_id),
  tag        text,
  started_at timestamptz NOT NULL,
  acked_at   timestamptz,
  acked_by   text,
  cleared_at timestamptz,
  note       text
);
```

**020_fuel: 연료 단가와 배출계수** (변경: 요금표 대신 연료 단가)

```sql
CREATE TABLE fuel_price (
  valid_from  date NOT NULL,
  fuel        text NOT NULL,             -- HFO, LNG, ...
  unit_price  double precision NOT NULL, -- 원/단위
  unit        text NOT NULL,             -- t, Nm3, GJ
  lhv_mj_per_unit double precision,      -- 발열량 (성적서 기준)
  source      text,
  PRIMARY KEY (valid_from, fuel)
);

CREATE TABLE emission_factor (
  valid_from date NOT NULL,
  medium     text NOT NULL,
  factor     double precision NOT NULL,
  source     text NOT NULL,
  PRIMARY KEY (valid_from, medium)
);
```

**030_balance: 에너지 수지와 기준선**

```sql
CREATE TABLE balance_period (
  period_start timestamptz NOT NULL,
  period_end   timestamptz NOT NULL,
  scope        text NOT NULL,            -- plant, HRSG-1, OFFGAS, ...
  input_mj     double precision,
  recovered_mj double precision,
  loss_mj      double precision,
  consumed_mj  double precision,
  err_pct      double precision,
  PRIMARY KEY (period_start, scope)
);

CREATE TABLE baseline (
  baseline_id  text PRIMARY KEY,
  period_start date NOT NULL,
  period_end   date NOT NULL,
  model        text NOT NULL,            -- 예: 부하·외기온 회귀
  params       jsonb NOT NULL,
  approved_on  date,
  approved_by  text
);

CREATE TABLE saving_daily (
  day               date NOT NULL,
  baseline_id       text REFERENCES baseline(baseline_id),
  ext_fuel_saved_mj double precision,
  saving_won        double precision,
  co2_saved_t       double precision,
  PRIMARY KEY (day, baseline_id)
);
```

**040_forecast: 예측** (참조 §4.6과 같음)

```sql
CREATE TABLE forecast (
  issued_at  timestamptz NOT NULL,       -- 예측을 만든 시각
  target_ts  timestamptz NOT NULL,       -- 예측 대상 시각
  tag        text        NOT NULL,
  model      text        NOT NULL,       -- 예: chronos-2, seasonal-naive
  q10 double precision, q50 double precision, q90 double precision
);
```

**050_recommendation: 권고와 응답**

```sql
CREATE TABLE recommendation (
  rec_id       bigserial PRIMARY KEY,
  issued_at    timestamptz NOT NULL,
  valid_until  timestamptz NOT NULL,
  policy       text NOT NULL,            -- cost, carbon
  action       jsonb NOT NULL,           -- 대상, 값, 변화량
  rationale    text NOT NULL,            -- 근거 (FR-602)
  exp_saving_won double precision,
  interlock_ok boolean NOT NULL          -- 검증 통과 여부
);

CREATE TABLE operator_response (
  rec_id       bigint REFERENCES recommendation(rec_id),
  responded_at timestamptz NOT NULL,
  user_id      text NOT NULL,
  decision     text NOT NULL,            -- accept, reject
  reason       text
);
```

**060_audit: 쓰기와 설정 변경 감사**

```sql
CREATE TABLE command_audit (
  cmd_id      bigserial PRIMARY KEY,
  rec_id      bigint REFERENCES recommendation(rec_id),
  issued_at   timestamptz NOT NULL,
  target_tag  text NOT NULL,
  old_value   double precision,
  new_value   double precision,
  approved_by text NOT NULL,
  result      text NOT NULL              -- ok, rejected_by_interlock, timeout, rolled_back
);

CREATE TABLE config_audit (
  at       timestamptz NOT NULL,
  user_id  text NOT NULL,
  target   text NOT NULL,                -- 태그, 임계치, 기준선, 목표
  old_val  jsonb,
  new_val  jsonb
);
```

### 4.3 보관 정책 (제안)

| 데이터 | 해상도 | 보관 | 이유 |
| --- | --- | --- | --- |
| 원시 측정값 | 초~분 | 1년 | 재계산, 상세 분석 |
| 집계 | 1분, 15분, 1시간 | 5년 이상 | 추이, 성과 보고 |
| 알람·권고·응답·쓰기·설정 감사 | 이벤트 | 규정에 따름 (예: 5년 이상) | 사고 조사, 감사 |
| 기준선과 설정 변경 이력 | 이력 | 영구 | 절감액 재검증 |
| 배출 관련 자료 | 규제 주기 | 규제 기준 | 법적 증빙 |

보관 기간은 사내 규정과 법규를 확인해서 확정한다.

---

## 5. 데이터 품질

| 규칙 | 판정 기준 | 처리 |
| --- | --- | --- |
| 범위 | 태그의 유효 범위 밖 | 품질 플래그, 계산에서 제외 |
| 고정값 | 일정 시간 변화 없음 | "센서 멈춤 의심" 플래그 |
| 급변 | 물리적으로 불가능한 변화율 | 플래그, 이전 값 유지 |
| 결측 | 수신 없음 | 선형 보간 또는 제외. 보정값은 `quality=1`로 표시 |
| 수지 정합성 | 에너지 수지 오차 > 기준(가정 5%) | `balance_error` 알람, 계측 점검 요청 |
| 시간 | 시계 어긋남 | NTP 동기화 확인 |

품질 지표는 수신율(목표 99% 이상, 가정)과 유효값 비율(목표 98% 이상, 가정)로 관리한다. 모든 보정은 값에 표시를 남기고, 보정된 값을 근거로 한 계산은 화면에 "보정 포함"을 보인다.

---

## 6. 서비스 설계

### 6.1 balance: 에너지 수지

**폐열**

```
Q_flue  = m_dot × cp × (T_in − T_out)                 [kW 또는 GJ/h]
```

**부생가스**

```
Q_offgas = V_dot × LHV                                [GJ/h]
LHV      = Σ (조성 i 의 몰분율 × 성분 i 의 발열량)      ← 조성 분석값에서 계산
```

**수지와 지표**

```
수지 오차   = (투입 − 회수 − 손실 − 소비) ÷ 투입
회수율      = Q_회수 ÷ Q_발생
활용률      = (V_발생 − V_플레어) ÷ V_발생
연료 대체율 = 회수 에너지가 대체한 열량 ÷ 전체 연료 열량
```

- 모든 계산은 DB 집계 뷰 위에서 **한 곳(balance 서비스)** 에서 하고 결과를 `balance_period`에 넣는다. 화면은 이 값만 읽는다.
- 부생가스 조성이 분석 주기 사이에 비면, 마지막 조성값을 쓰되 "추정"으로 표시하고 `lhv_swing` 알람 감시를 강화한다.
- 이슬점 계산의 입력은 배가스 조성(SOx, H₂O)이다. 조성이 없으면 설계값을 쓰고 화면에 "설계값 사용"을 표시한다.

### 6.2 alarm: 알람

- 규칙은 `alarm_rule` 테이블에 둔다. 코드에 박지 않는다.
- 같은 원인(같은 `rule_id`, 같은 설비)은 1분 안에 3번 넘게 울리면 한 건으로 묶고 횟수를 센다 (참조 §4.2).
- 10분에 10건 넘게 새로 생기면 알람 폭주로 보고, 같은 원인을 묶어서 보여준다.
- 알람 판정 결과는 `alarm_event`에 남기고, 화면에는 "지금"의 한 줄 요약을 보낸다.

### 6.3 economics: 경제성

```
절감액      = (기준선 연료 사용량 − 실제 연료 사용량) × 연료 단가
              + 감축 CO₂ × 탄소 비용
발전 단가   = 발전 비용 ÷ 발전량
투자 회수   = 총 투자비 ÷ (연간 절감액 − 연간 운영비)
```

- 연료 단가는 `fuel_price`에서 날짜로 조회한다. 단가가 바뀌면 행만 더한다.
- 기준선은 `baseline` 서비스가 만든다. 도입 전 12개월 실적을 발전 부하와 외기온으로 보정한 모델이다. 절감액에는 **어느 기준선으로 계산했는지**를 항상 함께 저장한다 (`saving_daily.baseline_id`).
- 유가 시나리오(FR-206)는 가정 계산이다. 사용한 가정(단가, 기간)을 결과와 함께 저장한다 (참조 §7.1, §7.2의 방식).

### 6.4 forecast: 예측

참조 §4.6의 구현을 따른다.

1. 예측 서비스를 화면과 분리한다. 매일 정해진 시각과 매 15분에 예측을 만들어 `forecast`에 넣는다.
2. 기준 모델(지난주 같은 요일 같은 시각)을 함께 돌려 오차를 나란히 보여준다.
3. 중앙값만 긋지 않고 10~90% 범위를 띠로 그린다.

| 예측 대상 | 공변량 (제안) |
| --- | --- |
| 폐열 발생량 | 가동 계획, 정비 일정, 외기온 |
| 부생가스 발생량 | 공정 가동 계획, 정비·정지 일정 |
| 증기·전력 수요 | 달력, 생산 계획, 기온 예보 |

- **정비·정지 계획을 반드시 입력으로 받는다.** 공장 부하는 휴일과 이상 상황에 약하다 [S25][S20].
- 모델 후보는 사전학습 시계열 모델(Chronos-2 등)이다 [S22]. 라이선스(TimesFM 3.0 비상업)와 외부 전송(TimeGPT) 제약을 먼저 확인한다 (참조 §4.6).
- 모델 버전과 배포는 MLflow 같은 도구로 관리할 수 있다 [S28].

### 6.5 optimizer: 배분 최적화

**문제**: 회수한 에너지를 어느 설비에 얼마나 배분해야 외부 연료 비용이 가장 낮은가.

| 구분 | 내용 |
| --- | --- |
| 결정 변수 | 설비별 회수 에너지 배분량, 외부 연료 혼소 비율, 보일러·터빈 부하, 증기 헤더 배분 |
| 목적 | 외부 연료 비용 + 탄소 비용 + 기동·정지 비용의 최소화 (정책에 따라 가중치 조정) |
| 제약 | 증기·전력 수요 충족, 설비 최소·최대 부하, 증기 압력·온도 허용 범위, 부생가스 헤더 압력 하한, 배가스 최저 온도(이슬점 여유), 변동폭(ramp rate), 배출 허용 기준 |
| 방법 | 규칙 기반으로 시작 → 선형·혼합정수 계획법(MILP)으로 확장 |
| 주기 | 실시간 재계산(5~15분)과 일간 계획(24시간 앞) |

이 서비스는 **권고만 만든다.** 설비에 직접 쓰지 않는다.

### 6.6 interlock: 안전 규칙 검증

권고가 화면에 나오기 전에 반드시 통과해야 하는 규칙이다. 규칙은 안전 담당이 정하고, EMS는 이를 **조회**하고 **검증**만 한다. 규칙을 EMS가 마음대로 바꾸지 못한다.

| 규칙 예 | 동작 |
| --- | --- |
| 부생가스 헤더 압력이 하한 이하면 혼소 비율 증가 금지 | 권고 폐기 |
| 배가스 온도가 이슬점 여유 이하면 회수 확대 금지 | 권고 폐기 |
| 한 번에 바꾸는 폭이 변동폭 한도를 넘으면 차단 | 권고 폐기 또는 분할 |
| 설비에 인터락 또는 정지 신호가 있으면 해당 설비 권고 차단 | 권고 폐기 |
| 데이터 품질 불량이거나 통신이 두절되면 자동 적용 중지 | 마지막 안전 상태로 복귀 |

---

## 7. 화면과 프로필

### 7.1 화면 등록표 (참조 §3.4, §1.2 규칙 5)

화면은 등록표와 프로필의 `stage`에 따라 메뉴가 열린다.

| 화면 ID | 이름 | 단계 | 폴더 |
| --- | --- | --- | --- |
| now | 지금 | 0 | `web/screens/now/` |
| alarms | 알람 | 1 | `web/screens/alarms/` |
| fuel-economics | 연료비·경제성 | 2 | `web/screens/fuel-economics/` |
| streams | 설비·스트림 | 3 | `web/screens/streams/` |
| summary | 요약 | 4 | `web/screens/summary/` |
| tomorrow | 내일 | 5 | `web/screens/tomorrow/` |
| optimize | 최적화·권고 | 6 | `web/screens/optimize/` |

공통 컴포넌트(`web/components/`): 회수 에너지 차트, 범위 막대, 에너지 흐름도(스트림 화면), 알람 줄, 빈 상태 안내.

### 7.2 유형 프로필 (변경: 다섯 번째 유형 추가)

```yaml
# profiles/petrochem_power.yaml  (값은 예시, 현장에서 정한다)
type: petrochem_power          # apartment | commercial | factory | cold_storage | petrochem_power
labels:
  site: 발전소
  zone: 스트림
  operator: 중앙제어실 운전원
normalizer:                    # 4단계 정규화 지표의 분모
  name: 발전량
  unit: MWh
breakdown:                     # 3단계 분해 축
  - { id: HRSG,   name: 폐열회수보일러 }
  - { id: OFFGAS, name: 부생가스 라인 }
  - { id: FLARE,  name: 플레어 }
  - { id: ORC,    name: 유기랭킨 }
  - { id: UTL,    name: 유틸리티·외부 연료 }
targets:
  fuel_substitution_pct: TBD   # 연료 대체율 목표. 진단 후 확정
  flare_limit_nm3h: TBD        # 플레어 기준 유량
  flue_t_min_c: TBD            # 배가스 출구 하한 (이슬점 + 여유)
alarm_rules:                   # 1단계 기본 규칙
  - { id: data_gap,      level: 주의, when: "no_data_seconds > 300" }
  - { id: balance_error, level: 주의, when: "balance_err_pct > 5" }
extra_alarm_rules:             # 유형 고유 규칙
  - { id: flare_surge,         level: 경고, when: "flare_nm3h > flare_limit_nm3h" }
  - { id: flue_dewpoint,       level: 위험, when: "flue_t_out_c < flue_t_min_c" }
  - { id: offgas_pressure_low, level: 경고, when: "offgas_press_kpa < offgas_press_min_kpa" }
  - { id: lhv_swing,           level: 경고, when: "abs(lhv_change_5min_pct) > lhv_swing_pct" }
  - { id: recovery_trip,       level: 위험, when: "recovery_state == 'trip'" }
stage: 0                       # 현재 구현 단계. 메뉴가 여기에 맞춰 열린다
```

프로필의 `TBD` 값은 진단 단계에서 채운다. 값이 비어 있으면 해당 알람은 꺼 두고 화면은 "기준값이 설정되면 표시됩니다"라고 안내한다 (참조 §1.2 규칙 4).

---

## 8. 인터페이스와 보안

### 8.1 인터페이스

| 구간 | 방식 | 방향 | 비고 |
| --- | --- | --- | --- |
| DCS·계측기 → collector | OPC UA, Modbus, Historian 접속 | 읽기 | 태그 이름·단위·시각을 표준화 |
| 분석기(GC) → collector | 분석기 제공 인터페이스 | 읽기 | 조성값과 분석 시각 |
| collector → DB | 내부 수집 인터페이스 | 쓰기 | 품질 플래그 부착 |
| 서비스 ↔ DB | 내부 질의 | 읽기·쓰기 | 계산 입력과 결과 |
| 서비스 → web | 실시간 메시지(§3.2), 질의 | 출력 | 역할별 화면 |
| web → 서비스 | 알람 확인, 권고 수락·거부, 설정 | 입력 | 권한 확인 후 |
| 승인된 권고 → DCS | 별도 승인 경로 (approved-writer) | 쓰기 | 6단계 이후, 기본 꺼짐 |
| 외부(ERP, CMMS, 배출권, 가격) | API 또는 파일 연계 | 양방향 | 협의 필요 |

### 8.2 보안 원칙 (제안)

- IT와 OT 망을 분리하고, OT에서 IT 방향의 읽기를 기본으로 한다.
- EMS → DCS 쓰기는 **읽기 경로와 분리한 별도 경로**로만 열고, 평소에는 꺼 둔다.
- 통신은 암호화한다. 계정은 역할별 최소 권한으로 준다 (§8.3).
- 모든 지령과 설정 변경을 `command_audit`, `config_audit`에 남긴다.
- 지령은 한 건마다 승인자, 시각, 이전 값, 새 값을 기록한다.

### 8.3 역할별 권한

| 역할 | 조회 | 알람 확인 | 권고 수락·거부 | 설정 변경 | 쓰기 승인 |
| --- | --- | --- | --- | --- | --- |
| 경영자 | 요약, 경제성 | | | | |
| 에너지 관리자 | 전체 | ○ | | 목표, 기준선, 최적화 정책 | |
| 운전원 | 지금, 알람, 권고 | ○ | ○ | | ○ (6단계 이후) |
| 설비·정비 | 설비·스트림, 알람 | ○ | | | |
| 안전·환경 | 알람, 요약 | ○ | | 인터락 규칙 조회·검토 | |
| 구매 | 내일, 경제성 | | | 연료 단가 | |
| IT·OT | 운영 상태 | | | 계정, 수집 설정 | |

---

## 9. 쓰기 승인 흐름 (6단계 이후)

권고가 설비에 닿기까지의 순서다. 한 단계라도 실패하면 거기서 멈추고 설비는 이전 상태를 유지한다.

1. **optimizer**가 권고를 계산한다.
2. **interlock**이 안전 규칙을 검증한다. 실패하면 권고를 폐기하고 사유를 기록한다.
3. 통과한 권고만 **recommendation**에 저장하고 운전원 화면에 보인다. 근거, 기대 효과, 유효 시간이 함께 표시된다.
4. 운전원이 **수락 또는 거부**한다. 거부 사유를 남긴다.
5. 수락한 경우 **approved-writer**가 현재 DCS 상태를 다시 읽고, 인터락과 변동폭을 한 번 더 검증한다.
6. 검증을 통과하면 쓰기를 실행하고 `command_audit`에 기록한다.
7. 실행 후 일정 시간 동안 결과를 감시한다. 이상이 보이면 **이전 값으로 되돌리고** 운전원에게 알린다.
8. 유효 시간이 지난 권고는 자동으로 무효가 된다.

**장애 시 동작**

| 상황 | 동작 |
| --- | --- |
| 수집 통신 두절 | 마지막 값을 "자료 끊김"으로 표시. 새 권고를 내지 않고 자동 적용을 중지 |
| 데이터 품질 불량 | 해당 입력을 쓰는 권고를 만들지 않음 |
| 승인 경로 오류 | 쓰기 중단, 설비는 이전 상태 유지, 알람 발생 |
| EMS 전체 정지 | DCS의 기본 제어와 SIS는 영향이 없다. 운전원은 기존 방식으로 운전한다 |
| 쓰기 후 이상 | 이전 값으로 복귀, 감사 로그에 `rolled_back` 기록 |

---

## 10. 개발·검증 환경

### 10.1 시뮬레이터 (`simulator/`, 참조 §3.4)

실제 계측이 없거나 쓰기를 시험할 때 쓰는 발생·회수 생성기다 (제안).

| 생성할 신호 | 특징 |
| --- | --- |
| 배가스 온도·유량 | 일 주기 패턴과 부하 변화 |
| 부생가스 유량·조성 | 평상시와 **공정 이상 시 급변**(플레어 급증, 발열량 변동) |
| 회수 설비 | 정지(trip)와 재기동, 열교환기 오염 진행 |
| 데이터 결함 | 결측, 센서 고정, 시계 어긋남 |

시뮬레이터에서 알람, 수지 오차, 권고 폐기, 쓰기 승인 실패 경로를 모두 시험한다. 실제 DCS에 쓰는 시험은 그 이후에, 안전 담당 입회로 진행한다.

### 10.2 환경

| 환경 | 용도 | 쓰기 경로 |
| --- | --- | --- |
| 개발 | 시뮬레이터로 기능 개발 | 모의 대상 |
| 시험 | 실제 이력을 재생해 계산 검증 | 꺼짐 |
| 운영 | 실제 계측 연결 | 6단계 전까지 꺼짐 |

### 10.3 검증 계획

| 대상 | 방법 | 통과 기준 (가정) |
| --- | --- | --- |
| 에너지 수지 | 과거 이력으로 수지를 계산해 수기 계산과 비교 | 오차 5% 이내 |
| 연료 대체율 | 구매·사용 실적으로 역산 | 5% 이내 |
| 알람 | 모의 이상을 주입 | 10초 이내 표시, 시간당 평균 12건 이하 |
| 알람 묶기 | 반복·폭주 시나리오 | 같은 원인이 한 건으로 묶임 |
| 예측 | 과거 구간을 가려 놓고 예측 | 기준 모델보다 나음. 정지·휴일 구간 오차를 따로 보고 |
| 최적화 | 모의 입력으로 제약 위반 여부 확인 | 제약 위반 권고 0건 |
| 인터락 | 위반 권고를 일부러 생성 | 100% 차단 |
| 장애 | 통신 두절, 승인 경로 오류 주입 | 마지막 안전 상태로 복귀 |
| 절감액 | 기준선 대비 계산을 두 번 독립적으로 수행 | 일치 |

---

## 11. 구현 구조

참조 §3.4의 디렉터리 구성을 유지하고 이 발전소에 필요한 것만 바꿨다.

```
ems-dashboard/
├── profiles/                 # 유형 프로필 (apartment, commercial, factory, cold_storage, petrochem_power)
├── collector/                # 수집기. 화면은 외부 API를 직접 부르지 않는다
│   ├── historian/            #   0단계: 기존 Historian·DCS 이력
│   ├── field/                #   0~3단계: OPC UA, Modbus
│   ├── analyzer/             #   0~1단계: 부생가스 조성
│   ├── weather/              #   5단계
│   ├── market/               #   2단계: 유가·연료 가격
│   └── approved-writer/      #   6단계 이후: 승인된 쓰기 (기본 꺼짐)
├── db/
│   ├── migrations/           # 000_base, 010_alarm, 020_fuel, 030_balance, 040_forecast, 050_recommendation, 060_audit
│   └── views/                # 연속 집계와 공통 지표 뷰
├── services/
│   ├── balance/              # 0~3단계: 에너지 수지, 회수율, 연료 대체율
│   ├── alarm/                # 1단계
│   ├── economics/            # 2단계: 연료비, 절감액, 발전 단가, 시나리오
│   ├── baseline/             # 2~4단계: 기준선과 정규화
│   ├── forecast/             # 5단계
│   ├── optimizer/            # 6단계
│   ├── interlock/            # 6단계: 권고 검증
│   └── whatif/               # 7단계: 가정 계산
├── web/
│   ├── screens/
│   │   ├── now/              #   0단계
│   │   ├── alarms/           #   1단계
│   │   ├── fuel-economics/   #   2단계
│   │   ├── streams/          #   3단계
│   │   ├── summary/          #   4단계
│   │   ├── tomorrow/         #   5단계
│   │   └── optimize/         #   6단계
│   └── components/
└── simulator/                # 발생·회수 생성기
```

**구현 순서 (제안)**

1. 0단계를 Grafana로 하루 이틀 안에 띄워 자료 흐름을 확인한다 (참조 §5).
2. 같은 시기에 **기준선 자료 수집**을 시작한다 (기능정의서 §5).
3. 1단계부터 자체 웹으로 옮기고, 단계마다 끝 조건을 통과한 뒤 다음으로 간다.
4. 쓰기 경로(`approved-writer`)는 6단계의 안전 검토가 끝날 때까지 만들지 않는다.

---

## 12. 미확정 사항과 위험

| 항목 | 내용 | 대응 |
| --- | --- | --- |
| 계측 공백 | 폐열·부생가스 주요 지점의 유량계, 온도계, 조성 분석기가 없을 수 있다 | 0단계에서 공백 목록을 만들고, 우선 추가할 계측점을 정한다 |
| 조성 분석 주기 | 분석 주기가 길면 발열량이 급변할 때 계산이 틀린다 | `lhv_swing` 알람, 추정 표시 |
| 기존 DCS·Historian 접속 | 모델과 통신 방식에 따라 수집 방식이 달라진다 | IT·OT 담당과 먼저 협의 |
| 인터락 정의 | EMS가 쓸 수 있는 범위를 안전 담당과 합의해야 한다 | 6단계 전에 문서로 확정 |
| 기준선 | 과거 12개월 자료가 없거나 가동이 크게 달라졌다 | 부분 기간으로 시작하고, 보정 한계를 화면에 표시 |
| 예측 정확도 | 정지·휴일·공정 이상에 약하다 | 정비·정지 계획 입력, 예측만으로 결정하지 않음 |
| 라이선스·데이터 반출 | 일부 예측 모델의 사용 제약 | 도입 전 확인 |
| 규제 수치 | 배출권, 공시 관련 수치는 검색 발췌 수준이다 | 보고서·제안서에 쓰기 전에 원문 확인 |
| 수치 전반 | 이 문서의 모든 수치는 가정값이다 | 진단 단계에서 실측으로 교체 |

---

## 13. 참조한 출처

본문의 `[S08]`, `[S09]`, `[S20]`, `[S22]`, `[S25]`, `[S28]` 같은 표기는 구현가이드의 "9. 조사 기록" 번호다. 이 문서는 해당 출처를 다시 열어 확인하지 않았고, 구현가이드의 내용을 그대로 옮겼다.
