---
title: 10. 실행계획 - 통계 정보, 실행 계획 확인
description: 실행 계획에 사용되는 통계 정보와 각 값을 확인하는 방법
date: 2024-07-20 +0900
categories: [Study, Real MySQL 8.0]
tags: [Team Study, RealMySQL, explain]
---

## 통계 정보
> 옵티마이저가 실행 계획을 생성할 때에는 여러 통계정보를 활용함 <br>
> 5.7 버전까지는 테이블, 인덱스에 대한 통계 정보만을 활용했지만, 8.0 부터는 인덱스되지 않은 컬럼에 대해서도 
> 데이터 분포를 수집해서 저장한 히스토그램 정보도 활용함
{: .prompt-info}

### [ 테이블 및 인덱스 통계 정보 ]
- MySQL 5.6 부터 스토리지 엔진을 사용하는 테이블의 통계 정보를 영구적으로 관리할 수 있도록 개선
  - 기존에는 메모리에서 관리되다보니 서버 재시작시 그동안의 통계 정보를 잃어버림

#### 1. 통계 정보 관리 테이블
- 관련 테이블
  - innodb_index_stats
  - innodb_table_stats
- 테이블에서 관리하는 정보
  - 인덱스
    - n_diff_pfx% : 인덱스가 가진 유니크 값으 개수
    - n_leaf_pages : 인덱스의 리프 노드 페이지 개수
    - size : 인덱스 트리의 전체 페이지 개수
  - 테이블
    - n_rows : 테이블의 전체 레코드 개수
    - clustered_index_size : 프라이머리 키의 크기 (InnoDB 페이지 개수)
    - sum_of_other_index_sizes : 프라이머리 키를 제외한 인덱스의 크기(InnoDB 페이지 개수)
- 5.6 부터는 테이블을 생성할 때 통계 정보의 영구 저장에 대한 옵션 설정이 가능함
  - `innodb_stats_persistent`
    - default 는 1 (true)
- 테이블별로 alter table, create table 으로 적용할 수 있음

#### 2. 통계 정보 자동 갱신
- 통계 정보를 자동 갱신하게되면 이전에는 range scan 으로 실행하던 쿼리가 테이블 풀 스캔을 하게될 수 있음
  - 자동 갱신 시점을 알지 못하기 때문에!!
- 자동 갱신 설정 변경
  - `innodb_stats_auto_recalc`
  - default 는 1 (true)
  - 영구적으로 이용하려면 0 으로 바꾸는 것을 권장
    - analyze table 명령 실행에만 수집
- 테이블별로 alter table, create table 으로 적용할 수 있음

#### 3. 통계 정보 샘플링
- innodb_stats_transient_sample_pages
  - 통계 정보 수집시 8개의 페이지만 임의로 샘플링하고 그 결과를 통계 정보로 활용함
  - default : 8
- innodb_stats_persistent_sample_pages
  - analyze table 명령어 사용시 임의 20 페이지를 샘플링해서 분석하고 그 결과를 영구 통계 테이블에 저장
  - default : 20

### [ 히스토그램 ]
> 컬럼의 데이터 분포를 저장하는 정보
{: .prompt-info}

#### 1. 히스토그램 정보 수집
- 자동으로 생성하지 않고 수동으로 명령어 작성 필요

```sql
-- 히스토그램 생성하기
ANALYZE TABLE employee.employees
UPDATE HISTORAM ON gender, hire_date; -- 원하는 컬럼에 대해서 생성

-- 생성된 히스토그램 확인
SELECT * 
FROM COLUMN_STATISTICS
WHERE schema_name = 'employee' and table_name='employee';
```

- 히스토그램으로 확인 가능한 정보
  - 1. histogram-type
    - Singleton
      - 컬럼 값 개별로 레코드 건수를 관리
      - value-based 히스토그램 또는 도수 분포라고 불림
      - 컬럼의 값, 발생 빈도의 비율 2개의 값을 가짐
      - ex) gender 컬럼은 남/여 두 값을 갖는 각각의 컬럼에 대해 히스토그램이 생성됨
    - Equi-Height
      - 컬럼의 범위를 균등한 개수로 구분해서 관리
      - height-based 히스토그램이라고 불림
      - 범위 시작 값, 범위 마지막 값, 빈도, 유니크 값 개수 4개의 값 가짐
      - ex) hire_date 와 같이 날짜와 관련된 값은 일정 범위로 나누어서 구간별로 히스토그램 생성
  - 2. sampling-rate
    - 히스토그램 정보 수집을 위해 스캔한 페이지의 비율
    - .35라면 전체 페이지 중 35% 를 스캔해서 히스토그램을 생성했음을 의미
    - ❗️8.0.19 미만은 해당 값에 상관 없에 테이블 풀스캔해서 히스토그램을 수집하고 있으므로 주의
  - 3. number-of-buckets-specified
    - 히스토그램 생성 시 설정한 버킷 개수
    - default : 100, max : 1024
    - 일반적으로 디폴드 값으로도 충분하다고 함
- 히스토그램 사용 제어하기
  ```sql
  -- 히스토그램 테이블 삭제
  ANALYZE TABLE employee.employee
  DRIP HISTOGRAM ON gender, hire_date;
  
  --condition_fanout_filter 옵션에 영향받는 다른 최적화 기능에도 영향
  set gloabl optimizer_switch='condition_fanout_filter=off';
        
  -- 현재 커넥션에서만 사용 안함
  set session optimizer_switch='condition_fanout_filter=off';
    
  -- 현재 쿼리에서만 사용 안함
  select /* set_val(optimizer_switch='condition_fanout_filter=off')*/ * from ~
  ```

#### 2. 히스토그램 용도
- 히스토그램이 존재하기 전에도 테이블과 인덱스에 대한 통계 정보가 존재했지만 테이블의 총 레코드 수와 인덱스 컬럼의 유니크 컬럼 개수 정도가 다였음
  - 정확한 실행 계획을 계산하기에는 정보가 부족함
- 히스토그램을 활용함으로써 드라이빙 테이블을 선택하거나, 컬럼의 필터링 퍼센티지 등의 계산의 정확도를 높일 수 있고
- 이는 실행 계획 분석을 통해 쿼리 비용을 적절하게 계산할 수 있음

#### 3. 인덱스와 히스토그램
- 인덱스가 있는 컬럼을 쿼리 조건을 사용하는 경우 히스토그램을 사용할까?
  - <b>사용하지 않는다</b>
  - 실제 인덱스 테이블(B-Tree) 를 샘플링해서 살펴봄 → 인덱스 다이브
  - 실제 검색 대상에 대한 샘플링이므로 히스토그램 분석보다 정확하기 때문
- 히스토그램은 인덱스가 없는 컬럼의 데이터 분포도를 참조하는 용도로 사용
- 인덱스가 있는 컬럼을 조건으로 사용하더라도 경우에 따라(in 조건으로 너무 많은 값 사용) B-Tree 샘플링 비용이 높을 수 있음

### [ 코스트 모델 ]
- 쿼리를 처리하기 위한 작업들의 비용을 의미
  - 디스크로부터 데이터 페이지 읽기
  - 메모리로부터(InnoDB 버퍼풀) 데이터 페이지 읽기
  - 인덱스 키 비교
  - 레코드 평가
  - 메모리 임시 테이블 작업
  - 디스크 임시 테이블 작업
- 5.7 이전까지는 각 단계별 비용을 상수화해서 사용했지만, 그 이후 버전에서는 값을 수정할 수 있도록 변경됨
- 관련 테이블 및 컬럼
  - server_cost : 인덱스를 찾고 레코드를 비교하고 임시테이블 처리에 대한 비용 관리
    - cost_name : 코스트 모델 각 단위 작업 이름
    - default_value : 단위 작업 비용(이전의 상수 값)
    - cost_value : 단위 비용 (관리자가 수정한 값)
    - last_updated
    - comment
  - engine_cost : 레코드를 가진 데이터 페이지를 가져오는데 필요한 비용 관리
    - 위 5개 동일
    - engine_name : 스토리지 엔진 이름
    - device_type : 디스크 타입
- cost_name 별 비용에 따라 옵티마이저가 실행 계획에 사용할 전략을 변경함
  - 예를들어, 
  - key_compare_cost(인덱스 키 비교) 비용이 높으면 가능하면 정렬하지 않는 방향의 실행 계획을 선택할 가능성이 높아짐
  - row_evaluate_cost(레코드 비교) 비용이 높으면 테이블 풀 스캔 비용이 높아지기 때문에 가능한 인덱스 레인지 스캔 방식을 선택할 가능성이 높음
  - dist_temptable_create_cost(디스크 임시 테이블 생성), dist_temptable_row_cost(디스크 임시 테이블 읽기) 비용이 높으면 디스크에 임시 테이블을 생성하지 않도록 실행 계획을 선택할 가능성이 높아짐

---
## 실행 계획 확인
### [ 출력 포멧 확인 ]
- 실행 계획 분석에는 explain, desc 가 사용
- explain 을 이용하면 포멧 지정이 가능
  - explain
  - explain format=tree
  - explain format=json
- 포멧 옵션별 표시되는 정보의 차이가 존재함

### [ 쿼리 실행 시간 확인 ]
- 좀 더 구체적인 정보 조회를 위해서는 `explain analyze` 사용
  - 8.0.18 부터 추가된 기능
- tree 포멧으로 결과 출력 (포멧 변경 불가능)
  - 같은 depth : 상단에 위치한 라인이 먼저 실행
  - 다른 depth : 안쪽에 위치한 라인이 먼저 실행
- 결과 변수
  - actual time : x .. y
    - 레코드 검색에 걸린 시간(ms)
    - x : 첫 번째 레코드 조회에 걸린 시간
    - y : 마지막 레코드 조회에 걸린 시간
  - rows 
    - 일치하는 레코드 수 (결과가 여러개라면 평균 레코드 수)
  - loops
    - 레코드 조회를 반복한 횟수
