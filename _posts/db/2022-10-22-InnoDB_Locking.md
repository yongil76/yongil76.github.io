---
title:  "Inno DB lock"
excerpt: "Inno DB lock"

categories:
- DB
  
tags:
- lock

toc: true

toc_sticky: true

toc_label: "lock"

last_modified_at: 2022-10-22T07:37:00-05:00

## Inno DB locking(Mysql 8.0)

### Shared lock
- Inno DB row-level 잠금
- Shared lock은 트랜잭션이 row를 읽기 위해 잠금


### Exclusive lock
- Inno DB row-level 잠금
- Exclusive lock은 트랜잭션이 row를 업데이트하거나 삭제할 수 있기 위한 잠금

### Situation(lock)
- 두 개의 트랜잭션(T1, T2)이 각 각 있는 상황
- T1이 row에 대해서 Shared lock을 가지고 있고, T2가 lock을 요청하는 상황
  - T2가 요청하는 lock이, Shared lock이라면 T1, T2 모두 lock을 소유하고 있음
  - T2가 요청하는 lock이, Exlcusive lock이라면 T1은 lock을 얻기 위해 기다려야함
  

- T1이 row에 대해서 Exclusive lock을 가지고 있고, T2가 lock을 요청하는 상황
  - T2가 요청하는 lock에 관계없이, T2는 lock을 얻기 위해 기다려야함
  

- lock을 기다리는 상황이 지속되면, lock wait timeout이 발생할 수 있음
  - Timeout을 단순히 늘리는 방법(SESSION, GLOBAL)
  - Isolation level 조정을 통해서, 동시성을 증가시키는 방법


### Intention lock?
~~~sql
SELECT ... FOR SHARE // Intention shared lock(IS)
SELECT ... FOR UPDATE // Intention exclusive lock(IX)
~~~
- Table-level 잠금(여러 개의 Row)
- Row-level의 Shared lock을 얻기 위해서는, IS를 먼저 얻어야 함
- Row-level의 Exclusive lock을 얻기 위해서는, IX를 먼저 얻어야 함

### Intention shared lock(IS)
- Table의 Row들에 대해서 Shared lock 획득

### Intention exclusive lock(IX)
- Table의 Row들에 대해서 Exclusive lock 획득


### Relation
| |X|IX|S|IS| 
|---|---|---|---|---|
|X|Conflict|Conflict |Conflict| Conflict| 
|IX |Conflict |Compatible |Conflict |Compatible| 
|S |Conflict |Conflict| Compatible| Compatible| 
|IS| Conflict |Compatible| Compatible| Compatible|

- Compatible인 경우 lock을 가질 수 있음
- Conflict인 경우는 lock이 해제될 때까지 대기
- Conflict이지만, lock을 얻을 수 없다고 판단이 되는 경우 Deadlock 발생
- Intention lock끼리는 Compatible인 이유는, Intention lock은 row-level의 잠금이 이뤄질꺼라고 알려주는 역할

### Record locks
- Row의 인덱스 레코드에 lock
~~~sql
SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;
~~~
- 위 쿼리가 발생하면, 다른 트랜잭션들이 t.c1 = 10 레코드에 INSERT,UPDATE,DELETE를 하지 못함
- Record lock은 항상 Index lock이다
  - 테이블에 인덱스 설정이 없더라도, Inno DB는 숨겨진 클러스터 인덱스를 만들고, 이 인덱스를 사용해서 레코드 락 발생

### Gap locks
- Gap lock은 index 레코드들 사이(Between), 이전(before), 이후(after)에 발생하는 lock
~~~sql
SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;
~~~
- 위 쿼리가 발생하면, t.c1 = 15에 INSERT 할 수 없음
- 단일 인덱스, 결합 인덱스, 빈 값에 대해서도 확장될 수 있음
- Gap lock은 성능과 동시성에 대한 Tradeoff, 일부 격리 수준에서 사용하고 있음

~~~sql
SELECT * FROM child WHERE id = 100;
~~~
- id가 인덱스가 아니거나, nonunique 인덱스에 속한다면 이전 레코드들을 lock
  - id = 100인 Row가 삽입되면 결과가 달라지므로
  
- Gap S-lock, Gap X-lock?
  - 한 트랜잭션에서 레코드가 인덱스에서 제거되는 경우에 다른 트랜잭션이 레코드에 보유하고 있는 Gap lock을 병합해야 하기 때문에 공존
  - Gap lock의 유일한 목적은 Gap 사이에 Insert를 막는 것
  - 따라서, Gap lock이 공존할 수 있으며, Gap S-lock, Gap X-lock은 차이가 없고 동일한 기능을 수행
  
  
- Gap lock을 비활성
  - READ_COMMITTED
    - search, index scan에 대해서 disable
    - 외래키 제약 조건 검사 및 중복키 검사에만 사용
    - Gap lock이 없기 때문에 Phantom read 현상이 발생
    ~~~sql
    SELECT * FROM child WHERE id > 100 FOR UPDATE;
    ~~~
    
### Next-Key Locks
- 인덱스 레코드에 대한 record lock + 인덱스 레코드 이전 간격에 대한 gap lock
- 한 트랜잭션이 record R에 대해 record lock을 가지고 있으면, 다른 트랜잭션은 신규 레코드를 즉시 삽입할 수 없음
- 격리 수준 REPEATABLE_READ에서 Next-key Lock을 통해서 Phantom Read를 억제

### Insert Intention Locks  
- INSERT 쿼리가 발생되기 이전에, 발생하는 Gap lock
- 같은 gap lock을 가진 트랜잭션들이 서로 INSERT시에 서로 기다리지 않게 하기 위함이 목적
  - 두 개의 트랜잭션이 4, 7에 gap lock을 가지고 있는 상황이라고 생각하면,
  - 각 각 5와 6을 삽입하려고 한다면, 서로를 Block하지 않음
  
### AUTO-INC Locks
- AUTO_INCREMENT column에 INSERT하기 위해 얻는 Table-level lock
- 어떤 트랜잭션이, AUTO_INCREMENT column에 데이터를 삽입하는중이라면, 다른 트랜잭션들은 대기해야 한다
- innodb_autoinc_lock_mode를 통해서 자동 증분과 관련된 알고리즘 제어를 통해서, 동시성을 확보할 수 있음


### Reference

--- 

- https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html