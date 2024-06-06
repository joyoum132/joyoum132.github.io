---
title: 05. 트랜잭션과 잠금
description:
date: 2024-06-06 +0900
categories: [Team Study, Real MySQL 8.0]
tags: [transaction, lock, InnoDB]
---
## [ Before Enter ]
잠금 : 동시성 제어 <br>
트랜잭션 : 데이터 정합성 보장 <br>
- 격리 수준을 통해 트랜잭션간 영향 범위 제한
- MyISAM 이나 메모리 스토리지 엔진은 트랜잭션을 지원하지 않음

<br>

---
## [ MySQL 엔진의 잠금 ]
> MySQL 엔진 레벨의 잠금은 모든 스토리지 엔진에 영향을 미치지만, 스토리지 엔진 레빌의 잠금은 스토리지 엔진 간 상호 영향을 미치지 않음
{: .prompt-info }

### 1. 글로벌 락
**MySQL 서버에 존재하는 모든 테이블을 잠금**
```sql
flush tables with read lock
```
- 실행중인 쿼리 또는 트랜잭션이 있다면 모두 완료될때까지 기다린 후에 전체를 잠금
- MyISAM, 메모리 스토리지 엔진 사용 시 mysqldump 로 전체 백업할 때 글로벌 락이 적용
  - InnoDB 스토리지 엔진에서는 좀 더 가벼운 백업락을 이용하기 때문에 DML 은 허용
  

### 2. 테이블 락
**테이블 단위 잠금**
- 방법
  - 명시적으로 잠금 : 테이블에 잠금, 잠금 해제 명령어를 사용
  - 묵시적으로 잠금 : 특정 명령어를 사용할 때 자동으로 테이블 잠금이 실행
    - MyISAM, 메모리 스토리지 엔진 : 데이터 레코드 변경 시
      - 트랜잭션을 지원하지 않기 때문에 전체 테이블을 잠가버림
    - InnoDB 스토리지 엔진 : DDL 사용시 다른 트랜잭션에서 데이터를 사용하지 못하도록 <u>테이블 쓰기 잠금 진행</u>
  

### 3. 네임드 락
**임의 문자열 단위 잠금**
- 명시적인 잠금, 잠금 해제 필요
- 여러 레코드에 대해 복잡한 요건으로 레코드를 변경하는 트랜잭션에 유용하게 사용
- **spring boot 에서 @transaction 과 함께 사용하는 경우라면 잠금의 획득/반납과 서비스 트랜잭션 분리해야 함**
  - 잠금 반납, 트랜잭션 커밋 사이에 새로운 스레드의 트랜잭션 획득을 예방하기 위함
  - lock 내부에 @transactional 메소드 호출

```sql
--문자열에 대해 time 만큼 잠금 획득 대기 (1: 성공, 0: 실패, null: 에러)
--time이 음수라면 무한대기
select get_lock(str, time)

--문자열에 잠금이 걸려있는지 확인
select is_free_lock(str)

--잠금 해제
select release_lock(str)

--모든 잠금 해제
select release_all_locks()
```


### 4. 메타 데이터 락
**DDL 쿼리 수행 시 다른 트랜잭션에서 구조 변경하지 못하도록 스키마 변경을 잠금**
```sql
ALTER TABLE
CREATE TABLE
DROP TABLE
RENAME TABLE
TRUNCATE TABLE
```
- (참고) InnoDB 에서 DDL 실행 시 테이블 락과 메타데이터 락이 동시에 발생할 수 있음
  - **`ALTER TABLE`** 쿼리를 실행하면:
    - 메타데이터 락이 걸려 다른 트랜잭션이 테이블 구조를 변경하지 못하게 함
    - 동시에, 테이블에 대한 쓰기 락이 걸려 다른 트랜잭션이 테이블에 데이터를 쓰지 못하게 함
    
<br>

- - -
## [ InnoDB 스토리지 엔진의 잠금 ]
> 레코드 기반 잠금이며 락 에스컬레이션은 없음
{: .prompt-info }

### [ 잠금 유형 ]

#### 1. 레코드 락
- 레코드 그 자체만을 잠금(Record Only Lock)
- **인덱스의 레코드를 잠금**


#### 2. 갭 락
- 잠금 대상 레코드와 인접한 레코드 사이의 간격(갭) 을 잠그는 것
- 조건에 매칭되는 갭을 잠그는 것이 아니라, 잠금 대상이 없는 갭을 결정하고 해당 갭 내에 행이 없음을 확인
  - 아래 표에서 serial_number == 3을 조회하는 경우
    - 인덱스 레인지 스캔의 첫번째인 id=2와 그 이전 레코드 사이의 serial_number 에 대해 갭락 설정 (2)
    - 인덱스 레인지 스캔의 마지막인 id=3과 그 다음 레코드의 serial_number 에 대해 갭락 설정 (4~7)
- 새로운 레코드의 Insert 를 제어하는 용도


#### 3. 넥스트 키 락
[(참고)[MySQL]MySQL 벼락치기(5) - 갭락(Gap Lock)과 넥스트 키 락(Next-Key Lock)](https://idea-sketch.tistory.com/46)
- 레코드 락 + 갭 락
- 보조 인덱스가 있어야 함
- Repeatable Read 격리 수준에서 주로 발생 

| id | serial_number (인덱스) | code |
| --- | --- | --- |
| 1 | 1 | 143 |
| 2 | 3 | 443 |
| 3 | 3 | 300 |
| 4 | 8 | 364 |
| 5 | 10 | 254 |
      
```sql

start transaction1
select * from table where serial_number between 3 and 3 -- transaction 1

start transaction2
insert into table values(2, 500) --실패
end transaction2

start transaction3
insert into table values(6, 500) --실패
end transaction3

-- serial_number의 잠금의 범위는 2~7
``` 

#### 4. 자동 증가 락
- auto-increment 의미
- `innodb_autoinc_lock_mode`
  - 0 : insert 마다 테이블 잠금 및 해제
  - 1 : 예측 가능한 bulk insert에 대해서는 한번의 잠금으로 값 증가 (중간에 loss 발생해도 값을 줄이지는 않음), 예측 어렵다면 0처럼 동작
  - 2 : 높은 동시성. 그러나 연속값이 아니기때문에 바이너리 로그 복구 어려움

<br>

---
### [ InnoDB의 레코드 잠금 == 인덱스 잠금 ]
- 데이터 변경에 필요한 데이터를 찾을 때 사용한 인덱스 레코드를 모두 잠금
- 인덱스가 없다면 테이블 풀스캔으로 대상 레코드 업데이트

```sql
--first_name 에 인덱스
SELECT COUNT(*) FROM employees WHERE first_name='DK'; -- 250건 조회된다고 가정
SELECT COUNT(*) FROM employees WHERE first_name='DK' AND last_name='J'; -- 1건 조회된다고 가정
      
UPDATE employees SET hire_date=NOW() WHERE first_name='DK' AND last_name='J'; --어디에 락이 걸릴까?
``` 
  - first_name 로 레인지 스캔 후 넥스트 키 락
  - 250개의 레코드가 잠김 대상

<br>

---
### [ InnoDB 레코드 잠금 상태 확인 ]
- information_schema 또는 performance_schema 테이블을 통해 확인 가능
    - 버전업되면서 performance_schema 가 좀 더 활성화됨
- 확인 가능한 상태
    - 실행중인 트랜잭션 목록
    - 잠금 대기 목록
    - 트랜잭션 kill
    - 등등
  
<br>

---
## [ 격리 수준 ]
> JPA 의 격리수준과 MySQL 의 격리수준이 다르다면 JPA 설정이 우선 <br>
> @transactional 의 지정 격리 수준이 우선 적용 (JPA default : Read Committed)
{: .prompt-tip }

### 1. Read Uncommitted
- Dirty Read : 커밋되지 않는 변경 데이터 조회 가능
- 먼저 진행된 트랜잭션이 롤백된다면 정합성 문제 발생


### 2. Read Comitted
- 언두 로그를 활용해서 커밋 전 데이터를 다른 트랜잭션에서 읽을 수 없도록 함
- 트랜잭션 내에서 같은 쿼리의 실행 결과가 달라지는 Non Repeatable Read 발생 가능
  - 언두로그는 select에 대한 MVCC 기능이기때문에 값이 추가되는 것을 제어하지 못함


### 3. Repeatable Read (MySQL default)
- 바이너리 로그 사용시 최소 사용 격리 수준
- 언두로그의 데이터를 읽을 때 트랜잭션 번호도 체크함
  - 자신의 트랜젝션 번호보다 빠른 트랜잭션에 의해 커밋된 데이터 혹은 언두로그만 읽음


### 4. Serializable
- 순수 Read 작업에도 레코드 잠금이 필요
- Phantom Read 발생하지 않음
  - InnoDB에서는 갭 락, 넥스트 키 락으로 Repeatable Read 격리 수준을 사용하더라도 Phantom Read는 발생하지 않음
- 동시성이 떨어지기 때문에 권장하는 격리 수준은 아님

<br>

- - -
## [ 기타 ]
### auto-commit 에 대한 정리
- auto-commit 이 true 일 때 하나의 SQL statement 가 트랜잭션 단위가 되기 때문에 쿼리 실행 후 바로 Commit
- ❓만약 트랜잭션 내에 Exception 이 발생한다면❓
  - 트랜잭션의 원자성과 auto-commit 의 개념이 충돌하지 않을까? 하는 의문이 들었음
  ```sql
  #수도코드임
  set auto-commit = true -- auto commit 활성
  
  TRANSACTION BEGIN;
  
  INSERT INTO table1 VALUES 1; -- First INSERT
  INSERT INTO table1 VALUES 1; -- Second INSERT 예외발생!!
  INSERT INTO table1 VALUES 1; -- Third INSERT
  
  COMMIT || ROLLBACK;
  TRANSACTION END
  ```
  - 정답은 **롤백**!!
  
  > In InnoDB, all user activity occurs inside a transaction. If autocommit mode is enabled, each SQL statement forms a single transaction on its own. By default, MySQL starts the session for each new connection with autocommit enabled, so MySQL does a commit after each SQL statement if that statement did not return an error. If a statement returns an error, the commit or rollback behavior depends on the error. See Section 15.21.5, “InnoDB Error Handling”.
  >
  >
  > A session that has autocommit enabled can <u> perform a multiple-statement transaction by starting it with an explicit START TRANSACTION or BEGIN statement and ending it with a COMMIT or ROLLBACK statement.</u> See Section 13.3.1, “START TRANSACTION, COMMIT, and ROLLBACK Statements”.
  >

  - 명시적으로 지정된 트랜잭션 구문 내의 multiple-statement 는 하나의 묶음으로 간주 -> 실행 중 예외 발생 시 롤백   
