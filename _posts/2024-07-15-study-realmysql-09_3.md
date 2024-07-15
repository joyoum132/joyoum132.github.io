---
title: 09. 옵티마이저와 힌트 - 고급최적화 2
description: 옵티마이저의 조인 최적화 알고리즘
date: 2024-07-15 +0900
categories: [Study, Real MySQL 8.0]
tags: [Team Study, RealMySQL, optimizer]
---

## <b>Exhausive 검색 알고리즘</b>
- MySQL 5.0 이하의 버전에서 사용하던 알고리즘
- from 절의 테이블에 대한 조합으로 실행계획을 분석하고 가장 최적 조합 1개를 찾음
- n개의 테이블에 대해 n! 번의 방법을 검색 (시간 소모가 큼)

## <b>Greedy 검색 알고리즘</b>
- optimizer_search_depth 로 지정한 갯수 만큼의 테이블 조합을 생성한 뒤에 
- 각 조합별 최적을 부분 실행 계획으로 저장하고
- 조합별 최적에 선택된 테이블을 제거한 나머지 테이블끼리 다시 조합을 생성하고 부분 실행 계획에 추가
- 모든 테이블을 다 사용할 때 까지 조합의 최적을 부분 실행 계획에 추가하고, 최종 부분 실행 계획을 실행계획으로 사용

### [ optimizer_search_depth ]
- 조인 테이블의 조합 개수를 결정하는 숫자
- default 는 62이고 0~62 정숫값 사용 가능 <b>(권장 4~5) </b>
- 값이 0
  - greedy  검색을 위한 최적의 조인 검색 테이블 개수를 MySQL 옵티마이저가 결정
  - 필요에 따라 Greedy, Exhausive 알고리즘을 동시에 사용
- 조인에 필요한 테이블 개수 > optimizer_search_depth
  - optimizer_search_depth 만큼은 Exhausive 알고리즘을 사용하고, 나머지만 Greedy 방식 사용
- 조인에 필요한 테이블 개수 < optimizer_search_depth
  - Exhausive 알고리즘 사용

### [ optimizer_prune_level ]
- Heuristic 검색 제어 (1: true, 0: false)
- 현재 계산하는 비용이 이전의 계산 값보다 크면 중단, 경험 기반 최적화 설정
- optimizer_search_depth 를 적절한 정수로 설정하더라도, 이 옵션이 비활성화 되어있다면 실행계획에 오랜 시간이 소요될 수 있음
