# SQL 최적화 완전 가이드

> **DAP/SQLP 시험 대비 | Oracle 중심 | 원리부터 실전까지**

---

## 1. SQL 최적화란?

SQL 최적화는 **동일한 결과를 더 빠르고 적은 자원으로 얻도록** SQL과 실행 환경을 개선하는 과정이다.

### 1.1 최적화의 3대 축

```
SQL 최적화
├── 1. SQL 튜닝        → SQL 문장 자체를 개선 (작성 방식, 힌트)
├── 2. 인덱스 최적화   → 인덱스 설계 및 관리
└── 3. 실행 계획 제어  → 옵티마이저 동작 이해 및 조정
```

### 1.2 SQL 처리 과정

```
┌──────────────────────────────────────────────────────────────────┐
│                       SQL 처리 5단계                              │
│                                                                  │
│  1. 파싱(Parsing)                                                │
│     └─ SQL 문법 검사 → 객체 존재 확인 → 권한 확인               │
│         ├─ Hard Parsing: 라이브러리 캐시 미스 → 실행 계획 새로 생성 │
│         └─ Soft Parsing: 캐시 히트 → 기존 실행 계획 재사용      │
│                                                                  │
│  2. 최적화(Optimization)                                         │
│     └─ 옵티마이저가 통계 정보 기반으로 최적 실행 계획 수립       │
│         ├─ Query Transformer   : SQL 변환 (서브쿼리 → 조인 등)  │
│         ├─ Plan Generator      : 여러 실행 계획 후보 생성        │
│         └─ Cost Estimator      : 각 계획의 비용 추정 → 최솟값 선택│
│                                                                  │
│  3. Row Source 생성                                              │
│     └─ 실행 계획을 실제 실행 가능한 코드(Row Source Tree)로 변환 │
│                                                                  │
│  4. 실행(Execute)                                                │
│     └─ Row Source Tree 실행, 데이터 읽기                        │
│                                                                  │
│  5. Fetch                                                        │
│     └─ 결과를 클라이언트로 전송                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 옵티마이저(Optimizer) 이해

### 2.1 RBO vs CBO

| 구분 | RBO (Rule-Based) | CBO (Cost-Based) |
|------|-----------------|-----------------|
| 기준 | 사전 정의된 우선순위 규칙 | 통계 정보 기반 비용 계산 |
| 통계 | 불필요 | **필수** |
| 정확도 | 낮음 | 높음 |
| 현황 | **Deprecated (Oracle 10g+)** | 현재 표준 |
| 힌트 강제 | `RULE` | `ALL_ROWS`, `FIRST_ROWS` |

### 2.2 CBO 비용 계산 요소

```
비용(Cost) = I/O Cost + CPU Cost + Memory Cost

I/O Cost    ← 블록 읽기 횟수 (Single Block / Multi Block)
CPU Cost    ← 정렬, 해시, 필터 연산량
Memory Cost ← Sort Area, Hash Area 사용량
```

### 2.3 통계 정보의 중요성

옵티마이저가 잘못된 계획을 세우는 **근본 원인의 90%는 통계 부재 또는 부정확한 통계**다.

```sql
-- 테이블 통계 수집
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname   => 'SCOTT',
    tabname   => 'EMP',
    cascade   => TRUE,         -- 인덱스 통계도 함께 수집
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE
  );
END;
/

-- 통계 확인
SELECT table_name, num_rows, blocks, last_analyzed
FROM dba_tables
WHERE owner = 'SCOTT';
```

---

## 3. 실행 계획(Execution Plan) 분석

### 3.1 실행 계획 확인 방법

```sql
-- 방법 1: EXPLAIN PLAN (예상 실행 계획)
EXPLAIN PLAN FOR
SELECT e.ename, d.dname FROM emp e, dept d WHERE e.deptno = d.deptno;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'ALL'));

-- 방법 2: 실제 실행 계획 (실행 후 확인)
SELECT /*+ GATHER_PLAN_STATISTICS */ e.ename, d.dname
FROM emp e, dept d WHERE e.deptno = d.deptno;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));

-- 방법 3: AutoTrace (SQL*Plus)
SET AUTOTRACE ON EXPLAIN STATISTICS;
```

### 3.2 실행 계획 읽는 법

```
실행 계획은 안쪽(들여쓰기 깊은 것)부터, 같은 레벨은 위에서 아래로 읽는다.

Id | Operation             | Name          | Rows | Bytes | Cost
--------------------------------------------------------------------
 0 | SELECT STATEMENT      |               |      |       |   5
 1 |  NESTED LOOPS         |               |   14 |   532 |   5   ← 마지막 실행
 2 |   TABLE ACCESS FULL   | DEPT          |    4 |    80 |   3   ← 1번째 실행
 3 |   TABLE ACCESS BY ROWID| EMP          |    4 |   72  |   1   ← 3번째 실행
 4 |    INDEX RANGE SCAN   | EMP_DEPT_IDX  |    4 |       |   0   ← 2번째 실행

읽기 순서: 2(DEPT FTS) → 4(인덱스 스캔) → 3(테이블 접근) → 1(NL 조인) → 0(반환)
```

### 3.3 실행 계획 주요 지표

| 지표 | 의미 | 해석 |
|------|------|------|
| **Rows (E-Rows)** | 예상 행 수 | 실제(A-Rows)와 크게 차이나면 통계 문제 |
| **Cost** | 예상 비용 | 상대적 수치, 낮을수록 좋음 |
| **Bytes** | 예상 처리 데이터 크기 | 메모리/I/O 부하 추정 |
| **A-Rows** | 실제 반환된 행 수 | E-Rows와 비교하여 통계 품질 판단 |
| **A-Time** | 실제 수행 시간 | 병목 구간 찾기 |
| **Buffers** | 실제 블록 I/O 수 | 클수록 비효율 |

---

## 4. 인덱스 최적화

### 4.1 인덱스 종류와 특징

| 인덱스 종류 | 구조 | 적합한 경우 | 주의사항 |
|------------|------|------------|----------|
| **B-Tree 인덱스** | 균형 이진 트리 | 일반적인 범위/동등 조회 | 기본값, 가장 범용적 |
| **비트맵 인덱스** | 비트 배열 | Cardinality 낮은 컬럼 (성별, 지역) | DML 시 Lock 범위 넓음 |
| **함수 기반 인덱스** | 표현식 결과 인덱싱 | `UPPER(col)`, 날짜 가공 조건 | 동일 표현식 사용 필수 |
| **복합 인덱스** | 다중 컬럼 B-Tree | 다중 조건 조회 | 선두 컬럼 선택 중요 |
| **클러스터 인덱스** | 데이터와 함께 정렬 | 범위 조회 많은 테이블 | 구조 변경 부담 |
| **역순(Reverse) 인덱스** | 키 바이트 역전 | RAC 환경 인덱스 경합 감소 | Range Scan 불가 |
| **파티션 인덱스** | 파티션별 인덱스 | 대용량 파티션 테이블 | 로컬/글로벌 구분 |

### 4.2 인덱스 스캔 방식 비교

```
【Index Range Scan】
  Root → Branch → Leaf(시작) → Leaf(끝) → Table(ROWID)
  ✅ 범위 조건 (BETWEEN, >=, LIKE 'A%')
  ✅ 동등 조건 (=)
  ❌ 대량 데이터 → FTS가 유리할 수 있음

【Index Full Scan】
  Root → Branch → Leaf 전체 순차 탐색 → Table
  ✅ 인덱스 컬럼 ORDER BY 시 Sort 회피
  ✅ MIN/MAX 빠른 조회
  ❌ 대량 데이터 → INDEX_FFS가 유리

【Index Fast Full Scan】
  인덱스 전체를 Multiblock I/O로 읽기 (순서 무보장)
  ✅ COUNT(*), SUM(인덱스컬럼) 등 인덱스만으로 해결 가능 시
  ✅ 대용량 인덱스 스캔
  ❌ 결과 정렬 보장 안 됨

【Index Skip Scan】
  선두 컬럼 Distinct값마다 나머지 컬럼으로 Range Scan
  ✅ 복합 인덱스에서 선두 컬럼 조건 없을 때
  ✅ 선두 컬럼 Distinct 값이 매우 적을 때 (예: 성별)
  ❌ 선두 컬럼 Distinct가 많으면 비효율
```

### 4.3 인덱스가 사용되지 않는 경우 (핵심!)

```sql
-- ❌ 인덱스 컬럼에 함수/연산 적용
WHERE SUBSTR(emp_no, 1, 4) = '2024'   -- 함수 기반 인덱스로 해결
WHERE sal * 12 > 60000000              -- WHERE sal > 5000000 으로 변경

-- ❌ 묵시적 형변환
WHERE emp_no = 20240001  -- emp_no가 VARCHAR2일 때 → 숫자로 형변환 발생
                          -- WHERE emp_no = '20240001' 로 변경

-- ❌ LIKE 패턴의 % 앞위치
WHERE ename LIKE '%KIM'   -- 전방 와일드카드 → FTS 발생
WHERE ename LIKE '%KIM%'  -- FTS 발생 (Full-Text Index 고려)
WHERE ename LIKE 'KIM%'   -- ✅ Range Scan 가능

-- ❌ IS NULL / IS NOT NULL (일반적으로)
WHERE col IS NULL          -- B-Tree 인덱스는 NULL 미저장 → FTS
                           -- Bitmap Index는 NULL도 저장

-- ❌ NOT 조건 (부정 조건)
WHERE deptno != 10         -- NOT 조건 → FTS (OR로 재작성 고려)
WHERE deptno NOT IN (10, 20)

-- ❌ OR 조건 (일부 경우)
WHERE deptno = 10 OR job = 'CLERK'  -- 각각 인덱스 있어도 비효율 가능
                                     -- UNION ALL로 분리 고려
```

### 4.4 복합 인덱스 설계 원칙

```
복합 인덱스 컬럼 순서 결정 기준:
  1. = 조건 컬럼을 범위 조건 컬럼보다 앞에
  2. Cardinality(선택도)가 높은 컬럼을 앞에
  3. 자주 사용되는 조건 조합 고려

예시: WHERE region = 'SEOUL' AND age BETWEEN 20 AND 30
  ✅ IDX(region, age)  → region으로 좁히고 age 범위 스캔
  ❌ IDX(age, region)  → age 범위 후 region 필터 (비효율)
```

---

## 5. 조인 최적화

### 5.1 조인 방식 선택 기준

| 상황 | 권장 조인 | 이유 |
|------|-----------|------|
| 소량 데이터 + 인덱스 있음 | **Nested Loop** | 인덱스 활용, 빠른 첫 행 |
| 대용량 + 동등 조인 | **Hash Join** | 정렬 없이 해시로 빠른 매칭 |
| 대용량 + 이미 정렬됨 | **Sort Merge** | 정렬 비용 절약 |
| 비동등 조인 | **NL 또는 Sort Merge** | Hash는 동등 조인만 가능 |
| 카티션 곱 의도적 필요 | **Cartesian Join** | Cross Join 명시 |

### 5.2 조인 순서 최적화

```
NL Join 성능 공식:
  총 처리량 = 드라이빙 테이블 건수 × (Inner 테이블 블록 I/O)

드라이빙 테이블 선택 원칙:
  1. WHERE 조건에 의해 가장 많이 줄어드는 테이블
  2. 결과 건수가 가장 적은 테이블
  3. Inner 테이블에 효율적인 인덱스 존재해야 함
```

```sql
-- 비효율: emp(14건)을 드라이빙하고 orders(100만건) 조회
SELECT /*+ LEADING(e o) USE_NL(o) */ e.ename, o.order_no
FROM emp e, orders o
WHERE e.empno = o.empno AND e.deptno = 10;

-- 최적: orders에 empno 인덱스 있을 때, emp(조건 후 소량)를 드라이빙
-- ✅ dept=10 필터 후 emp 4건 → orders를 4번만 인덱스 스캔
```

### 5.3 서브쿼리 vs 조인 변환

```sql
-- ❌ 비효율적인 서브쿼리 (반복 실행 가능성)
SELECT ename, sal
FROM emp
WHERE sal > (SELECT AVG(sal) FROM emp WHERE deptno = e.deptno);
-- → 상관 서브쿼리: 메인 쿼리 행마다 서브쿼리 실행

-- ✅ 인라인 뷰로 변환 (한 번만 집계)
SELECT e.ename, e.sal
FROM emp e,
     (SELECT deptno, AVG(sal) avg_sal FROM emp GROUP BY deptno) a
WHERE e.deptno = a.deptno
  AND e.sal > a.avg_sal;

-- ✅ 윈도우 함수 활용 (Oracle 분석 함수)
SELECT ename, sal
FROM (
    SELECT ename, sal,
           AVG(sal) OVER (PARTITION BY deptno) avg_sal
    FROM emp
)
WHERE sal > avg_sal;
```

---

## 6. SQL 작성 최적화 기법

### 6.1 SELECT 절 최적화

```sql
-- ❌ SELECT * (불필요한 컬럼 포함)
SELECT * FROM emp WHERE deptno = 10;

-- ✅ 필요한 컬럼만 (I/O 및 네트워크 절약)
SELECT empno, ename, sal FROM emp WHERE deptno = 10;

-- ✅ 인덱스 커버링: 인덱스 컬럼만 조회 시 Table Access 제거
-- IDX(deptno, empno, sal) 인덱스 존재 시
SELECT /*+ INDEX(e emp_dept_sal_idx) */ empno, ename, sal
FROM emp e WHERE deptno = 10;
-- → Table Access 없이 인덱스만으로 처리 (Covering Index)
```

### 6.2 WHERE 절 최적화

```sql
-- ❌ 불필요한 형변환 유발
WHERE TO_CHAR(order_date, 'YYYY') = '2024'

-- ✅ 범위 조건으로 변환 (인덱스 사용 가능)
WHERE order_date >= DATE '2024-01-01'
  AND order_date <  DATE '2025-01-01'

-- ❌ OR 조건 (인덱스 활용 저해)
WHERE status = 'A' OR status = 'B'

-- ✅ IN 절로 변환
WHERE status IN ('A', 'B')

-- ✅ UNION ALL로 분리 (컬럼이 다를 경우)
SELECT * FROM orders WHERE status = 'A'
UNION ALL
SELECT * FROM orders WHERE status = 'B'

-- ❌ NULL 비교 오류
WHERE col = NULL   -- 항상 FALSE! NULL 비교는 IS NULL

-- ✅ NVL/COALESCE 활용 (단, 인덱스 사용 불가 주의)
WHERE NVL(col, 'X') = 'X'   -- FTS 발생
WHERE col IS NULL             -- ✅
```

### 6.3 DISTINCT와 GROUP BY

```sql
-- ❌ 불필요한 DISTINCT (중복 없는 컬럼에 사용)
SELECT DISTINCT empno, ename FROM emp;  -- empno가 PK면 DISTINCT 불필요

-- EXISTS로 대체 가능한 DISTINCT
-- ❌
SELECT DISTINCT d.dname
FROM dept d, emp e
WHERE d.deptno = e.deptno;

-- ✅ 더 효율적: EXISTS 사용
SELECT dname FROM dept d
WHERE EXISTS (
    SELECT 1 FROM emp e WHERE e.deptno = d.deptno
);

-- GROUP BY 최적화: 필터 조건은 HAVING보다 WHERE에서 처리
-- ❌ HAVING으로 필터 (집계 후 필터)
SELECT deptno, SUM(sal)
FROM emp
GROUP BY deptno
HAVING deptno IN (10, 20);

-- ✅ WHERE에서 먼저 필터 (집계 전 데이터 축소)
SELECT deptno, SUM(sal)
FROM emp
WHERE deptno IN (10, 20)
GROUP BY deptno;
```

### 6.4 UNION vs UNION ALL

```sql
-- UNION: 중복 제거를 위한 Sort 발생 → 비용 높음
SELECT deptno FROM emp
UNION
SELECT deptno FROM dept;

-- UNION ALL: Sort 없음 → 빠름 (중복 허용)
-- 중복이 없거나 중복을 허용하는 경우 반드시 UNION ALL 사용
SELECT deptno FROM emp WHERE job = 'CLERK'
UNION ALL
SELECT deptno FROM emp WHERE job = 'MANAGER';
-- ✅ job 조건이 다르므로 중복 없음 → UNION ALL 사용 가능
```

### 6.5 페이징 처리 최적화

```sql
-- ❌ 비효율적인 페이징 (전체 정렬 후 잘라냄)
SELECT *
FROM (SELECT * FROM orders ORDER BY order_date DESC)
WHERE ROWNUM <= 20;

-- ✅ 효율적인 페이징 (인덱스 활용)
-- IDX(order_date DESC) 인덱스 존재 시
SELECT /*+ INDEX_DESC(o ord_date_idx) */ *
FROM orders o
WHERE ROWNUM <= 20;  -- 정렬 없이 인덱스 역순으로 20건만 읽음

-- ✅ 중간 페이지 조회 (rownum 이중 중첩)
SELECT *
FROM (
    SELECT ROWNUM rn, a.*
    FROM (
        SELECT order_no, order_date, amount
        FROM orders
        ORDER BY order_date DESC
    ) a
    WHERE ROWNUM <= 40   -- 마지막 행
)
WHERE rn >= 21;           -- 시작 행
```

---

## 7. 소트(Sort) 최적화

### 7.1 소트 발생 원인 및 회피

| SQL 구문 | 소트 발생 | 회피 방법 |
|----------|-----------|-----------|
| `ORDER BY` | 정렬 | 인덱스로 정렬 대체 |
| `GROUP BY` | 집계용 정렬 | 인덱스 기반 스트림 집계 |
| `DISTINCT` | 중복 제거 정렬 | EXISTS, UNION ALL로 대체 |
| `UNION` | 중복 제거 | UNION ALL로 대체 |
| `MINUS` | 차집합 정렬 | NOT EXISTS로 대체 가능 |
| `INTERSECT` | 교집합 정렬 | EXISTS로 대체 가능 |
| `Sort Merge Join` | 조인 정렬 | Hash Join 또는 NL Join으로 대체 |

```sql
-- ❌ ORDER BY 정렬 발생
SELECT ename, sal FROM emp
WHERE deptno = 10
ORDER BY sal;

-- ✅ IDX(deptno, sal)이 있다면 인덱스 순서로 정렬 완료 → Sort 생략
SELECT /*+ INDEX(e emp_dept_sal_idx) */ ename, sal
FROM emp e
WHERE deptno = 10
ORDER BY sal;  -- 인덱스가 이미 sal 순으로 정렬되어 있음
```

### 7.2 소트 영역 설정

```sql
-- Sort 작업이 메모리 초과 시 Temp 테이블스페이스(Disk Sort) 발생
-- → 성능 급격히 저하

-- 세션 레벨 Sort Area 조정
ALTER SESSION SET SORT_AREA_SIZE = 104857600; -- 100MB

-- PGA_AGGREGATE_TARGET으로 자동 관리 (권장)
-- → 개별 Sort/Hash 영역을 Oracle이 자동 조정
ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 2G;
```

---

## 8. 분석 함수(Window Function) 활용 최적화

분석 함수는 **집계와 개별 행을 동시에 처리**하여 Self-Join이나 서브쿼리를 제거한다.

```sql
-- ❌ 부서별 평균 급여와 개인 급여 비교 (서브쿼리 방식)
SELECT e.ename, e.sal,
       (SELECT AVG(sal) FROM emp WHERE deptno = e.deptno) avg_sal
FROM emp e;

-- ✅ 분석 함수 방식 (한 번의 스캔으로 처리)
SELECT ename, sal,
       AVG(sal) OVER (PARTITION BY deptno) avg_sal,
       RANK()   OVER (PARTITION BY deptno ORDER BY sal DESC) rank_in_dept,
       SUM(sal) OVER () total_sal
FROM emp;

-- ❌ 연속 순위 처리 (Self Join)
SELECT a.ename, a.sal, COUNT(*) rank
FROM emp a, emp b
WHERE a.sal <= b.sal
GROUP BY a.ename, a.sal;

-- ✅ RANK() / DENSE_RANK() 활용
SELECT ename, sal,
       RANK()       OVER (ORDER BY sal DESC) rank,    -- 동순위 시 건너뜀
       DENSE_RANK() OVER (ORDER BY sal DESC) d_rank,  -- 동순위 시 연속
       ROW_NUMBER() OVER (ORDER BY sal DESC) row_no   -- 고유 번호
FROM emp;
```

### 8.1 주요 분석 함수 목록

| 함수 | 설명 | 활용 예 |
|------|------|---------|
| `ROW_NUMBER()` | 고유 순번 부여 | 중복 제거, 페이징 |
| `RANK()` | 동순위 시 건너뜀 (1,2,2,4) | 순위 표시 |
| `DENSE_RANK()` | 동순위 시 연속 (1,2,2,3) | 메달 순위 |
| `LAG(col, n)` | n행 이전 값 | 전일 대비, 증감 |
| `LEAD(col, n)` | n행 이후 값 | 다음 이벤트 날짜 |
| `SUM() OVER()` | 누적/파티션 합계 | 누적 매출, 비율 |
| `AVG() OVER()` | 이동 평균 | 이동 평균 계산 |
| `FIRST_VALUE()` | 파티션 내 첫 번째 값 | 최초 주문일 |
| `LAST_VALUE()` | 파티션 내 마지막 값 | 최근 주문일 |
| `NTILE(n)` | n개 그룹으로 분할 | 분위수 |

---

## 9. DML 최적화

### 9.1 대용량 INSERT 최적화

```sql
-- ❌ 일반 INSERT (Buffer Cache 경유, UNDO 생성, 느림)
INSERT INTO target_table SELECT * FROM source_table;

-- ✅ Direct Path INSERT (Buffer 우회, HWM 이후 직접 기록)
INSERT /*+ APPEND */ INTO target_table
SELECT * FROM source_table;
COMMIT;  -- 필수! 없으면 다음 DML 블락

-- ✅ 병렬 Direct Path INSERT
INSERT /*+ APPEND PARALLEL(t 4) */ INTO target_table t
SELECT /*+ PARALLEL(s 4) */ * FROM source_table s;
COMMIT;

-- ✅ CTAS (Create Table As Select) - 가장 빠름
CREATE TABLE target_table /*+ PARALLEL(4) */
AS SELECT * FROM source_table;
```

### 9.2 대용량 UPDATE/DELETE 최적화

```sql
-- ❌ 대용량 UPDATE 한 번에 (Undo/Redo 폭발, Lock 장시간)
UPDATE orders SET status = 'C' WHERE order_year = 2022;

-- ✅ 배치 커밋 (일정 건수마다 커밋)
DECLARE
  CURSOR c IS SELECT order_id FROM orders WHERE order_year = 2022;
  TYPE t IS TABLE OF orders.order_id%TYPE;
  v t;
BEGIN
  OPEN c;
  LOOP
    FETCH c BULK COLLECT INTO v LIMIT 10000;  -- 10000건씩
    EXIT WHEN v.COUNT = 0;
    FORALL i IN 1..v.COUNT
      UPDATE orders SET status = 'C' WHERE order_id = v(i);
    COMMIT;
  END LOOP;
  CLOSE c;
END;
/

-- ✅ DELETE 대신 파티션 DROP (파티션 테이블)
-- DELETE FROM sales WHERE sale_year = 2020;  ← 느림
ALTER TABLE sales DROP PARTITION p_2020;       -- ✅ 순간 처리
```

### 9.3 MERGE문 활용

```sql
-- INSERT + UPDATE를 MERGE로 통합 (두 번 읽기 방지)
MERGE INTO target t
USING source s ON (t.id = s.id)
WHEN MATCHED THEN
    UPDATE SET t.val = s.val, t.upd_dt = SYSDATE
WHEN NOT MATCHED THEN
    INSERT (id, val, ins_dt) VALUES (s.id, s.val, SYSDATE);
```

---

## 10. 파티션(Partition) 활용 최적화

### 10.1 파티션 Pruning

파티션 테이블에서 **필요한 파티션만 접근**하는 최적화

```sql
-- 파티션 키 조건 사용 → Partition Pruning 발생
SELECT SUM(amount)
FROM sales
WHERE sale_date >= DATE '2024-01-01'  -- 파티션 키가 sale_date일 때
  AND sale_date <  DATE '2025-01-01';
-- → 2024년 파티션만 접근, 나머지 파티션 Skip

-- 실행 계획에서 확인: "Pstart" "Pstop"으로 파티션 범위 표시
-- PARTITION RANGE SINGLE   → 1개 파티션
-- PARTITION RANGE ITERATOR → 여러 파티션
-- PARTITION RANGE ALL      → 전체 파티션 (Pruning 안 됨)
```

### 10.2 파티션 전략

| 파티션 유형 | 기준 | 적합한 경우 |
|------------|------|------------|
| **Range** | 값의 범위 | 날짜 기반 이력 데이터 |
| **List** | 특정 값 목록 | 지역, 상태 코드 |
| **Hash** | 해시 함수 | 균등 분산, 특정 기준 없을 때 |
| **Composite** | 복합 (Range-Hash 등) | 대용량 + 고른 분산 |
| **Interval** | Range 자동 생성 | 새 파티션 자동 관리 (Oracle 11g+) |

---

## 11. 바인드 변수(Bind Variable) 활용

### 11.1 리터럴 vs 바인드 변수

```sql
-- ❌ 리터럴 사용: 쿼리마다 Hard Parsing 발생 → 라이브러리 캐시 낭비
SELECT * FROM emp WHERE empno = 7369;
SELECT * FROM emp WHERE empno = 7499;
SELECT * FROM emp WHERE empno = 7521;
-- → 동일한 SQL이 아니므로 3번 Hard Parsing

-- ✅ 바인드 변수: 동일 SQL → Soft Parsing (캐시 재사용)
SELECT * FROM emp WHERE empno = :v_empno;
-- → 한 번만 Parsing, 이후 실행 계획 재사용
```

### 11.2 바인드 변수 미사용의 부작용

- **Library Cache 오염**: 동일 패턴의 쿼리가 각각 다른 커서로 저장
- **Hard Parsing 폭주**: CPU 사용률 급등
- **래치(Latch) 경합**: Library Cache Latch 대기 발생

---

## 12. SQL 최적화 단계별 체크리스트

```
【Step 1: 문제 파악】
  □ 현재 수행 시간 측정 (DBMS_UTILITY.GET_TIME)
  □ 실행 계획 확인 (EXPLAIN PLAN / AUTOTRACE)
  □ 실제 실행 통계 수집 (GATHER_PLAN_STATISTICS)

【Step 2: 병목 분석】
  □ Full Table Scan이 적절한가? (소량 → 인덱스, 대량 → FTS 유리)
  □ E-Rows vs A-Rows 차이 확인 (통계 문제?)
  □ Buffers(블록 I/O)가 불필요하게 큰가?
  □ Sort 연산이 Disk로 내려가는가?

【Step 3: 개선 방안 선택】
  □ 인덱스 생성/변경 검토
  □ SQL 재작성 (서브쿼리 → 조인, UNION → UNION ALL)
  □ 힌트로 실행 계획 제어
  □ 통계 정보 갱신

【Step 4: 검증】
  □ 개선 전후 수행 시간 비교
  □ 전체 부하(Buffers, CPU) 비교
  □ 다른 파라미터 조합에서도 유효한지 확인
  □ 운영 반영 후 모니터링
```

---

## 13. 전체 최적화 기법 요약 테이블

| 구분 | 기법 | 핵심 내용 | 효과 |
|------|------|-----------|------|
| **인덱스** | 복합 인덱스 설계 | = 조건 선두, 선택도 높은 컬럼 앞 | Range Scan 최적화 |
| **인덱스** | 커버링 인덱스 | 조회 컬럼을 인덱스에 포함 | Table Access 제거 |
| **인덱스** | 함수 기반 인덱스 | 가공 컬럼에 인덱스 | 함수 사용 시 인덱스 활용 |
| **조인** | 드라이빙 테이블 최적화 | 선택도 높은 테이블 선행 | NL Join 루프 최소화 |
| **조인** | 조인 방식 선택 | 소량=NL, 대량=Hash | 상황별 최적 조인 |
| **SQL** | 불필요한 형변환 제거 | 타입 일치 조건 사용 | 인덱스 활용 |
| **SQL** | 부정 조건 개선 | NOT IN → NOT EXISTS | 인덱스 활용 |
| **SQL** | UNION ALL 사용 | 중복 없는 경우 UNION ALL | Sort 제거 |
| **SQL** | 분석 함수 활용 | Window Function | Self Join/서브쿼리 제거 |
| **SQL** | 페이징 최적화 | ROWNUM + 인덱스 | 불필요한 Sort 제거 |
| **소트** | 인덱스로 정렬 대체 | ORDER BY 컬럼 인덱스 포함 | Sort 연산 제거 |
| **DML** | APPEND 힌트 | Direct Path Insert | Buffer 우회, 고속 적재 |
| **DML** | MERGE 활용 | Upsert 통합 | 두 번 읽기 방지 |
| **DML** | 배치 커밋 | BULK COLLECT + FORALL | Undo/Lock 최소화 |
| **파티션** | Partition Pruning | 파티션 키 조건 사용 | 불필요 파티션 Skip |
| **파싱** | 바인드 변수 | 리터럴 대신 :변수 | Hard Parsing 방지 |
| **통계** | 통계 정보 갱신 | DBMS_STATS | 옵티마이저 정확도 향상 |
| **병렬** | Parallel 힌트 | DOP 지정 | 대용량 처리 시간 단축 |

---

## 14. DAP/SQLP 시험 핵심 암기 포인트

| 항목 | 핵심 내용 |
|------|-----------|
| Hard vs Soft Parsing | Hard=새 실행 계획 생성, Soft=캐시 재사용 |
| 인덱스 미사용 | 함수 적용, 묵시적 형변환, 앞 % LIKE, IS NULL |
| 조인 방식 | NL=소량/인덱스, Hash=대량/동등, Merge=비동등 |
| Sort 회피 | UNION ALL, 인덱스 정렬, EXISTS 대체 |
| 선두 컬럼 원칙 | = 조건 → 범위 조건 순으로 복합 인덱스 구성 |
| APPEND | Direct Path = HWM 이후, Exclusive Lock, COMMIT 필수 |
| 파티션 Pruning | 파티션 키 WHERE 조건 필수, 함수 사용 금지 |
| 분석 함수 | PARTITION BY(그룹), ORDER BY(정렬), ROWS/RANGE(프레임) |
| 바인드 변수 | Library Cache 재사용, Hard Parsing 방지 |

---
*본 문서는 Oracle 12c 이상 기준으로 작성되었습니다.*
