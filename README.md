# report-template

WhaTap Instant Report 템플릿 저장소.

보고서 템플릿은 **마크다운 파일 1개**로 정의한다 — YAML front matter(제목·주기·파라미터) + 본문(마크다운) + 위젯 펜스(쿼리·시각화 선언). 대시보드가 "현재 상태"라면 보고서는 "기간과 파라미터를 받아 문서를 만드는 함수"다.

## 구조

```
templates/   보고서 템플릿 (.md)
SPEC.md      템플릿 문법 스펙 (front matter · 위젯 펜스 · transform · 시간변수)
```

## 템플릿 예시

````markdown
---
title: GPU 워크로드별 사용량 보고서
cadence: monthly
params:
  - name: gpu_group
    label: GPU 그룹
    type: select
    source: {category: kube_pod, tag: gpu_group}
    default: all
---

# GPU 워크로드별 사용량 보고서

```chart-counter-summary
cards:
  - title: 평균 GPU 활용률
    query: |
      >> avg(DCGM_FI_DEV_GPU_UTIL)
    transform:
      - {type: reduce, fields: {value: avg}}
    suffix: "%"
```
````

## 규칙

- **pcode는 템플릿에 넣지 않는다.** 실행 시 실행자의 프로젝트/권한으로 쿼리가 돈다 (제작자 권한 아님).
- 시크릿(세션·토큰) 금지 — 인증은 실행 환경이 담당한다.
- 위젯 펜스·transform·시간변수 전체 문법은 [SPEC.md](SPEC.md) 참고.

## 제공 템플릿

| 파일 | 주기 | 내용 |
|---|---|---|
| `gpu-workload-usage-monthly.md` | 월간 | GPU 그룹(파드 라벨)별 활용률·메모리·전력·온도 |
| `gpu-usage-by-node-monthly.md` | 월간 | 노드 필터 · 일별 추이 · 사용률 구간 분포 |
| `gpu-pod-memory-weekly.md` | 주간 | 네임스페이스 필터 · 파드별 메모리 Top |
