---
title: 09. 옵티마이저와 힌트 - 고급최적화 3
description: 옵티마이저의 쿼리 힌트
date: 2024-07-15 +0900
categories: [Study, Real MySQL 8.0]
tags: [Team Study, RealMySQL, optimizer]
---

> 쿼리 힌트 <br>
> 옵티마이저의 실행계획이 최적이 아닐 때 실행 계획 수정을 위해 힌트를 지정할 수 있음 <br>
> <b>인덱스 힌트</b>, <b>옵티마이저 힌트</b>
{: .prompt-info}

## <b>인덱스 힌트</b>
- 옵티마이저 힌트 도입 이전에 사용되던 기능으로 ANSI 표준 X
- select, update 절에만 사용 가능

### [STRAIGHT JOIN]
- 조인 순서를 고정하는 힌트
- 일반적인 경우에 드라이빙 테이블을 결정하는 조건인데, 제대로 실행계획이 생성되지 않는 경우 사용할 수 있음
  - 조인을 위한 컬럼의 인덱스 여부
  - 조건을 만족하는 레코드가 적은 경우
- select 키워드 바로 뒤에 사용해야 하고 from 절에 명시된 테이블 순서대로 조인하도록 유도

  ```sql
  select STRAIGHT JOIN e.first_name, e.last_name, d.dept_name
  from employees e, dept_emp de, departments d
  where e.emp_no=de.emp_no and d.dept_no=de.dept_no;
  
  select /*STRAIGHT JOIN*/ e.first_name, e.last_name, d.dept_name
  from employees e, dept_emp de, departments d
  where e.emp_no=de.emp_no and d.dept_no=de.dept_no;
  ```

- 비슷한 역할
  - join_fixed_order (straight join 동일)
  - join_order (일부 테이블의 조인 순서에만 제안)
  - join_prefix (일부 테이블의 조인 순서에만 제안)
  - join_suffix (일부 테이블의 조인 순서에만 제안)

### [ USE INDEX / FORCE INDEX / IGNORE INDEX ]
- 특정 인덱스의 사용을 유도하는 힌트
- 사용하려는 인덱스를 갖는 테이블 뒤에 명시

#### 기본 사용법
1. use index
- 특정 테이블의 인덱스 사용을 권장
- 옵티마이저는 주로 힌트의 인덱스를 사용하지만 항상은 아님
2force index
- use index 보다 강한 권장
- force 를 주더라도 옵티마이저가 항상 사용하는건 아님
- 대부분의 경우 use index 로 충분함
3. ignore index
- 인덱스를 사용하지 않도록 함
- 풀 테이블 스캔을 유도하기 위해
4. 인덱스의 용도를 지정
- 인덱스 사용시 용도를 지정할 수 있음(그러나 대체로 옵티마이저가 최적을 판단을 하기때문에 자주 사용하지 않음)
  - use index for join (조인 + 레코드 검색)
  - use index for order (order by 용도로만 사용)
  - use index for group by (group by 용도로만 사용)

  ```sql
  select * from employees FORCE INDEX(PRIMARY) where emp_no=10001;   -- 클러스터링 인덱스는 PRIMARY 라고 단순하게 표시
  select * from employees USE INDEX(PRIMARY) where emp_no=10001;
  select * from employees IGNORE INDEX(PRIMARY) where emp_no=10001;
  select * from employees FORCE INDEX(ix_firstname) where emp_no=10001;
  ```

#### 주의
- 좋은 실행 계획이라는 판단을 내리기 어렵다면 임의로 인덱스를 지정하는 것은 되도록 피하자!
- 최적의 실행 계획은 항상 바뀔 수 있기때문에, 옵티마이저가 그 당시의 통계 정보를 활용해서 선택하는 것이 가장 좋음

### [SQL_CALC_FOUND_ROWS]
- 쿼리에 limit 절이 있더라도 모든 결과를 조회하고, limit 개수만을 리턴
- 개발자의 편의를 위한(데이터 확인 등) 힌트값이므로 <u>사용하지 않아야 함</u>
  ```sql
  select SQL_CALC_FOUND_ROWS * from employees where first_name='MATT' limit 20;
  select FOUND_ROWS() as totalCnt;
  ```
- totalCnt 를 가져오기 위한 추가 쿼리가 필요해 총 두 번의 쿼리 발생
- ix_firstname 이라는 인덱스가 있더라도 select * 을 위해 모든 조건과 일치하는 모든 레코드에 대한 랜덤 I/O 발생

  ```sql
  select * from employees where first_name='MATT' limit 0, 20;
  select count(*) from employees where first_name='MATT';
  ```
- 마찬가지로 쿼리는 두 번 
- select 를 위해 ix_firstname 인덱스를 사용할 수 있고, 만족하는 결과 중 20개만 조회하면 되기 때문에 20번의 랜덤 I/O 발생
- 카운트 쿼리를 실행에는 커버링 인덱스를 사용해서 랜덤 I/O 없음

## <b>옵티마이저 힌트</b>
- 영향 범위로 구분
  - 인덱스 : 특정 인덱스의 이름을 사용할 수 있음
  - 테이블 : 특정 테이블의 이름을 사용할 수 있음
  - 쿼리 블록 : 특정 쿼리 블록에 사용할 수 있음. 힌트가 명시된 쿼리 블록에만 영향
  - 글로벌 : 전체 쿼리에 영향
- 영향 범위가 다르더라도 힌트를 주는 쿼리의 위치는 고정 (select 절 다음에 주석)

### [ MAX_EXECUTION_TIME]
<b> 글로벌 </b><br>
- 옵티마이저 힌트 중 유일하게 쿼리 실행 계획에는 영향을 미치지 않음
- 실행 시간을 초과하면 쿼리 실패 리턴

  ```sql
  select /* MAX_EXECUTION_TIME(100)*/ *
  from employees
  order by last_name limit 1;
  ```
  
### [ SET_VAR ]
<b> 글로벌 </b><br>
- 실행 계획에 사용하기위해 시스템 변수 값을 변경
- 실행 계획을 바꾸기도 하고, 조인버퍼나 정렬용 버퍼(소트 버퍼) 크기를 일시적으로 증가시켜서 쿼리 성능을 향상시키기도 함

  ```sql
  select /* MAX_EXECUTION_TIME(100)*/ *
  from employees
  order by last_name limit 1;
  ```
  
### [ SEMI_JOIN / NO_SEMI_JOIN ]
<b> 쿼리 블록 </b><br>
- table pull-out 최적화 전략은 세미 조인 전략 중 가장 좋기때문에, 나머지 최적화 세미 조인 전략들 중에 우회하는 용도로 사용할 수 있음
- 외부 쿼리가 아닌 서브쿼리에 명시해야 함

1. 쿼리 블록 이름을 사용하는 방식
  ```sql
  select /* SEMI_JOIN(@sub1 MATERIALIZATION) */ *
  from departmemts d
  where d.dept_no in 
  (select /* QB_NAME(sub1)*/ de.dept_no from dept_emp de)
  order by last_name limit 1;
  ```

2. 서브쿼리에 사용하는 방식
  ```sql
  select *
  from departmemts d
  where d.dept_no in 
  (select /* SEMIJOIN(MATERIALIZATION) */ de.dept_no from dept_emp de)
  order by last_name limit 1;
  ```

### [ SUBQUERY ]
<b> 쿼리 블록 </b><br>
- 안티 세미 조인 최적화에 사용
- 세미조인 최적화 힌트와 사용 방법이 동일 (쿼리블록에 사용 또는 서브쿼리에 사용)
  - /* SUBQUERY(INTOEXISTS) */
  - /* SUBQUERY(MATERIALIZATION) */  

### [ BNL / NO_BNL / HASHJOIN / NO_HASHJOIN ] 
- BNL / NO_BNL : 이름과 달리 해시조인 사용을 유도하는 힌트
- HASHJOIN / NO_HASHJOIN : 8.0.18 버전에서만 유효

### [ JOIN_FIXED_ORDER, JOIN_ORDER, JOIN_PREFIX, JOIN_SUFFIX ]
- STRAIGHT_JOIN 은 from 절의 모든 테이블의 순서를 고정해버리기 때문에 일부만 강제하기 위한 다른 힌트들도 필요함
- JOIN_FIXED_ORDER
  - STRAIGHT_JOIN 와 동일
- JOIN_ORDER 
  - from 절의 순서가 아니라 힌트에 사용된 순서대로 조인
- JOIN_PREFIX 
  - 드라이빙 테이블만 강제
- JOIN_SUFFIX
  - 드리븐 테이블만 강제

### [ MERGE, NO_MERGE ]
- from 절의 서브쿼리 실행시 항상 내부 임시 테이블을 생성함
- 옵티마이저는 내부 임시 테이블에 사용되는 추가적인 메모리 소모를 줄이기 위해 from 절의 서브쿼리를 외부 쿼리와 병합(join 처럼 동작하게) 하는 최적화를 진행함

### [ INDEX_MERGE, NO_INDEX_MERGE ]
- 대체로 하나의 쿼리에 하나의 인덱스를 사용하지만
- 하나의 인덱스로 검색 범위를 좁히지 못한다면 다른 인덱스를 함께 사용하기도 함
- 상황마다 두개 이상의 인덱스를 사용하는것이 성능 개선에 도움이 되기도하고 오히려 성능이 나빠지기도 함

### [ NO_ICP ]
- 인덱스 컨디션 푸시다운 최적화는 사용 가능하다면 항상 성능 향상에 도움이 됨
  - 그래서 옵티마이저도 사용 가능하다면 항상 사용하도록 실행계획을 분석
  - 그래서 ICP 힌트 활성화 옵션은 없음!
- 간혹 인덱스 컨디션 푸시다운의 비용 계산이 잘못될때가 ICP 옵션을 비활성화

### [ SKIP_SCAN, NO_SKIP_SCAN ]
- 인덱스 스킵 스캔은 선행 컬럼의 유니크 개수가 적을 때 유리하다
- 상황에 따라 스킵 스캔 ON, OFF 가능
  ```sql
  explain select /* NO_SKIP_SCAN(employee ix_gender_birthday) */ gender, birthday
  from employees
  where birthday > '2000-12-31';
  ```

### [ INDEX, NO_INDEX ]
- 기존의 인덱스 힌트를 대처하는 옵티마이저 힌트가 존재
- 옵티마이저에서 관련 힌트를 줄 때에는 테이블명 + 인덱스명 함께 명시

  ```sql
  -- 인덱스 힌트
  explain select *
  from employees use_index(ix_firstname)
  where first_name = 'MATT';
  
  -- 옵티마이저 힌트
  explain select /*INDEX(employees ix_firstname)*/ *
  from employees
  where first_name = 'MATT';
  ```
