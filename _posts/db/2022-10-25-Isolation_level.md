---
title:  "Inno DB Isolation level"
excerpt: "Inno DB Isolation level"

categories:
- DB
  
tags:
- Isolation level

toc: true

toc_sticky: true

toc_label: "lock"

last_modified_at: 2022-10-25T07:37:00-05:00

---

## Transaction Isolation Levels(Mysql 8.0)


### Isolation

---

- ACID의 I를 담당
- Isolation level은 다양한 트랜잭션이 동작할 때, 성능과 신뢰성, 일관성, 결과 재현성간의 균형을 맞춰줌
- Inno DB는 4가지 Isolation level을 제공
- 세션에 대해서 Isolation level을 지정할 수 있음
  - spring, 
    ~~~java
    @Transcational(isolation = Isolation.READ_COMMITTED)
    public void betranscational() {
        // ...
    }
    ~~~
    
- 별도 설정하지 않으면, 글로벌 설정을 따라가게됨
- Isolation level은 locking 전략을 사용해서 구별됨


### REPEATABLE_READ

---

- Inno DB Default Isolation level
- Consistent Nonlocking read
  - 한 트랜잭션 내에서 첫번째로 읽은 레코드의 스냅샷을 계속 읽음(SELECT)
- Locking read
  - Unique index로 unique search 조건으로 select 시에는, row-level의 lock이 발생
  - Index range scan이 발생하면, Gap lock이나 next-key lock이 발생하여 다른세션으로부터 gap사이에 새로운 데이터가 들어오는걸 막아줌

### READ_COMMITTED

---

- Consistent Nonlocking read
  - 가장 최신의 스냅샷을 읽음
- Locking read
  - Gap과 관련된 lock은 사용되지 않고 index record lock만 사용
  - Gap lock은 오직 외래키에 대한 제약을 체크할 때나 중복키 체크에 사용

- Gap locking이 불가능하기 때문에, 새로운 레코드가 확인되는 Phantom row 현상 발생
- UPDATE, DELETE 발생에 대해서 오직 해당되는 row들만 lock을 잡고 있기 떄문에, deadlock을 줄이는데 효과적
- 이미 락이 걸려 있는 Row의 Record 대해서 UPDATE가 필요할 때는, 가장 최근 커밋된 버전의 레코드를 가져와서 데이터가 일치하는지 확인하고 Lock 획득 여부를 결

### Locking example(REPEATABLE_READ & READ_COMMITTED)

#### CREATE
~~~sql
CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
COMMIT;
~~~
  - 해당 테이블은 인덱스가 없어서, hidden clustered index가 record locking을 위해서 사용되는 상황
  - 클러스터 인덱스?
    - 테이블당 1개
    - 테이블을 정의할 때, PK가 Inno DB 클러스터 인덱스로 사용
    - PK를 정의하지 않으면, UNIQUE Index의 NOT NULL인 모든 컬럼을 클러스터 인덱스로 사용
    - PK나 UNIQUE Index가 없는 상황이라면, Inno DB는 클러스터 인덱스(GEN_CLUST_INDEX)를 생성하고 InnoDB가 정의한 row ID로 정렬이 되어 있
    - 항상 정렬 상태를 유지
      - INSERT, UPDATE, DELETE 비용이 비싸다
    - 클러스터 인덱스가 데이터가 많을수록 강력해진다 
      - Row data를 포함하고 있는 데이터로 바로 연결이 되기 때문에, 클러스터 인덱스는 I/O가 절약됨
      - 보조 인덱스는 다양한 페이지에 Row data를 저장하므로, I/O 비용이 있음
  - 보조 인덱스?
    - 클러스터형 인덱스 이외의 인덱스를 보조 인덱스라고 한다
    - 보조 인덱스의 각 레코드에는 보조 인덱스에 대해 지정된 열과 기본키 열이 포함
    - 이 기본키를 통해서 클러스터형 인덱스에서 행을 검색
  

#### UPDATE
~~~sql
# Session A (First update)
START TRANSACTION;
UPDATE t SET b = 5 WHERE b = 3;

# Session B (Second update)
UPDATE t SET b = 4 WHERE b = 2;
~~~


- 위 상황에서,
- REPEATABLE_READ
  ~~~sql
  x-lock(1,2); retain x-lock
  x-lock(2,3); update(2,3) to (2,5); retain x-lock
  x-lock(3,2); retain x-lock
  x-lock(4,3); update(4,3) to (4,5); retain x-lock
  x-lock(5,2); retain x-lock
  ~~~
  
    - First update에 의해서 모든 row에 대해서 읽고, lock을 해제하지 않는다,
  ~~~sql
  x-lock(1,2); block and wait for first UPDATE to commit or roll back
  ~~~
  
    - Second update는 first update가 commit이 되거나 roll back되기를 기다린다
  
- READ_COMMITTED
  ~~~sql
  x-lock(1,2); unlock(1,2)
  x-lock(2,3); update(2,3) to (2,5); retain x-lock
  x-lock(3,2); unlock(3,2)
  x-lock(4,3); update(4,3) to (4,5); retain x-lock
  x-lock(5,2); unlock(5,2)
  ~~~
  
    - First update에 의해 모든 row를 읽고, 수정되지 않는 레코드에 대해서는 lock을 소유하지 않음
  ~~~sql
  x-lock(1,2); update(1,2) to (1,4); retain x-lock
  x-lock(2,3); unlock(2,3)
  x-lock(3,2); update(3,2) to (3,4); retain x-lock
  x-lock(4,3); unlock(4,3)
  x-lock(5,2); update(5,2) to (5,4); retain x-lock
  ~~~
     - Second update는 반일관적 읽기를 통해서, 가장 최근에 커밋된 레코드를 통해, update문을 수행
  
- 참고로! 위 상황들은, 클러스터 인덱스가 사용될 때 상황이고, 보조 인덱스로 진행되는 경우에는 해당 컬럼에만 lock이 걸리게 된다
  
  
### READ_UNCOMMITTED

---

- 커밋이 이미 됐지만, 트랜잭션에서 이전 버전의 데이터를 보고 있을 수 있음(Dirty read)
- 그 외에는 READ_COMMITTED와 동일

### SERIALIZABLE

---

- autocommit?
  - 1로 설정할 경우(활성화), 테이블에 대한 변화가 즉시 반영됨
  - 0으로 설정할 경우는(비활성화), COMMIT를 통해서 반영시키거나, 반영시킨 내용을 ROLLBACK을 사용해서 롤백할 수 있음
  
- autocommit이 비활성화되어 있을 경우에, SELECT를 SELECT FOR SHARE(S-Lock)으로 변경
- autocommit이 활성화되어 있을 경우에는, SELECT는 자체가 트랜잭션이 된다
  - SELECT 자체가 트랜잭션이 될 수 있으므로, 이 격리 수준은 직렬화 단계로 데이터를 읽을 수 있으며,
    - 직렬화?
      - 여러 트랜잭션이 동시에 병행 수행되더라도, 각 트랜잭션이 하나씩 차례대로 수행하여 데이터 베이스의 일관성을 보장
  - 다른 트랜잭션을 블락할 필요가 없다.


## Reference

- [Mysql 8.0](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)