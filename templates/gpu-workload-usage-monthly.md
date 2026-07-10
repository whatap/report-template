---
title: GPU 워크로드별 사용량 보고서 (6월)
cadence: monthly
params:
  - name: gpu_group
    label: GPU 그룹
    type: select
    source: {category: kube_pod, tag: gpu_group}
    default: all
---

# GPU 워크로드별 사용량 보고서

6월 한 달간 GPU 활용률, 메모리, 전력, 온도 등 주요 지표를 GPU 그룹(파드 라벨 기준) 단위로 분석한 보고서입니다.

## 주요 지표 요약

```chart-counter-summary
data-type: MXQL
cards:
  - title: 평균 GPU 활용률
    query: |
      >> avg(DCGM_FI_DEV_WEIGHTED_GPU_UTIL)
    value-field: value
    transform:
      - {type: reduce, fields: {value: avg}}
    format: "#,##0.0"
    suffix: "%"
  - title: 평균 GPU 메모리 사용률
    query: |
      >> avg(DCGM_FI_DEV_FB_USED_PERCENT)
    value-field: value
    transform:
      - {type: reduce, fields: {value: avg}}
    format: "#,##0.0"
    suffix: "%"
  - title: 평균 온도
    query: |
      >> avg(DCGM_FI_DEV_GPU_TEMP)
    value-field: value
    transform:
      - {type: reduce, fields: {value: avg}}
    format: "#,##0.0"
    suffix: "°C"
  - title: 총 GPU 전력 소비
    query: |
      >> sum(DCGM_FI_DEV_POWER_USAGE)
    value-field: value
    transform:
      - {type: reduce, fields: {value: avg}}
    format: "#,##0"
    suffix: "W"
```

## GPU 활용률 추이

```chart-line
data-type: MXQL
title: GPU 그룹별 GPU 활용률 추이
query: |
  >> avg by (gpu_group) (DCGM_FI_DEV_WEIGHTED_GPU_UTIL{gpu_group!=""})
x-axis: {field: time, format: epoch-ms}
y-axis:
  - {id: left, title: GPU 활용률, unit: "%"}
series:
  - {name: GPU 활용률, field: value, y-axis: left}
series-by: gpu_group
transform:
  - {type: scale, field: value, factor: 100}
  - {type: filterParam, field: gpu_group, param: gpu_group}
  - {type: groupByTime, interval: 1d, agg: avg, by: gpu_group}
```

## GPU 메모리 사용률 추이

```chart-line
data-type: MXQL
title: GPU 그룹별 GPU 메모리 사용률 추이
query: |
  >> avg by (gpu_group) (DCGM_FI_DEV_FB_USED_PERCENT{gpu_group!=""})
x-axis: {field: time, format: epoch-ms}
y-axis:
  - {id: left, title: 메모리 사용률, unit: "%"}
series:
  - {name: 메모리 사용률, field: value, y-axis: left, type: areaspline}
series-by: gpu_group
transform:
  - {type: filterParam, field: gpu_group, param: gpu_group}
  - {type: groupByTime, interval: 1d, agg: avg, by: gpu_group}
```

## GPU 그룹별 GPU 메모리 사용량

```chart-bar
data-type: MXQL
title: GPU 그룹별 평균 GPU 메모리 사용량
query: |
  CATEGORY kube_pod
  TAGLOAD
  TIME-RANGE {stime: {{ month-start }}, etime: {{ month-end }}}
  SELECT [gpu_group, mem_usage]
label-field: gpu_group
value-field: mem_usage
unit: "MB"
horizontal: true
transform:
  - {type: filterParam, field: gpu_group, param: gpu_group}
  - {type: groupBy, keys: [gpu_group], fields: {mem_usage: avg}}
  - {type: sortBy, by: mem_usage, desc: true}
  - {type: limit, n: 15}
```

## SM(Streaming Multiprocessor) 활용 현황

```chart-line
data-type: MXQL
title: SM 활성도 및 점유율
query: |
  >> DCGM_FI_PROF_SM_ACTIVE
x-axis: {field: time, format: epoch-ms}
y-axis:
  - {id: left, title: SM 활성도, unit: "%"}
series:
  - {name: SM Active, field: value, y-axis: left, type: line}
transform:
  - {type: groupByTime, interval: 1d, agg: avg}
```

## GPU 연산 엔진 활용 분포

```chart-donut
data-type: MXQL
title: Tensor Core 활용률 분포
query: |
  >> DCGM_FI_PROF_PIPE_TENSOR_ACTIVE
value-field: value
transform:
  - {type: binValues, field: value, thresholds: [30,50,80], labels: ["30% 미만","30~50%","50~80%","80% 이상"]}
```

## GPU 그룹별 상세 사용량

```table
data-type: MXQL
title: GPU 그룹별 GPU 리소스 사용 현황
query: |
  CATEGORY kube_pod
  TAGLOAD
  TIME-RANGE {stime: {{ month-start }}, etime: {{ month-end }}}
  SELECT [gpu_group, cpu_total, mem_usage, gpu_request_counts]
columns:
  - {field: gpu_group, title: "GPU 그룹", align: left}
  - {field: cpu_total, title: "평균 CPU 사용률", align: right, format: "#,##0.0"}
  - {field: mem_usage, title: "평균 메모리 사용량 (MB)", align: right, format: "#,##0"}
  - {field: gpu_request_counts, title: "GPU 요청 수", align: right, format: "#,##0"}
transform:
  - {type: filterParam, field: gpu_group, param: gpu_group}
  - {type: groupBy, keys: [gpu_group], fields: {cpu_total: avg, mem_usage: avg, gpu_request_counts: sum}}
  - {type: sortBy, by: cpu_total, desc: true}
limit: 20
zebra: true
```

## GPU 전력 소비 분석

```chart-line
data-type: MXQL
title: 일별 GPU 전력 사용량
query: |
  >> sum(DCGM_FI_DEV_POWER_USAGE)
x-axis: {field: time, format: epoch-ms}
y-axis:
  - {id: left, title: 전력 사용량, unit: "W"}
series:
  - {name: 전력 소비, field: value, y-axis: left, type: areaspline}
transform:
  - {type: groupByTime, interval: 1d, agg: avg}
```
