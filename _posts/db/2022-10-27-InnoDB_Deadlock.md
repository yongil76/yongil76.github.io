---
title:  "Deadlocks in InnoDB"
excerpt: "Deadlocks in InnoDB"

categories:
- DB
  
tags:
- Deadlock

toc: true

toc_sticky: true

toc_label: "Deadlock"

last_modified_at: 2022-10-27T07:37:00-05:00

---

## Deadlocks in InnoDB


### Deadlock?

---

- 다른 트랜잭션들이 서로의 필요한 락을 가지고 있기 때문에, 진행할 수 없는 상황
- Deadlock은 UPDATE, SELECT FOR UPDATE처럼 테이블에 락을 거는 경우에 발생할 수 있다
- Deadlock은 Index record의 range와 gap에 대해서 발생할 수 있음
- Deadlock의 가능성을 줄이려면?
  - LOCK TABLES와 같은 명령어보단 트랜잭션 단위를 사용
    - 트랜잭션은 INSERT, UPDATE 발생 시에, 불필요하게 오랫동안 열어두지 않음
  - Deadlock의 가능성은 Isolation level의 영향을 받지 않는다
    - 왜냐하면 Isolation level은 읽기 작업의 동작을 변화시키며, Deadlock은 쓰기 동작때문에 발생
      - Isolation level의 따른 읽기 작업 : Dirty read, Phantom row
- Deadlock이 발생하면, 트랜잭션들중에 하나를 희생자로 삼아 롤백시킨다.
  - 만약에 Deadlock detection이 비활성화(innodb_deadlock_detect)되어 있으면, Inno DB는 lcok wait timeout을 통해서 롤백시킨다
- 마지막 Deadlock 상태를 보기 위해서는 SHOW_ENGINE_INNODB_STATUS를 사용
- innodb_print_all_deadlocks 활성화를 통해서, 빈번하게 자주 발생하는 Deadlock 정보를 로그 발생시킬 수 있음

### An InnoDB Deadlock Example

---

- 두 세션이 있다고 가정하자, A, B
- 아래 순서대로, 쿼리가 발생하고 있는 상황

#### A 세션

~~~sql
mysql> CREATE TABLE t (i INT) ENGINE = InnoDB;
Query OK, 0 rows affected (1.07 sec)

mysql> INSERT INTO t (i) VALUES(1);
Query OK, 1 row affected (0.09 sec)

mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM t WHERE i = 1 FOR SHARE;
+------+
| i    |
+------+
|    1 |
+------+
~~~

- A는 i = 1인 레코드에 대해서 S-lock을 발생시킨 상황

#### B 세션

~~~sql
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> DELETE FROM t WHERE i = 1;
~~~

- B는 DELETE하기 위해서는, X-lock을 얻어야 한다, 그러나 A에 의해 발생한 S-lock으로 인해서 얻을 수 없음
- B의 요청은, lock request를 위한 queue에 들어가게 되고, B는 block이 된다

#### A 세션

~~~sql
DELETE FROM t WHERE i = 1;
~~~
- A가 S-lock을 잡고 있는 상황에서, X-lock을 필요로 한다.
- 그러나, B가 i = 1에 대해서 X-lock을 먼저 요청하고 있고, A의 S-lock 해제를 기다리고 있음
- B의 요청이 먼저 있는 상황때문에, A의 X-lock 요청은 먼저 이뤄질 수 없음
- 결과적으로, Inno DB는 에러를 발생시키고, A, B 세션 중 하나에 오류를 발생시키고 잠금을 해제시킴

~~~sql
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
~~~

#### 가장 중요한 점은, B에 의해서 lock request가 발생했을 때, A가 delete 쿼리를 발생시켰다는 것

### Deadlock Detection

---

- Deadlock detection이 활성화되면, Inno DB는 자동으로 deadlock을 탐지하고 deadlock을 깨기 위해서, 트랜잭션 혹은 트랜잭션들을 롤백시킴
- Inno DB는 가장 작은 단위를 롤백할 수 있는 트랜잭션을 선택한다. 작은 단위는 insert, update, delete의 수에 의존한다.
- Inno DB는 deadlock을 탐지하기 위해서, table-level, row-level 락을 인식하고 있다.
- 높은 동시성 시스템에서는, deadlock detection이 수많은 스레드들에 대해 성능 저하를 발생시킬 수 있음
  - 이럴 경우, innodb_deadlock_detect를 disable하고, innodb_lock_wait_timeout 값을 적절히 사용하기도 함

### How to Minimize and Handle Deadlocks

---

- 트랜잭션 단위를 작게 하고, 지속시간을 짧게 가지게 하기
- 트랜잭션의 커밋을 즉시하고, 커밋되지 않은 트랜잭션이 계속 세션을 열고 있는 일이 있도록 하지 않기
- Locking read를 사용한다면, 낮은 격리 수준의 레벨을 사용하기(READ_COMMITTED)
- 쿼리가 더 적은 수의 인덱스 레코드를 스캔할 수 있도록, 인덱스를 사용
- SELECT 이전 스냅샷에서 데이터를 반환하도록 허용하는 경우, FOR UPDATE, FOR SHARE를 남용하지 말 것
- 어떠한 방법도 도움이 되지 않는다면, 트랜잭션들을 Table-level ㅣock을 사용
  - LOCK TABLES은 autocommit을 비활성화하고, 외부적으로 트랜잭션이 커밋될떄 까지 UNLOCK TABLES를 발생시키지 않음
  - 단, 테이블의 교착 상태는 방지할 수 있으나 시스템의 성능은 떨어질 수 있음


## Reference

- [Mysql 8.0](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)