- https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks-handling.html

### 교착상태 감지
  - 교착 상태 감지가 활성화된 경우(기본값), InnoDB는 자동으로 트랜잭션 교착 상태를 감지하고 트랜잭션을 롤백하여 교착 상태를 해제합니다. 
  - InnoDB 는 롤백할 작은 트랜잭션을 선택하려고 하며, 트랜잭션의 크기는 삽입, 업데이트 또는 삭제된 행 수에 따라 결정됩니다.

### 교착상태 최소화
- InnoDB는 자동 행 수준 잠금을 사용합니다. 
  - 단일 행을 삽입하거나 삭제하는 트랜잭션의 경우에도 교착 상태가 발생할 수 있습니다. 
  - 이러한 작업은 실제로 "원자적"이지 않기 때문입니다. 삽입되거나 삭제된 행의 (아마도 여러 개의) 인덱스 레코드에 자동으로 잠금을 설정합니다.
- 교착상태 발생 시 대처 방법
  - SHOW ENGINE INNODB STATUS 를 실행하여 가장 최근의 교착 상태의 원인을 확인합니다.
  - 교착 상태 경고가 자주 발생하면 innodb_print_all_deadlocks 변수를 활성화하여 보다 광범위한 디버깅 정보를 수집합니다. 
    - 최신 교착 상태뿐만 아니라 각 교착 상태에 대한 정보가 MySQL 오류 로그에 기록됩니다. 
    - 디버깅이 끝나면 이 옵션을 비활성화합니다.
  - 교착 상태로 인해 실패할 경우 항상 트랜잭션을 다시 발행할 준비를 하십시오. 
    - 교착 상태는 위험하지 않습니다. 
    - 다시 시도하면 됩니다.
  - 충돌 가능성을 줄이기 위해 트랜잭션을 작고 짧게 유지하십시오.
  - 충돌 가능성을 줄이기 위해 관련 변경 사항을 적용한 후 즉시 트랜잭션을 커밋하십시오. 
    - 특히 커밋되지 않은 트랜잭션이 있는 대화형 MySQL 세션을 오랫동안 열어 두지 마십시오.
  - 잠금 읽기(SELECT ... FOR UPDATE 또는 SELECT ... FOR SHARE)를 사용하는 경우 READ COMMITTED 와 같은 낮은 격리 수준을 사용해 보십시오.
  - 트랜잭션 내에서 여러 테이블을 수정하거나 동일한 테이블의 여러 행 세트를 수정할 때는 매번 일관된 순서로 해당 작업을 수행하십시오. 
    - 그러면 트랜잭션이 잘 정의된 대기열을 형성하고 교착 상태가 되지 않습니다.
    - 예를 들어, 여러 개의 유사한 INSERT, UPDATE 및 DELETE 명령문 시퀀스를 다른 위치에 코딩하는 대신 데이터베이스 작업을 애플리케이션 내의 함수로 구성하거나 저장된 루틴을 호출합니다.
  - 테이블에 잘 선택된 인덱스를 추가하여 쿼리가 더 적은 인덱스 레코드를 스캔하고 더 적은 잠금을 설정합니다. 
    - EXPLAIN SELECT 를 사용하여 MySQL 서버가 쿼리에 가장 적합하다고 생각하는 인덱스를 확인합니다.
  - 잠금을 덜 사용합니다. 
    - SELECT가 이전 스냅샷에서 데이터를 반환하도록 허용할 수 있다면 FOR UPDATE 또는 FOR SHARE 절을 추가하지 마세요. 
    - READ COMMITTED 격리 수준을 사용하는 것이 좋습니다.
    - 동일한 트랜잭션 내의 각 일관된 읽기가 자체 최신 스냅샷에서 읽기 때문입니다.
  - 다른 방법이 도움이 되지 않으면 테이블 수준 잠금으로 트랜잭션을 직렬화합니다. 
    - InnoDB 테이블과 같은 트랜잭션 테이블에서 LOCK TABLES를 사용하는 올바른 방법은 SET autocommit = 0(START TRANSACTION이 아님)으로 트랜잭션을 시작한 다음 LOCK TABLES 를 사용하고 트랜잭션을 명시적으로 커밋할 때까지 UNLOCK TABLES 를 호출하지 않는 것입니다. 
    - 예를 들어, 테이블 t1에 쓰고 테이블 t2에서 읽어야 하는 경우 다음과 같이 할 수 있습니다.
        - SET autocommit=0;
          LOCK TABLES t1 WRITE, t2 READ, ...;
          ... 여기서 테이블 t1과 t2에 대해 무언가를 합니다...
          COMMIT;
          UNLOCK TABLES; 
    - 테이블 수준 잠금은 테이블에 대한 동시 업데이트를 방지하여 바쁜 시스템의 응답성이 떨어지는 대신 교착 상태를 방지합니다.
  - 트랜잭션을 직렬화하는 또 다른 방법은 단일 행만 포함하는 보조 "세마포어" 테이블을 만드는 것입니다. 
    - 각 트랜잭션이 다른 테이블에 액세스하기 전에 해당 행을 업데이트하도록 합니다. 
    - 이런 방식으로 모든 트랜잭션이 직렬 방식으로 발생합니다. 
    - InnoDB 인스턴트 데드락 감지 알고리즘도 이 경우에 작동하는데, 직렬화 잠금이 행 수준 잠금이기 때문입니다. 
    - MySQL 테이블 수준 잠금의 경우, 데드락을 해결하려면 타임아웃 방법을 사용해야 합니다.


### [트랜잭션 deadlock] select for update 시 없는 record 에 대한 테스트

- 테스트1) select for update / select for update / insert / insert
  - A 트랜잭션 : select for update - IX table / X_GAP record
  - B 트랜잭션 : select for update - IX table / X_GAP record
  - A 트랜잭션 : insert - IX table / X_GAP, INSERT_INTENTION record ->>>> INSERT_INTENTION 이 A의 X_GAP과 충돌
  - B 트랜잭션 : insert - IX table / X_GAP, INSERT_INTENTION record->>>>> INSERT_INTENTION 이 B의 X_GAP과 충돌
- 테스트2) insert / insert
  - A 트랜잭션 : insert - IX table / X_REC_NOT_GAP record
  - B 트랜잭션 : insert - IX table / S_REC_NOT_GAP record ->>>>> A 의 X락 획득을 위해 대기
- 테스트3) select for update / insert
  - A 트랜잭션 : select for update - IX table / X_GAP record
  - B 트랜잭션 : insert - IX table / X_GAP, INSERT_INTENTION record ->>>> INSERT_INTENTION 이 A의 X_GAP과 충돌
- 테스트4) update / update ->>> record 없어서 문제 없음
  - A 트랜잭션 : update - IX table / X_GAP record
  - B 트랜잭션 : update - IX table / X_GAP record
- 요약
  - https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html
  - X_GAP <-> X_GAP 은 Compatible
  - X_GAP <-> INSERT_INTENTION 은 Conflict
  - Insert - insert 는 한개 이상 insert 앞에 있으면 뒤에는 S_REC_NOT_GAP 이 잡힌다. (innodb)
  - INSERT_INTENTION 은 베타적 잠금(X락)을 얻기 위해 기다린다.
  - 레코드가 없을때 select for update, update, delete 는 X_GAP 을 얻는다. (X_REC_NOT_GAP 은 레코드가 있을때만 잡힌다)

```SQL
--- 8.0
SELECT * FROM performance_schema.data_locks
ORDER BY ENGINE_TRANSACTION_ID, THREAD_ID
;

--- 5.7
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS
;
```