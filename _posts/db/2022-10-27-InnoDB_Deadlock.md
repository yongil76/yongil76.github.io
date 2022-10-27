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









## Reference

- [Mysql 8.0](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)