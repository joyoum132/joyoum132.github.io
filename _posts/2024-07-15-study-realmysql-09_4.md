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
