---
title: 6월 GPU 월간 사용량 보고서
cadence: monthly
params:
  - name: node
    label: 노드
    type: select
    source: {category: kube_node, tag: oname}
    default: all
---

# 6월 GPU 월간 사용량 보고서

## GPU 사용 현황 요약

```chart-counter-summary
data-type: MXQL
cards:
  - title: 평균 GPU 사용률
    query: |
      >> avg by (Hostname) (DCGM_FI_DEV_WEIGHTED_GPU_UTIL)
    value-field: value
    transform:
      - {type: filterParam, field: Hostname, param: node}
      - {type: reduce, fields: {value: avg}}
    format: "#,##0.0"
    suffix: "%"
  - title: 평균 GPU 메모리 사용률
    query: |
      >> avg by (Hostname) (DCGM_FI_DEV_FB_USED_PERCENT)
    value-field: value
    transform:
      - {type: filterParam, field: Hostname, param: node}
      - {type: reduce, fields: {value: avg}}
    format: "#,##0.0"
    suffix: "%"
  - title: 평균 GPU 온도
    query: |
      >> avg by (Hostname) (DCGM_FI_DEV_GPU_TEMP)
    value-field: value
    transform:
      - {type: filterParam, field: Hostname, param: node}
      - {type: reduce, fields: {value: avg}}
    format: "#,##0.0"
    suffix: "°C"
  - title: 평균 전력 소비
    query: |
      >> avg by (Hostname) (DCGM_FI_DEV_POWER_USAGE)
    value-field: value
    transform:
      - {type: filterParam, field: Hostname, param: node}
      - {type: reduce, fields: {value: avg}}
    format: "#,##0.0"
    suffix: "W"
```

## 일별 GPU 사용률 추이

```chart-line
data-type: MXQL
title: 일별 GPU 사용률 추이
query: |
  >> avg by (Hostname) (DCGM_FI_DEV_WEIGHTED_GPU_UTIL)
x-axis: {field: time, format: epoch-ms}
y-axis:
  - {id: left, title: GPU 사용률, unit: "%"}
series:
  - {name: GPU 사용률, field: value, y-axis: left, type: areaspline}
series-by: Hostname
transform:
  - {type: scale, field: value, factor: 100}
  - {type: filterParam, field: Hostname, param: node}
  - {type: groupByTime, interval: 1d, agg: avg}
```

## 일별 GPU 메모리 사용 추이

```chart-line
data-type: MXQL
title: 일별 GPU 메모리 사용 추이
query: |
  >> avg by (Hostname) (DCGM_FI_DEV_FB_USED)
x-axis: {field: time, format: epoch-ms}
y-axis:
  - {id: left, title: 메모리 사용량, unit: "MiB"}
series:
  - {name: 메모리 사용량, field: value, y-axis: left, type: areaspline}
series-by: Hostname
transform:
  - {type: filterParam, field: Hostname, param: node}
  - {type: groupByTime, interval: 1d, agg: avg}
```

## GPU 사용률 구간 분포

```chart-donut
data-type: MXQL
title: GPU 사용률 구간 분포
query: |
  >> avg by (Hostname) (DCGM_FI_DEV_WEIGHTED_GPU_UTIL)
value-field: value
transform:
  - {type: scale, field: value, factor: 100}
  - {type: filterParam, field: Hostname, param: node}
  - {type: binValues, field: value, thresholds: [30, 50, 80], labels: ["30% 미만", "30~50%", "50~80%", "80% 이상"]}
```

## 노드별 평균 GPU 사용률

```chart-bar
data-type: MXQL
title: 노드별 평균 GPU 사용률
query: |
  >> avg by (Hostname) (DCGM_FI_DEV_WEIGHTED_GPU_UTIL)
label-field: Hostname
value-field: value
unit: "%"
horizontal: true
transform:
  - {type: scale, field: value, factor: 100}
  - {type: filterParam, field: Hostname, param: node}
  - {type: groupBy, keys: [Hostname], fields: {value: avg}}
  - {type: sortBy, by: value, desc: true}
  - {type: limit, n: 15}
```

## GPU 상세 지표

```table
data-type: MXQL
title: 노드별 GPU 상세 지표
query: |
  >> avg by (Hostname, modelName) (DCGM_FI_DEV_WEIGHTED_GPU_UTIL)
columns:
  - {field: Hostname, title: "노드", align: left}
  - {field: modelName, title: "GPU 모델", align: left}
  - {field: value, title: "평균 사용률 (%)", align: right, format: "#,##0.0"}
limit: 20
zebra: true
```
