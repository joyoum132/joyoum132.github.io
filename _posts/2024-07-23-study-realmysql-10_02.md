---
title: 10. 실행계획 - 실행 계획 분석
description: 실행 계획 테이블의 칼럼 이해하기 (id, select_type, table, partitions)
date: 2024-07-23 +0900
categories: [Study, Real MySQL 8.0]
tags: [Team Study, RealMySQL, explain]
---

## 실행 계획 분석

### [ id 칼럼 ]
- select 쿼리마다 부여되는 식별자
  - 서브쿼리 포함
- 하나의 select 쿼리에서 여러 테이블을 조인하면 실행 계획의 테이블 레코드는 증가하더라도 id (식별자) 는 같은 값을 갖음
- ‼️가장 기본 포멧인 `explain ~` 을 사용할 때 id 컬럼이 테이블 접근 순서를 의미하는 것은 아님 (id 가 작다고 항상 먼저 실행되는게 아님!)
  - `explain format=tree` 를 사용해서 들여쓰기 depth 로 순서 확인 가능

---

### [ select_type 칼럼 ]
<b> select 쿼리가 어떤 타입의 쿼리인지 표시 </b>

#### 1. SIMPLE
- union, 서브쿼리가 없고 조인만 있는 쿼리
- 쿼리가 복잡하다면 가장 바깥의 select 문의 select_type 임

#### 2. PRIMARY
- union, 서브쿼리를 갖는 쿼리의 실행계획을 분석할 때 가장 바깥의 select 문의 타입
- SIMPLE 과 마찬가지로 복잡한 쿼리더라도 PRIMARY 값을 갖는 select_type 은 단 하나이고, 가장 바깥의 select 문

#### 3. UNION
- union 을 사용하는 쿼리 중 첫번째 select 를 제외한 나머지의 select_type
- union 의 첫 select 쿼리는 나머지 union 결과를 모아서 저장하는 DRIVED (임시테이블) 로 표시됨

#### 4. DEPENDENT UNION
- union 쿼리의 결과를 외부 쿼리의 조건으로 사용할 때 (union 의 결과가 외부 쿼리에 의해 영향을 받는 것)

#### 5. UNION RESULT
- union, union all 쿼리에 사용되는 임시 테이블
  - 8.0 부터 union all 사용시 임시테이블 사용하지 않도록 개선됨
- 임시 테이블이기 때문에 id 를 부여하지 않음
  - union result 레코드의 table 정보에는 임시 테이블이 저장한 레코드의 id를 표시

#### 6. SUBQUERY
- from 절 이외에 사용되는 서브쿼리를 의미
  - from 절에 사용 → DRIVED 

> 서브쿼리 용어 정리 <br>
> - 중첩된 쿼리(Nested Query) : select 절에 사용된 서브 쿼리 <br>
> - 서브 쿼리 : where 절에 사용된 서브 쿼리 <br>
> - 파생 테이블(Drived Table) : from 절에 사용된 서브쿼리 (인라인 뷰, 서브 설렉트)<br>
{: .prompt-info}
>서브쿼리가 반환하는 값 구분
> - 스칼라 서브 쿼리 : 하나의 값만 반환(칼럼 1개인 하나의 레코드)
> - 로우 서브쿼리(row subquery) : 칼럼 개수와 관계 없이 하나의 레코드 반환
{: .prompt-info}

#### 7. DEPENDENT SUBQUERY
- 서브쿼리가 바깥쪽 select 절에 정의된 컬럼을 사용하는 경우

#### 8. DERIVED
<b> 실행계획에서 확인 된다면 튜닝 대상 </b>

- from 절의 서브쿼리 결과로 임시테이블 사용하는 경우
  - 옵티마이저 옵션에 따라 from 서브쿼리를 외부 쿼리와 통합(join 처럼) 할 수 있고, 이때에는 DERIVED 표시X
- select 결과로 임시테이블을 사용하는 경우
  - 5.6 부터 임시 테이블을에도 인덱스 추가 및 적용 가능 (옵티마이저 옵션으로 제어)

#### 9. DEPENDENT DERIVED
- 래터럴 조인을 사용해서 from 절 서브쿼리에 외부 칼럼을 사용하는 경우

#### 10. UNCACHEABLE SUBQUERY
- 서브쿼리를 사용할 때
- 조건이 같은 서브쿼리는 한번 실행 후 그 결과를 내부적인 캐시 공간에 저장해서 재사용하는데, 이 캐시 기능을 사용할 수 없는 경우
  - 사용자 변수가 서브쿼리에 사용된 경우
  - UUID(), RAND() 처럼 호출할 때마다 결과가 달라지는 함수가 서브쿼리에 사용된 경우
  
#### 11. UNCACHEABLE UNION
- 위와 유사함

#### 12. MATERIALIZED
- in 절에 서브쿼리를 사용할 때, 그 결과를 임시테이블로 만들어서 조인하도록 최적화 함
  - 5.6 까지는 외부 테이블의 레코드마다 조건의 select 문을 실행해서 결과를 비교했음

--- 

### [ table 칼럼 ]
- 단위 select 문이 사용한 테이블
- from 절이 없다면 null 로 표시
- UNION, DERIVED 같은 경우 어떤 select 문의 결과로 생성된 항목인지 id 를 보여줌 (여러개라면 comma 로 구분)

---

### [ partitions 칼럼 ]
- 쿼리의 실행 결과가 저장된 파티션 테이블 표시
- 옵티마이저는 나머지 파티션에 대한 데이터 분포도 등의 분석을 실행하지 않음
- <b> 파티션 프루닝 : 쿼리의 조건을 통해 필요한 파티션 테이블을 골라내는 작업 </b>

> <b> partitions 칼럼이 있고, type 칼럼이 ALL (테이블 풀 스캔) </b><br>
> 대부분의 DBMS 에서 파티션 테이블은 물리적으로 나누어져있음 <br>
> 그렇기 때문에 전체 테이블 풀 스캔이 아니라, 프루닝 된 파티션 테이블에 대한 풀 스캔을 의미
{: .prompt-info}
