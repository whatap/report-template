# Instant Report 템플릿 스펙 v0

보고서 = **마크다운 파일 1장**. 문서 구조는 표준 마크다운(GFM), 데이터 위젯은 펜스 코드블록,
메타/파라미터는 YAML front matter. 렌더러(`index.html`)는 범용 — 템플릿에만 보고서 지식이 담긴다.

**포맷 계보 (호환성 선언):** WhaTap SmartReport의 SmartMD 위젯 블록 컨벤션을 따르는 **확장(superset)**이다.
문서 골격(front matter + 펜스 + `{{ }}` 변수)은 MyST/Quarto/R Markdown 계열의 공용 관행,
축·시리즈 구조는 Highcharts/ECharts 옵션 컨벤션, transform 연산 이름은 Grafana transformations를 따른다.
우리 고유 확장은 두 가지: **params(렌더 후 인터랙션)** 와 **transform(클라이언트 변환 파이프라인)**.
쿼리는 MXQL 직접 — 데이터 핸들러 allowlist 없음. 필드 정합성은 tagmeta 카탈로그
(catalog.json, 290 카테고리/4,638 필드)로 검증.

## 1. Front matter

```yaml
---
title: GPU 월간 사용량 보고서
cadence: monthly            # daily | weekly | monthly | custom → 시간변수 세트 결정
pcode: 1234  # 선택 — 실행 시 실행자의 프로젝트가 우선한다
params:                     # 렌더 후 사용자가 바꿀 수 있는 셀렉터 (선택)
  - name: node
    label: 노드 선택
    type: multi-select      # multi-select | select | text
    source:                 # 선택지를 라이브 조회로 채움
      category: kube_node
      tag: onodeName
    default: all
---
```

- `cadence` → 시간변수 자동 바인딩: monthly는 `{{ month-start }}`/`{{ month-end }}`,
  daily는 `{{ target-date-start }}`/`{{ target-date-end }}`, weekly는 `{{ week-start }}`/`{{ week-end }}`.
  모든 값은 epoch ms. 렌더러 상단 기간 선택기가 이 변수를 채운다.
- `params[].source` — `CATEGORY <category> TAGLOAD GROUP {pk:[<tag>]}` 로 태그값을 열거해 셀렉터를 만든다.
- 템플릿 본문에서 `{{ param:node }}` 로 참조. `all`이면 필터 미주입.

## 2. 위젯 펜스 (v0: 4종)

펜스 이름과 키 구조는 SmartMD의 위젯 블록(`chart-line`, `chart-donut`, `chart-counter-summary`,
`table`)을 따른다. 축·시리즈 구조는 Highcharts/ECharts 옵션 컨벤션(`x-axis`/`y-axis[]`/`series[]`).

공통 키:

| 키 | 필수 | 설명 |
|---|---|---|
| `data-type` | ✓ | v0은 `MXQL`만. (`API`는 향후 확장 여지) |
| `query` | ✓ | MXQL 본문 (블록 스칼라 `\|`). `{{ }}` 변수 치환 후 실행 |
| `title` | – | 위젯 제목 |
| `transform` | – | 클라이언트 변환 파이프라인 (아래 §3) |
| `height` | – | px, 기본 280 |

### chart-line — 시계열 라인/에어리어

```chart-line
data-type: MXQL
title: 일별 사용량 추이
query: |
  CATEGORY gpu_usage
  TAGLOAD
  TIME-RANGE {stime: {{ month-start }}, etime: {{ month-end }}}
  SELECT [time, host, usage]
x-axis:
  field: time
  format: epoch-ms
y-axis:
  - id: left
    title: 사용률
    unit: "%"
series:
  - name: 사용률
    field: usage
    y-axis: left
    type: areaspline      # line(기본) | areaspline
series-by: host           # [확장] 이 태그값별로 시리즈 동적 분리 (SmartMD에 없음)
transform:
  - type: groupByTime
    interval: 1d
    agg: avg
```

- `series[]`는 정적 시리즈(컬럼 → 시리즈). `series-by`는 우리 확장 — 태그 필드의 값별로
  시리즈를 동적 생성한다(파드별/노드별 분리). 둘 다 있으면 `series-by` 우선.

### chart-donut — 구성비/구간 분포

```chart-donut
data-type: MXQL
title: 리소스 사용별 대상 수량
query: |
  ...
value-field: usage
transform:
  - type: binValues        # 값 구간별 개체 수 집계 (K201 도넛)
    field: usage
    thresholds: [30, 50, 80]
    labels: ["30% 미만", "30~50%", "50~80%", "80% 이상"]
colors: ["#8BC34A", "#64B5F6", "#FFF176", "#EF9A9A"]
```

### chart-counter-summary — KPI 카드 그리드

```chart-counter-summary
data-type: MXQL
cards:
  - title: 물리 GPU
    query: |
      ...
    value-field: value
    format: "#,##0"
  - title: 평균 GPU 사용률
    query: |
      ...
    value-field: value
    format: "#,##0.0"
    suffix: "%"
```

### table — 표

```table
data-type: MXQL
title: 노드별 상세
query: |
  ...
columns:
  - { field: onodeName, title: "노드", align: left }
  - { field: usage, title: "사용률(%)", align: right, format: "#,##0.0" }
limit: 50
zebra: true
```

## 3. transform 파이프라인

쿼리 결과(row 배열)에 순서대로 적용. **비율·구간·정렬은 서버가 아니라 여기서 한다**
(이유: ① 표현 로직은 템플릿의 일부다 ② dev yard 벡터 중첩 연산 회귀(OPMT-108) 우회).
연산 이름은 Grafana transformations 컨벤션에 정렬 (`groupBy`, `binValues`, `sortBy`, `limit`).

| type | 파라미터 | 동작 | Grafana 대응 |
|---|---|---|---|
| `groupByTime` | `interval`(1h/1d), `agg`(avg/sum/max/min), `by`(선택) | 시간 버킷 집계 | group by + interval |
| `binValues` | `field`, `thresholds[]`, `labels[]` | 값 구간별 count (도넛/분포용) | (유사: histogram) |
| `ratio` | `numer`, `denom[]`, `as`, `scale`(기본 100) | `as = numer / sum(denom) * scale` 파생 컬럼 | calculateField |
| `groupBy` | `keys[]`, `fields{name: fn}` | 그룹 집계 (fn: avg/sum/max/min/count) | groupBy |
| `scale` | `field`, `factor` | 값 배율 (예: 0~1 비율 → %) | calculateField |
| `sortBy` | `by`, `desc` | 정렬 | sortBy |
| `limit` | `n` | 상위 n행 | limit |
| `filterParam` | `field`, `param` | params 셀렉터 값으로 행 필터 (all이면 통과) | (우리 확장) |

## 4. 변수 화이트리스트

`{{ pcode }}` · `{{ month-start }}` `{{ month-end }}` · `{{ week-start }}` `{{ week-end }}`
· `{{ target-date-start }}` `{{ target-date-end }}` · `{{ param:<name> }}`
목록 밖 변수는 렌더 시 경고 + 원문 유지.

## 5. 데이터 모드

- **mock** — 시드 PRNG(LCG, seed=위젯 title)로 결정적 합성 데이터. `file://`로도 동작.
- **live** — WhaTap 메트릭 API(`/yard/api/flush`) POST로 실데이터 조회. 인증은 실행 환경의
  세션이 담당하므로 **템플릿·렌더러·배포본에는 시크릿이 존재하지 않는다.**

## 5.5 GPU 메트릭 선택 (MIG 주의)

- 활용률: `DCGM_FI_DEV_WEIGHTED_GPU_UTIL` (0~1 비율 — `{type: scale, field: value, factor: 100}`로 % 변환).
  `DCGM_FI_DEV_GPU_UTIL`은 **MIG 활성 GPU에서 미보고** — MIG 장비가 보고서에서 조용히 누락되므로 금지.
- MIG 인스턴스별 활용률: `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (인스턴스별 시리즈).
- 메모리: `DCGM_FI_DEV_FB_USED`/`FB_TOTAL`은 MIG 인스턴스별로 정상 분할 수집.
- 온도/전력(`GPU_TEMP`/`POWER_USAGE`)은 물리 GPU 단위만 존재 — MIG 인스턴스로 그룹핑 금지(중복 표시).

## 6. 검증 규칙 (생성기·수동 작성 공용)

1. 펜스의 MXQL `CATEGORY`/`SELECT` 필드가 catalog.json에 존재해야 한다 (tag 여부 포함).
2. front matter `cadence`와 본문 시간변수 세트가 일치해야 한다.
3. 위젯 펜스는 §2의 4종만. 미지원 펜스는 코드블록으로 fallback 렌더.
4. MXQL에 `A / (A + B)` 형태의 벡터 중첩 이항연산 금지 → transform `ratio`로 대체 (OPMT-108).
