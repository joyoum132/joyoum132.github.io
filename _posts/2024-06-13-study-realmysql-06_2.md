---
title: 06. 인덱스(2)
description: MySQL 의 기타 인덱스와 알고리즘
date: 2024-06-13 +0900
categories: [Study, Real MySQL 8.0]
tags: [Team Study, RealMySQL, index]
---
## <b>R-Tree(공간 인덱스)</b>
> 공간 데이터 저장 및 검색에 사용되는 알고리즘 <br>
> MySQL 의 컬럼는 2차원의 공간 데이터 저장, 검색, 연산이 가능하며
> 공간 데이터 검색시 R-Tree 자료구조 기반의 인덱스 테이블이 생성
{: .prompt-info}

### [ MBR (Minimum Bounding Rectangle) ]
- 최소 경계 도형
- 공간 데이터 종류
  - 점 (Point)
  - 선 (Line)
  - 도형 (Polygon)
- 공간 데이터를 감싸는 최소 크기의 사각형
- 가장 최소 크기의 MBR 은 상위 MBR 로 감싸지고 가장 바깥의 MBR 이 ROOT, 가장 작은 단위의 MBR 이 Leaf 가 됨

### [ R-Tree 인덱스의 연산 ]
<b>반경 이내의 좌표 검색히기</b>
- 좌표를 중심으로 컴파스를 원으로 그린다고 할 때, 원에 대한 MBR 내부에 좌표가 있는지 검색
- 원이 아닌 사각형을 중심으로 검색하기 때문에 <u>원 바깥, 사각형 안 </u> 좌표도 검색 결과에 포함
```sql
-- parameter 순서 다름 유의
select * from locations where ST_Contains(반경, 중심 좌표 점)
select * from locations where ST_Within(중심 좌표 점, 반경)
```
- 정확히 원 내부의 좌표만 검색하기 위해서는 ST_Distance_Sphere() 함수를 조건에 추가해야 함

<br>

---
## <b>전문 검색 인덱스</b>
- 저장 데이터가 큰 컬럼(TEXT, LONGTEXT 등)에 대한 인덱스
  - 일반적으로 인덱스 테이블에 사용되는 컬럼의 크기는 작지만, 이와 달리 문서 단위를 저장하는 컬럼에 대한 인덱스
- 전문 검색 인덱스에서는 문서 내용 중 사용자가 검색할만한 키워드로 인덱스를 구축
- 키워드 분류 과정에서 중요한 점
  - 불용어 처리
  - 어근 분석
  

### [ 키워드를 추출하는 방법에 따라 분류 ]
#### 1. 어근 분석
- 한글의 경우 문장의 형태소 분석을 통해 명사, 조사를 구분
- 형태소 분석을 위해서는 MeCab 플러그인을 주로 사용
- MeCab 플러그인은 MySQL 에 쉽게 설정할 수 있고 전문적인 전문 분석 알고리즘 기법을 제공하지만 완성도를 위한 추가적인 노력과 절차가 복잡함

#### 2. n-gram 알고리즘
- 문서를 글자 단위로 쪼개서 단어를 추출하고 인덱싱하는 방법
- 문장을 이해하지 않기때문에 MeCab 플러그인 적용보다 범용적으로 사용
  - 단 정확도나 전문성은 MeCab 보다 떨어질 수 있음
- n 은 토큰화 할 문자 개수를 의미
- 방식
  - 공백과 마침표(.) 기준으로 문자열 분리 후 n 글자씩 중첩해서 토큰 분리
  - 2-gram 알고리즘이 "to be or not to be" 를 분석한 결과
    - to
    - bo
    - or
    - not -> no, no
  - 분석 결과를 바탕으로 불용어는 제외시킴 (불용어를 포함하거나, 일치하거나)


### [ 불용어 핸들링 ]
#### 1. 스토리지 엔진 관계 없이 사용
- 불용어 파일 경로 설정(시스템 변수임)
```sql
ft_stopword_file= ''
```
- 빈 값을 넣어주면 불용어가 없는 상태
- 파일 경로 넣어주면 사용자 지정 불용어 정의 가능

#### 2. InnoDB 에서 사용
- 불용어 사용 ON, OFF (시스템 변수 아님)
```sql
innodb_ft_enable_stopword=OFF -- 다른 스토리지 엔진 테이블에는 불용어 적용 
```

- 불용어 전용 테이블 생성
```sql
set global innodb_ft_server_stopword_table=''
```
- 불용어 전용 테이블 생성 후 불용어 전용 인덱스 생성해아하고
- 전문 검색 전용 문법을 사용해야 함(미사용시 테이블 풀 스캔)
  ```sql
  alter table add FULTEXT INDEX index_name(idx_column) WITH PARSER ngram; -- 전문 검색 전용 인덱스 생성
  select * from doc_table where MATCH(idx_column) AGAINST ('keyword1' IN BOOLEAN MODE);
  ```

<br>

---
## <b>함수 기반 인덱스</b>
- 데이터를 조회할 때 값을 변경하는 경우 사용하는 경우 유용

#### 1. 가상 컬럼 이용
- 8.0 이상부터 가능

> 5.7 까지는 가상 컬럼은 논리적으로 존재하는 컬럼으로써 인덱스로 사용할 수 없었음 <br>
> 즉, 논리적으로 존재하다가 쿼리 시에 실시간으로 계산되어 사용 <br>
> 8.0 에서도 물리적인 컬럼으로 존재하는건 아니지만 가상컬럼을 인덱스로는 사용할 수 있게됨 (B-Tree 테이블 내에만 존재..)
{: prompt-info}

  ```sql
    alter table user
    add user_info varchar(256) as (concat(login_id, ':', nickname)) virtual,
    add index idx_userinfo(user_info);
  ```
![가상 컬럼 생성 후 테이블 정보](assets/docs/realmysql/ch8_06_1.png){: width="750"}


#### 2. 함수 이용
- 8.0 이상부터 가능
- 실제 컬럼이 아닌 논리적으로 존재
> 함수를 직접 사용하는 인덱스는 테이블 구조는 변경하지 않고, 계산된 결괏값의 검색을 빠르게 만들어 준다.
  - 가상 컬럼과 마찬가지로 물리적으로 새로운 컬럼을 추가하진 않되, 함수를 적용한 결과로 인덱스 테이블을 생성한다는 의미
- 인덱스 생성에 사용된 함수를 where 조건에 그대로 사용해야 함

> 이 두가지 방법은 사용법과 SQL 문장의 문법에서 조금 차이가 있다. 하지만 실제 가상 컬럼(Virtual Column)을 이용한 방법과
> 직접 함수를 이용한 함수 기반 인덱스는 동일한 구현 방법을 사용한다.
> 어떤 방법을 사용하더라도 둘의 성능 차이는 발생하지 않는다는 것을 의미한다.
> 
- 물리 컬럼으로서 존재X, B-Tree 의 키(Key) 로만 사용한다는 의미
