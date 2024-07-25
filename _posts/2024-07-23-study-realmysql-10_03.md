---
title: 10. 실행계획 - 실행 계획 분석 2
description: 실행 계획 테이블의 칼럼 이해하기 (type)
date: 2024-07-23 +0900
categories: [Study, Real MySQL 8.0]
tags: [Team Study, RealMySQL, explain]
---

## 실행 계획 분석
### [ type 칼럼 ]
- 조인 타입 이라고 명시
- 실질적으로는 <u> 테이블의 접근 방법 </u>
- 단위 select 마다 하나의 접근 방법을 갖고있으며, 하나의 인덱스만을 사용 (index_merge 제외) 
- 성능이 빠른 순서이고 ALL 을 제외한 나머지는 인덱스를 이용한 접근 방식을 의미함

#### 1. system
- 테이블에 레코드가 없거나, 하나만 있을 때
- MyISAM, MEMORY 스토리지 엔진의 접근 방법이고, InnoDB 스토리지 엔진에서는 ALL 로 표시됨

#### 2. const
- 반드시 1건을 반환하는 쿼리
- PK 나 유니크 키 컬럼을 조건으로 사용
- 유니크 인덱스 스캔이라고도 함
- 인덱스 또는 프라이머리키가 복합키로 구성되어있고, 조건에는 그 중 하나의 컬럼만을 사용한다면 const 가 아닌 ref 로 표시
- 옵티마이저는 const 로 조회되는 값 자체를 상수화해서 조건으로 사용

#### 3. eq_ref
- 조인할 때 드리븐 테이블의 pk 또는 유니크 컬럼을 조건을 사용하는 경우
- 조인 테이블의 결과가 반드시 1개의 레코드를 반환

#### 4. ref
- equal 조건을 검색해서 인덱스를 사용하는 경우
  - type 칼럼이 아닌 ref 칼럼에 const 가 출력되는 경우
    - ref 접근 방법(equal 비교) 에 사용된 조건의 값이 상수임(`d0004`)을 의미
```sql
explain select * from dept_emp where detp_no='d0004';
```

#### 5. fulltext
- 전문 검색 인덱스를 사용해서 레코드를 읽는 경우
- 우선순위가 높아 인덱스 접근 방식이 const, eq_ref, ref 가 아닌 경우 fulltext 방식을 사용할 가능성이 높음
- <b> MySQL 에서는 우선순위는 높지만, 실제 쿼리 성능을 비교해보았을 때 range 접근 방식이 유리한 경우가 많기 때문에 전문 검색을 하는 경우 조건 별로 성능 비교를 해보는 것 권장 </b>
- <b> 참고 </b>
  - 전문 검색 인덱스는 통계 정보가 관리되지 않음
  - 전문 검색을 위해서는 반드시 전문 검색 전용 인덱스가 필요함

#### 6. ref_or_nll
- ref 접근 방식과 동일함
- null 비교 추가

#### 7. unique_subquery
- in 조건에 select 문을 사용한 경우, 해당 절에 중복값이 없을 때

#### 8. index_subquery
- in 조건에 select 절 또는 상수값이 올 때 중복 제거가 필요한 경우
- unique_subquery : in 절의 변수에 중복 값이 없는 경우

#### 9. range
- 인덱스 레인지 스캔
- const, ref, range 를 묶어서 인덱스 레인지 스캔이라고 부르기도 함

#### 10. index_merge
- 하나 이상의 인덱스를 사용한 경우
- 여러 인덱스를 읽어야 하기때문에 range 보다 성능이 떨어지고
- 전문 검색 인덱스와 함께 사용 불가
- 두개 이상의 인덱스 결과를 합치기 두 결과의 교집합, 합집합, 중복 제거 등의 부가 작업이 필요

#### 11. index
- 인덱스 풀 스캔
- range, const, ref 사용 불가능 && 커버링 인덱스로 처리 가능
- range, const, ref 사용 불가능 && 인덱스로 정렬 또는 그루핑 가능한 경우

#### 12. all
- 풀 테이블 스캔
- 리드 어헤드 기능으로 연속된 페이지를 몇 번 읽은 후에는 백그라운드 스레드에서 처리함
  - `innodb_read_ahead_threshold`
  - `innodb_random_ahead_threshold`
