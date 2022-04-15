---
title:  "Oracle DB Partition"
excerpt: "오라클 DB 파티션"

categories:
- DB
  
tags:
- Partition

toc: true

toc_sticky: true

toc_label: "Index"

last_modified_at: 2022-04-09T15:46:00-05:00

---

### Partition?

---

> Oracle Partitioning allows tables and indexes to be partitioned into smaller, more manageable units, providing database administrators with the ability to pursue a "divide and conquer" approach to data management. With partitioning, maintenance operations can be focused on particular portions of tables.

> 오라클에서 테이블과 인덱스가 나눠지도록 하여, 좀 더 효율적인 관리(Divide and conquer)가 가능하도록 지원하는 기능

- 파티션 테이블 확인

~~~sql
SELECT * FROM ALL_TAB_PARTITIONS WHERE TABLE_NAME = 'TARGET';

SELECT * FROM TARGET PARTITION(PARTITION_NAME);
~~~

### Partition type?

---

- 범위 파티셔닝
  ~~~sql
  PARTITION BY RANGE(TARGET_SEQ)(
    PARTITION "TARGET_202201" VALUES LESS THAN ('2022020000000000000'),
    PARTITION "TARGET_202202" VALUES LESS THAN ('2022030000000000000'),
    PARTITION "TARGET_202203" VALUES LESS THAN ('2022040000000000000')
  );
  ~~~
  

- 해시 파티셔닝
  ~~~sql
  PARTITION BY HASH(TARGET_TYPE_CD)(
    PARTITION TYPE1,
    PARTITION TYPE2,
    PARTITION TYPE3,
    PARTITION TYPE4
  );
  ~~~
  - 어떤 파티션에 들어갈지 해쉬값에 의해 결정되기 때문에, 데이터에 대한 보존 주기 등의 관리가 어려움
  

- 리스트 파티셔닝
  ~~~sql
  PARTITION BY RANGE(TARGET_TYPE_CD)(
    PARTITION "TARGET_TYPE_CD" VALUE ('TYPE1'),
    PARTITION "TARGET_TYPE_CD" VALUE ('TYPE2'),
    PARTITION "TARGET_TYPE_CD" VALUE ('TYPE3')
  );
  ~~~
  - 특정 컬럼을 기준으로 정렬
  - TARGET_TYPE_CD가 골고르게 분포될 경우에 적절
  

- 인터벌 파티셔닝 : 범위 파티셔닝과 흡사하나 스스로 새로운 파티션을 생성
- 참조 파티셔닝 : 부모/자식 관계에서 부모 테이블의 파티셔닝을 상속하여 진행
- 복합 파티셔닝 : 위 파티셔닝을 복합적으로 사용

### Partition Index?

---
#### Partition Index / Non Partition Index
- Partition Index 조회
  ~~~sql
  SELECT * FROM dba_part_indexes WHERE table_name = 'TARGET';
  ~~~
- Partition Index : 파티션과 관련된 인덱스
- Non Partition Index : 파티션과 무관한 인덱스

#### Local Index
- Local Prefixed Index
  - 인덱스의 선두 컬럼과 파티션의 컬럼이 일치
- Local Non-Prefixed Index
  - 인덱스의 선두 컬럼이 파티션의 컬럼과 불일치

- 성능 비교(Prefixed vs Non-Prefixed)
> It is more expensive to probe into a nonprefixed index than to probe into a prefixed index. If an index is prefixed (either local or global) and Oracle is presented with a predicate involving the index columns, then partition pruning can restrict application of the predicate to a subset of the index partitions. 

#### Global Index
- Global Prefixed Index
  - 기생성된 파티션 테이블과 무관하게 파티션 인덱스를 생성시키고 싶을 때 사용
  - 로컬 인덱스로 해결이 안되는 상황에서만 사용
  
### Experience?

---

#### Index hint with partition table
> target_seq를 기준으로 범위 파티셔닝이 되어 있는 상황으로 가정, target_ymdt와 target_seq는 순서에 연관 관계가 없음

~~~sql
SELECT *
FROM (
       SELECT /*+INDEX_DESC ( t IDX_TARGET )*/
         t.target_seq,
         t.target_ymdt
       FROM target t
       WHERE t.target_ymdt >= TO_DATE( '2022-01-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS')
         AND t.target_ymdt <= TO_DATE( '2022-01-01 23:59:59', 'YYYY-MM-DD HH24:MI:SS')
     )
~~~
~~~sql
SELECT /*+INDEX_DESC ( t IDX_TARGET )*/
  t.target_seq,
  t.target_ymdt
FROM target t
WHERE t.target_ymdt >= TO_DATE( '2022-01-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS')
  AND t.target_ymdt <= TO_DATE( '2022-01-01 23:59:59', 'YYYY-MM-DD HH24:MI:SS')
ORDER BY t.target_ymdt
~~~

> IDX_TARGET가 파티션 인덱스가 아니라면, 두 쿼리의 결과는 같을 것이고 전자의 쿼리가 인덱스를 타므로 더 좋은 쿼리라고 생각할 수 있다. 하지만 파티션 인덱스이기 때문에, 후자를 사용해야 원하는 결과를 얻을 수 있다.

> 전자의 쿼리 결과는 가장 후행에 위치한 파티션부터 Local Index가 스캔된다.


### Reference

--- 

- [파티션 인덱스 - 구루비](http://www.gurubee.net/lecture/1914)
- [Oracle Reference](https://docs.oracle.com/database/121/VLDBG/GUID-A43726D5-300D-4F5E-8FF3-85F057BC4CD3.htm#VLDBG1263)

