---
title: GPU 파드별 메모리 사용량 주간 보고서
cadence: weekly
params:
  - name: namespace_filter
    label: 네임스페이스
    type: select
    source: {category: kube_pod, tag: namespace}
    default: all
---

# GPU 파드별 메모리 사용량 주간 보고서

이 보고서는 주간 GPU 파드의 메모리 사용 현황을 분석하여 리소스 최적화와 용량 계획에 활용할 수 있는 인사이트를 제공합니다.

## 주요 메트릭 요약

```chart-counter-summary
data-type: MXQL
cards:
  - title: 평균 파드 메모리 사용량
    query: |
      CATEGORY kube_pod
      TAGLOAD
      TIME-RANGE {stime: {{ week-start }}, etime: {{ week-end }}}
      SELECT [mem_usage]
    value-field: mem_usage
    transform:
      - {type: filterParam, field: namespace, param: namespace_filter}
      - {type: reduce, fields: {mem_usage: avg}}
      - {type: scale, field: mem_usage, factor: 0.00000095367}
    format: "#,##0.0"
    suffix: " MiB"
  
  - title: 최대 파드 메모리 사용량 (주간 평균 기준)
    query: |
      CATEGORY kube_pod
      TAGLOAD
      TIME-RANGE {stime: {{ week-start }}, etime: {{ week-end }}}
      SELECT [podName, mem_usage]
    value-field: mem_usage
    transform:
      - {type: filterParam, field: namespace, param: namespace_filter}
      - {type: groupBy, keys: [podName], fields: {mem_usage: avg}}
      - {type: reduce, fields: {mem_usage: max}}
      - {type: scale, field: mem_usage, factor: 0.00000095367}
    format: "#,##0.0"
    suffix: " MiB"
  
  - title: 전체 파드 수
    query: |
      CATEGORY kube_pod
      TAGLOAD
      TIME-RANGE {stime: {{ week-start }}, etime: {{ week-end }}}
      SELECT [podName]
    value-field: podName
    transform:
      - {type: filterParam, field: namespace, param: namespace_filter}
      - {type: groupBy, keys: [podName], fields: {podName: count}}
      - {type: reduce, fields: {podName: count}}
    format: "#,##0"
  
  - title: 메모리 Limit 대비 평균 사용률
    query: |
      CATEGORY kube_pod
      TAGLOAD
      TIME-RANGE {stime: {{ week-start }}, etime: {{ week-end }}}
      SELECT [memory_per_limit]
    value-field: memory_per_limit
    transform:
      - {type: filterParam, field: namespace, param: namespace_filter}
      - {type: reduce, fields: {memory_per_limit: avg}}
    format: "#,##0.0"
    suffix: "%"
```

## 메모리 사용량 추이

```chart-line
data-type: MXQL
title: 파드별 메모리 사용량 주간 추이
query: |
  CATEGORY kube_pod
  TAGLOAD
  TIME-RANGE {stime: {{ week-start }}, etime: {{ week-end }}}
  SELECT [time, mem_usage, podName]
x-axis: {field: time, format: epoch-ms}
y-axis:
  - {id: left, title: "메모리 사용량", unit: "MiB"}
series:
  - {name: "메모리 사용량", field: mem_usage, y-axis: left, type: areaspline}
series-by: podName
transform:
  - {type: filterParam, field: namespace, param: namespace_filter}
  - {type: groupByTime, interval: 1d, agg: avg, by: podName}
  - {type: scale, field: mem_usage, factor: 0.00000095367}
```

## 메모리 Limit 대비 사용률 추이

```chart-line
data-type: MXQL
title: 메모리 Limit 대비 사용률 주간 추이
query: |
  CATEGORY kube_pod
  TAGLOAD
  TIME-RANGE {stime: {{ week-start }}, etime: {{ week-end }}}
  SELECT [time, memory_per_limit]
x-axis: {field: time, format: epoch-ms}
y-axis:
  - {id: left, title: "사용률", unit: "%"}
series:
  - {name: "Limit 대비 사용률", field: memory_per_limit, y-axis: left, type: areaspline}
transform:
  - {type: filterParam, field: namespace, param: namespace_filter}
  - {type: groupByTime, interval: 1d, agg: avg}
```

## 메모리 사용률 분포

```chart-donut
data-type: MXQL
title: 파드별 메모리 Limit 대비 사용률 분포
query: |
  CATEGORY kube_pod
  TAGLOAD
  TIME-RANGE {stime: {{ week-start }}, etime: {{ week-end }}}
  SELECT [memory_per_limit, podName]
value-field: memory_per_limit
transform:
  - {type: filterParam, field: namespace, param: namespace_filter}
  - {type: groupBy, keys: [podName], fields: {memory_per_limit: avg}}
  - {type: binValues, field: memory_per_limit, thresholds: [30, 50, 80], labels: ["30% 미만", "30~50%", "50~80%", "80% 이상"]}
```

## 파드별 메모리 상세 현황

```table
data-type: MXQL
title: 파드별 평균 메모리 사용 상세
query: |
  CATEGORY kube_pod
  TAGLOAD
  TIME-RANGE {stime: {{ week-start }}, etime: {{ week-end }}}
  SELECT [podName, namespace, mem_usage, memory_limit, memory_per_limit, mem_working_set]
columns:
  - {field: podName, title: "파드명", align: left}
  - {field: namespace, title: "네임스페이스", align: left}
  - {field: mem_usage, title: "평균 사용량 (MiB)", align: right, format: "#,##0.0"}
  - {field: memory_limit, title: "메모리 Limit (MiB)", align: right, format: "#,##0.0"}
  - {field: memory_per_limit, title: "Limit 대비 사용률 (%)", align: right, format: "#,##0.0"}
  - {field: mem_working_set, title: "Working Set (MiB)", align: right, format: "#,##0.0"}
transform:
  - {type: filterParam, field: namespace, param: namespace_filter}
  - {type: groupBy, keys: [podName, namespace], fields: {mem_usage: avg, memory_limit: avg, memory_per_limit: avg, mem_working_set: avg}}
  - {type: scale, field: mem_usage, factor: 0.00000095367}
  - {type: scale, field: memory_limit, factor: 0.00000095367}
  - {type: scale, field: mem_working_set, factor: 0.00000095367}
  - {type: sortBy, by: mem_usage, desc: true}
limit: 20
zebra: true
```

## GPU 메모리 사용 현황

```chart-line
data-type: MXQL
title: GPU Frame Buffer 사용량 주간 추이
query: |
  >> avg(DCGM_FI_DEV_FB_USED) by (uuid)
x-axis: {field: time, format: epoch-ms}
y-axis:
  - {id: left, title: "GPU 메모리 사용량", unit: "MB"}
series:
  - {name: "FB Used", field: value, y-axis: left, type: areaspline}
series-by: uuid
transform:
  - {type: groupByTime, interval: 1d, agg: avg}
```
