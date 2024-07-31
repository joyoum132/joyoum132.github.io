---
title: 10. 실행계획 - 실행 계획 분석 4
description: 실행 계획 테이블의 칼럼 이해하기 -  Extra 칼럼
date: 2024-07-30 +0900
categories: [Study, Real MySQL 8.0]
tags: [Team Study, RealMySQL, explain]
---

## 실행 계획 분석
### [ extra 칼럼 ]
- 성능에 대한 추가 설명으로 중요한 정보가 많음

#### 1. const row not found
- const 접근 방법으로 테이블을 읽었지만 결과가 1건도 없는 경우

#### 2. Deleting all rows
- where 문이 없는 delete 문을 사용하는 경우에 출력
  - 모든 레코드를 한번에 delete all 처리하는 API 를 호출할 때
- 8.0 부터는 출력되지 않고, where 없이 delete 를 하는 경우 truncate table 명령어를 권장

#### 3. Distinct
- 조인할 때 필요한 항목만 읽었음을 의미

#### 4. FirstMatch
- FirstMatch 세미조인 최적화 전략을 사용한 경우
- FirstMatch(테이블명) 포멧으로 출력
  - 기준 테이블 (해당 테이블의 조회 결과에서 만족하는 한 건의 레코드 조회)

#### 5. Full scan on Null key
```sql
selet * from table where con1 in (select col2 from table2 ...)
```
- 위에서 col1 이 null 일 때 서브쿼리는 테이블 풀 스캔을 하게됨을 의미
  - 내부적으로 null in (select col2 from table2) 로 바뀌고
  - 서브쿼리가 1개 이상의 레코드를 반환한다면 결과는 null
  - 서브 쿼리 결과가 0 이면 false 
    - null(알 수 없는 값) in (empty list) --> false
- 해결
  - col1 에 not null 제약조건 지정
  - 해당 쿼리 조건 앞에 co1 is not null 추가

#### 6. Impossible HAVING
- having 조건을 만족하는 레코드가 없음
- 쿼리를 잘못 작성한 경우가 많으므로 쿼리 재점검

#### 7. Impossible WHERE
- where 의 결과가 항상 false 인 경우
- 예를들어 not null 제약조건이 있는 컬럼에 is null 조건을 주는 경우

#### 8. Loose Scan
- loose scan 세미 조인 최적화 방식을 사용하는 경우

#### 9. No Matching min/max row
- min, max 집합함수를 이용하는 컬럼의 where 조건과 일치하는 레코드가 없는 경우
  - 집합함수가 없다면 Impossible WHERE 이 출력

#### 10. no matching row in const table
- 조인에 사용된 테이블에서 const 방법으로 접근했지만 일치하는 레코드가 없는 경우

#### 11. No matching rows after partition pruning
- 파티셔닝 된 테이블에 update, delete 할 때 검색 할 파티셔닝 된 테이블이 없는 경우
  - 만약 파티션 테이블이 있다면 using where 등 다른 extra 값이 출력됨

#### 12. No tables used
- from 절이 없는 쿼리, from dual 쿼리

#### 13. Not exists
- left join 할 때 드리븐 테이블에 조건에 맞는 데이터가 있다 / 없다 만 확인하는 경우
- 조건에 일치하는 레코드가 여러건이더라도 1건 조회한 이후 처리를 완료하는 최적화를 의미

#### 14. Plan isn't ready yet
- 특정 커넥션이 실행 중인 쿼리의 실행 계획을 분석할 때 `explain for connection {connection-id}`
- 해당 커넥션에서 아직 실행 계획을 수립하지 못한 상황
- 잠시 뒤에 다시 커넥션의 실행 계획을 확인하면 됨

#### 15. Range checked for each record(index map : N)
- 테이블을 조인할 때, 레코드마다 인덱스 레인지 스캔을 해야하는 경우
```sql
explain
select * from employees e1, employees e2
where e2.emp_no > e1.emp_no
```
- 괄호 안의 N 은 레인지 스캔에 사용할 인덱스 후보의 순번을 16 진수로 나타낸 값
  - 순번은 `show create table employees` 에 출력된 순서
- 이 경우 type 에 ALL 이 표시되는데, 만약 인덱스가 도움이 안된다면 테이블 풀 스캔을 하기 때문임

#### 16. Recursive
- WITH 구문으로 재귀 쿼리를 작성하는 경우
- WITH 로 작성한 CTE 가 재귀로 동작하는 경우에만 표시

#### 18. Select tables optimized away
- select 절에 min(), max() 를 사용하거나
- group by 로 min(), max() 조회하는 쿼리가 인덱스를 오름/내림차순으로 1건만 읽는 경우
- 레인지 스캔 후, 해당 범위의 가장 첫/마지막 값만 읽는 경우를 의미함

#### 19. Start temporary, End temporary
- 세미 조인 최적화를 위해 Duplicate weed-out 전략을 사용하는 경우

#### 20. Unique row not found
- 유니크 컬럼을 아우터 조인을 할 때 조건에 맞는 레코드 없음

#### 21. Using filesort
- order by 절에 사용할 인덱스가 없어 MySQL 자체 정렬 알고리즘을 사용하게될 때
- 정렬을 위해 소트버퍼를 사용하게 된다

#### 22. Using index(커버링 인덱스)
- 인덱스의 조합만으로 쿼리 실행이 가능한 경우
- type 의 index (인덱스 풀 스캔) 과 구분!!

#### 23. Using index condition
- 인덱스 컨디션 푸시 다운 최적화를 사용하는 경우

#### 24. Using index for group-by
- group by 절에 인덱스를 사용하는 경우
  - avg(), sum(), count() 를 사용한다면 인덱스를 사용하더라도 모든 범위의 값을 읽고 계산하기때문에 위 extra 항목이 표시되지 않음
  - 타이트 인덱스 스캔
- 루스 인덱스 스캔 방식을 사용하는 경우
  - where 조건이 없고 select 와 group by 에서 사용 가능한 인덱스가 있을 때
- where 조건과 group by 가 동일한 인덱스를 사용하는 경우
  - where 이 사용할 수 있는 인덱스의 우선순위가 높다
- where 조건에 사용 할 인덱스가 없고, group by 절에만 인덱스가 있는 경우
  - group by 먼저 읽은 후 where 조건 비교
  - 루스 인덱스 스캔 X
  - 타이트 인덱스 스캔 O

#### 25. Using index for skip scan
- 인덱스 스킵 스캔 방식을 이용한 경우

#### 26. Using join buffer
- 드리븐 테이블에 사용할 인덱스가 없어 블록 네스티드 조인, 해시 조인 방식을 사용하게 되어 조인 버퍼를 사용하는 경우
- 조인 버퍼는 일반 상용 서비스라면 1MB 정도면 충분

#### 27. Using MRR
- 조인할 때 MySQL 엔진이 넘겨주는 값을 하나 하나 건별로 읽어야 하는 비효율을 개선
- MRR 방식을 이용해 스토리지 엔진이 읽어야 하는 레코드를 조인 버퍼에 저장해두고 한번에 접근하는 경우

#### 28. Using sort_union(), Using union(), Using intersect()
- index_merge 방식을 사용해 두개 이상의 인덱스를 사용한 경우, 해당 결과를 어떻게 병합했는지 나타냄
- 3개로 구분
  - Using intersect
    - 각 처리 결과의 교집합만을 추출
  - Using union
    - 각 처리 결과의 합집합
  - Using sort_union
    - 각 처리 결과의 합집합이지만, using union 으로 처리하기에는 너무 많은 range 가 존재할 때
    - 프라이머리 키만 먼저 읽어서 정렬 후 병합
- 조건이 모두 동등 비교라면 using union 을 사용하고, 아니라면 sort_union 을 사용함

#### 29. Using Temporary
- 쿼리 실행 중 중간 결과를 담아두는 용도로 임시 테이블을 사용하는 경우
  - 대표적으로 인덱스를 사용 못하는 group by 절이 있을 때
- Using Temporary 가 없더라도 임시 테이블을 생성하는 경우 존재
  - from 절에 사용하는 서브쿼리
  - count(distinct col1) 을 포함하는 경우
  - union 또는 union distinct 쿼리 (union all 은 성능이 개선되어 임시테이블 사용 X)
  - 존재하는 인덱스로 정렬할 수 없는 경우에 정렬 레코드가 많아지면 임시 테이블을 사용하게 될 수 있음

#### 30. Using where
- 스토리지 엔진에서 데이터를 읽은 후 MySQL 엔진에서 추가적인 데이터 가공이 있었음을 의미
- 보통 작업 범위 결정 조건은 스토리지 엔진에서 처리하지만, 필터 조건은 MySQL 엔진에서 처리함
- `filtered` 칼럼을 함께 참고해서 Using Where 라는 값이 성능 이슈를 유발할 수 있을지 체크해보면 좋음
  - `filtered` 칼럼의 값이 작다면 스토리지 엔진이 아닌 MySQL 엔진에서 필터링하는 값이 많음을 의미하기 때문에
  - 인덱스 등을 적절히 활용해서 스토리지 엔진에서 레코드를 많이 거를 수 있도록 수정하면 좋음

#### 31. Zero limit
- 쿼리 결과는 필요 없고 결과에 대한 메타데이터만 필요할 때 쿼리 마지막에 `limit 0` 을 붙여주면 되는데
- 그 경우에 해당함


