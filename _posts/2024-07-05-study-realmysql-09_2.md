---
title: 09. 옵티마이저와 힌트 - 스위치 옵션
description: 옵티마이저의 스위치 옵션의 종류
date: 2024-07-02 +0900
categories: [Study, Real MySQL 8.0]
tags: [Team Study, RealMySQL, optimizer]
---

## <b> MRR 과 배치 키 액세스(MRR and Batch Key Access) </b>
- MRR 아닌 Nest Loop Join 방식
  - 드라이빙 테이블의 레코드를 읽고, 드리븐 테이블에서 일치하는 레코드 찾기
  - 조인처리는 MySQL 엔진이 처리하지만 데이터 조회는 스토리지 엔진에서 처리하는데, 건별로 레코드를 조회하는 이 조인 방식을 사용하게 되면 스토리지 엔진에서 최적화 하기 어려움
- MRR
  - Multi Range Read
  - 드라이빙 테이블의 레코드를 읽고 <b>조인 버퍼에 버퍼링</b> (즉, 즉시 조인하지 않고 조인 대상을 버퍼링)
  - 조인 버퍼가 가득 차면 스토리지 엔진에 한번에 요청
    - 조인 버퍼에는 key 로 정렬해서 저장
    - 디스크의 읽기 작업 최소화
- BKA
  - Batch Key Access
  - MRR 옵션을 활성화 할 때, 사용할 수 있는 알고리즘
  - default : "OFF"
    - 쿼리 특성마다 성능 차이가 있는 편임
    - 부가적인 정렬로 인한 성능 지연 가능성 존재

