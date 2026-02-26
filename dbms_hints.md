# DBMS 힌트(Hint) 완전 가이드

> **DAP/SQLP 시험 대비 | Oracle 중심 정리**

---

## 1. 힌트(Hint)란?

### 1.1 개념

**힌트(Hint)** 는 SQL 문장 내에 특수 주석 형태로 삽입하여 **옵티마이저(Optimizer)의 실행 계획을 개발자가 직접 제어**하는 지시어다.

옵티마이저는 통계 정보를 기반으로 최적의 실행 계획을 자동 수립하지만, 다음 상황에서는 개발자가 힌트를 통해 개입해야 한다.

- 통계 정보가 부정확하거나 최신화되지 않은 경우
- 특수한 업무 패턴으로 인해 옵티마이저가 잘못된 계획을 선택하는 경우
- 대용량 배치 처리 등 특정 액세스 패턴이 유리한 경우
- 조인 순서나 방식에 명시적인 제어가 필요한 경우

---

### 1.2 힌트의 동작 원리

```
┌─────────────────────────────────────────────────────────────┐
│                        SQL 처리 흐름                         │
├─────────────────────────────────────────────────────────────┤
│  SQL 파싱                                                    │
│    └─→ 힌트 파싱 (주석 형태이지만 옵티마이저가 별도 인식)    │
│          └─→ 옵티마이저 힌트 해석                            │
│                └─→ 힌트 유효성 검사                          │
│                      ├─[유효] → 힌트 기반 실행 계획 수립    │
│                      └─[무효] → 힌트 무시, 자동 최적화 수행 │
└─────────────────────────────────────────────────────────────┘
```

**핵심 원리:**
1. 힌트는 **명령(Command)이 아닌 권고(Suggestion)** 이지만, Oracle은 대부분 힌트를 강제적으로 따름
2. 잘못된 힌트(오타, 존재하지 않는 인덱스 등)는 **에러 없이 무시**됨
3. 힌트는 **쿼리 블록(Query Block) 단위**로 적용됨
4. 충돌하는 힌트가 여러 개 있을 경우, 일반적으로 **가장 구체적인 힌트**가 우선

---

### 1.3 힌트 작성 문법

```sql
-- 기본 문법 (/*+ */)
SELECT /*+ HINT명 */ 컬럼 FROM 테이블;

-- 여러 힌트 동시 사용 (공백으로 구분)
SELECT /*+ FULL(e) USE_NL(e d) ORDERED */ e.ename, d.dname
FROM emp e, dept d
WHERE e.deptno = d.deptno;

-- 쿼리 블록 지정 힌트
SELECT /*+ FULL(@subq t) */ ...
FROM (SELECT /*+ QB_NAME(subq) */ * FROM emp t) ...;
```

> ⚠️ `--+` 형태의 힌트는 한 줄만 적용되므로, 여러 힌트는 `/*+ */` 방식을 권장

---

## 2. 힌트 분류 체계

```
DBMS 힌트
├── 옵티마이저 목표 힌트        → ALL_ROWS, FIRST_ROWS, RULE
├── 액세스 방식 힌트            → FULL, INDEX, INDEX_FFS, NO_INDEX
├── 조인 순서 힌트              → ORDERED, LEADING
├── 조인 방식 힌트              → USE_NL, USE_HASH, USE_MERGE
├── 서브쿼리 처리 힌트          → UNNEST, NO_UNNEST, PUSH_SUBQ
├── 병렬 처리 힌트              → PARALLEL, NO_PARALLEL
├── 결과 캐싱 힌트              → RESULT_CACHE, NO_RESULT_CACHE
└── 기타 힌트                   → APPEND, DRIVING_SITE, QB_NAME
```

---

## 3. 옵티마이저 목표(Goal) 힌트

옵티마이저의 **최적화 방향** 자체를 변경하는 힌트

| 힌트 | 설명 | 사용 목적 | 적용 대상 |
|------|------|-----------|-----------|
| `ALL_ROWS` | 전체 결과 집합을 최소 비용으로 처리 (비용 기반) | 배치, DW, 집계 쿼리 | 쿼리 전체 |
| `FIRST_ROWS(n)` | 처음 n건을 최대한 빨리 반환 (응답 시간 최적화) | OLTP, 페이지 조회 | 쿼리 전체 |
| `RULE` | 규칙 기반 옵티마이저(RBO) 강제 사용 | 레거시 시스템 호환 | 쿼리 전체 |
| `CHOOSE` | 통계 있으면 CBO, 없으면 RBO (Deprecated) | 구버전 호환 | 쿼리 전체 |

```sql
-- 예시: 처음 10건을 빠르게 가져올 때
SELECT /*+ FIRST_ROWS(10) */ empno, ename
FROM emp
WHERE deptno = 10
ORDER BY sal DESC;
```

**원리:** `FIRST_ROWS`는 정렬이나 집계보다 인덱스 스캔 + 빠른 반환을 선호하여 Nested Loop Join을 유도함

---

## 4. 액세스 방식(Access Method) 힌트

테이블이나 인덱스에 **어떤 방식으로 접근할지** 결정하는 힌트

### 4.1 액세스 방식 힌트 목록

| 힌트 | 설명 | 특징 | 적합한 상황 |
|------|------|------|------------|
| `FULL(table)` | Full Table Scan 강제 | 모든 블록 읽기 | 대부분 데이터 조회, 통계 갱신 후 검증 |
| `INDEX(table index)` | 특정 인덱스 Range Scan 강제 | 선택도 높은 컬럼 | 소량 데이터 조회, 인덱스 컬럼 조건 있을 때 |
| `NO_INDEX(table index)` | 특정 인덱스 사용 금지 | 지정 인덱스 배제 | 잘못된 인덱스 선택 방지 |
| `INDEX_ASC(table index)` | 인덱스 오름차순 스캔 | 기본값과 동일 | ORDER BY ASC와 인덱스 방향 일치 시 |
| `INDEX_DESC(table index)` | 인덱스 내림차순 스캔 | 역방향 스캔 | MAX값 빠른 조회, ORDER BY DESC |
| `INDEX_FFS(table index)` | Index Fast Full Scan | 인덱스 전체를 Multiblock Read | 인덱스 컬럼만 필요한 COUNT, SUM |
| `INDEX_SS(table index)` | Index Skip Scan | 선두 컬럼 조건 없어도 스캔 | 복합 인덱스 선두 컬럼 Distinct가 낮을 때 |
| `INDEX_JOIN(table)` | 인덱스 조인 | 2개 이상 인덱스를 Hash Join | 테이블 액세스 없이 인덱스만으로 처리 가능 |
| `CLUSTER(table)` | Cluster Scan 사용 | 클러스터 테이블 전용 | 클러스터 키 범위 조회 |
| `HASH(table)` | Hash Scan 사용 | Hash 클러스터 전용 | Hash 클러스터 동등 조회 |
| `ROWID(table)` | ROWID Scan 강제 | 직접 블록 접근 | ROWID를 이미 알고 있는 경우 |

### 4.2 인덱스 스캔 방식 비교

```
Full Table Scan:  [블록1]-[블록2]-[블록3]-...-[블록N]  → 전체 읽기
                  ↓ Multiblock I/O로 빠르지만 데이터 많으면 비효율

Index Range Scan: [Root] → [Branch] → [Leaf] → 테이블 접근
                  ↓ 선택도 높을 때 효율적, 랜덤 I/O 발생

Index FFS:        [인덱스 전체를 Multiblock으로 읽기]
                  ↓ 테이블 접근 없음, 인덱스 컬럼만 필요할 때 최적

Index Skip Scan:  [Root] → [선두컬럼 값 변경점마다 Branch 탐색]
                  ↓ 선두 컬럼 없어도 가능하지만 비효율 가능
```

```sql
-- INDEX 힌트 사용 예시
SELECT /*+ INDEX(e emp_deptno_idx) */ empno, ename
FROM emp e
WHERE deptno = 20;

-- INDEX_DESC로 최신 데이터 빠르게 조회
SELECT /*+ INDEX_DESC(o ord_date_idx) */ *
FROM orders o
WHERE order_date >= TRUNC(SYSDATE)
AND ROWNUM <= 10;

-- INDEX_FFS로 카운트 최적화
SELECT /*+ INDEX_FFS(e emp_pk) */ COUNT(*)
FROM emp e;
```

---

## 5. 조인 순서(Join Order) 힌트

**어떤 테이블을 먼저 드라이빙(Driving) 할지** 결정하는 힌트

| 힌트 | 문법 | 설명 | 특징 |
|------|------|------|------|
| `ORDERED` | `ORDERED` | FROM 절의 테이블 순서대로 조인 | 순서 직접 제어, 단순 |
| `LEADING` | `LEADING(t1 t2 t3)` | 지정한 테이블 순서로 조인 (Oracle 10g+) | FROM 절 순서 무관, 권장 방식 |

```sql
-- ORDERED: FROM 절 순서(emp → dept → sal_grade) 그대로 조인
SELECT /*+ ORDERED USE_NL(dept) USE_NL(sg) */
       e.ename, d.dname, sg.grade
FROM emp e, dept d, sal_grade sg
WHERE e.deptno = d.deptno
  AND e.sal BETWEEN sg.losal AND sg.hisal;

-- LEADING: FROM 절과 무관하게 dept → emp 순으로 조인
SELECT /*+ LEADING(d e) USE_NL(e) */
       e.ename, d.dname
FROM emp e, dept d
WHERE e.deptno = d.deptno;
```

**드라이빙 테이블 선택 원칙:**
- 조건에 의해 **가장 많이 걸러지는(선택도 높은) 테이블**을 선행 테이블로
- NL Join에서는 드라이빙 테이블의 건수가 **루프 횟수**가 됨
- 소량 → 대량 순서로 조인하는 것이 일반적으로 유리

---

## 6. 조인 방식(Join Method) 힌트

**어떤 알고리즘으로 조인할지** 결정하는 힌트

### 6.1 조인 방식 힌트 목록

| 힌트 | 문법 | 조인 방식 | 특징 |
|------|------|-----------|------|
| `USE_NL` | `USE_NL(inner_table)` | Nested Loop Join | OLTP, 소량, 인덱스 필수 |
| `USE_HASH` | `USE_HASH(table)` | Hash Join | 대용량, 동등 조인, 메모리 사용 |
| `USE_MERGE` | `USE_MERGE(table)` | Sort Merge Join | 정렬된 대용량, 비동등 조인 가능 |
| `USE_NL_WITH_INDEX` | `USE_NL_WITH_INDEX(t idx)` | NL Join + 인덱스 강제 | NL과 인덱스 동시 제어 |
| `NO_USE_NL` | `NO_USE_NL(table)` | NL Join 금지 | 특정 조인 방식 배제 |
| `NO_USE_HASH` | `NO_USE_HASH(table)` | Hash Join 금지 | 메모리 부족 시 |
| `NO_USE_MERGE` | `NO_USE_MERGE(table)` | Sort Merge Join 금지 | 정렬 부하 방지 |

### 6.2 조인 방식별 동작 원리

```
【Nested Loop Join】
  FOR each row in 드라이빙 테이블
    → Inner 테이블에서 조인 컬럼으로 인덱스 탐색
    → 매칭 행 반환
  특징: 소량 데이터, 인덱스 필수, 빠른 첫 행 응답

【Hash Join】
  1단계: 작은 테이블을 Hash Table로 메모리에 빌드
  2단계: 큰 테이블 읽으며 Hash Table과 매칭
  특징: 대용량, 동등 조인만 가능, 메모리 소비

【Sort Merge Join】
  1단계: 양쪽 테이블 조인 컬럼으로 정렬
  2단계: 정렬된 두 집합을 순차적으로 병합
  특징: 비동등 조인 가능, 이미 정렬된 경우 유리
```

| 비교 항목 | NL Join | Hash Join | Sort Merge Join |
|-----------|---------|-----------|-----------------|
| 조인 조건 | 동등/비동등 | **동등만** | 동등/비동등 |
| 인덱스 필요 | **필수** (Inner) | 불필요 | 불필요 |
| 데이터 크기 | **소량** | **대용량** | 중/대용량 |
| 메모리 사용 | 낮음 | **높음** | 중간 (정렬) |
| 첫 행 응답 | **빠름** | 느림 | 느림 |
| OLTP 적합성 | **높음** | 낮음 | 낮음 |
| DW/배치 적합성 | 낮음 | **높음** | 높음 |

```sql
-- NL Join 강제: 소량 데이터 OLTP
SELECT /*+ LEADING(d) USE_NL(e) INDEX(e emp_deptno_idx) */
       e.ename, d.dname
FROM dept d, emp e
WHERE d.deptno = e.deptno
  AND d.loc = 'SEOUL';

-- Hash Join 강제: 대용량 집계
SELECT /*+ USE_HASH(o c) FULL(o) FULL(c) */
       c.cust_name, SUM(o.order_amt)
FROM orders o, customers c
WHERE o.cust_id = c.cust_id
GROUP BY c.cust_name;

-- Sort Merge Join: 범위 조인
SELECT /*+ USE_MERGE(e sg) */
       e.ename, sg.grade
FROM emp e, sal_grade sg
WHERE e.sal BETWEEN sg.losal AND sg.hisal;
```

---

## 7. 서브쿼리(Subquery) 처리 힌트

**서브쿼리를 어떻게 처리할지** 결정하는 힌트

| 힌트 | 설명 | 효과 | 적용 위치 |
|------|------|------|-----------|
| `UNNEST` | 서브쿼리를 메인 쿼리와 조인으로 변환 | 옵티마이저가 조인 순서 자유롭게 선택 | 서브쿼리 |
| `NO_UNNEST` | 서브쿼리 unnesting 방지 | 필터 방식으로 처리 강제 | 서브쿼리 |
| `PUSH_SUBQ` | 가능한 빨리 서브쿼리 필터 적용 | 중간 결과 집합 축소 | 서브쿼리 |
| `NO_PUSH_SUBQ` | 서브쿼리 필터 지연 적용 | 서브쿼리 늦게 실행 | 서브쿼리 |
| `PUSH_PRED` | 조인 조건을 뷰 내부로 밀어 넣기 | 뷰 내 인덱스 활용 가능 | 뷰 포함 쿼리 |
| `NO_PUSH_PRED` | 조인 조건 pushdown 방지 | 뷰를 그대로 실행 | 뷰 포함 쿼리 |
| `MERGE` | 뷰/인라인 뷰 Merging 강제 | 뷰를 메인 쿼리에 병합 | 인라인 뷰 |
| `NO_MERGE` | 뷰 Merging 방지 | 뷰를 독립 실행 | 인라인 뷰 |

```sql
-- UNNEST: 서브쿼리를 조인으로 변환하여 Hash Join 유도
SELECT /*+ UNNEST USE_HASH(dept) */ ename
FROM emp
WHERE deptno IN (
    SELECT /*+ UNNEST */ deptno
    FROM dept
    WHERE loc = 'SEOUL'
);

-- NO_MERGE: 인라인 뷰 독립 실행 유지
SELECT /*+ NO_MERGE(v) */ v.ename, v.avg_sal
FROM (
    SELECT /*+ NO_MERGE */ deptno, AVG(sal) avg_sal, ename
    FROM emp
    GROUP BY deptno, ename
) v
WHERE v.deptno = 10;
```

---

## 8. 병렬 처리(Parallel) 힌트

**병렬 실행 여부와 정도**를 제어하는 힌트

| 힌트 | 문법 | 설명 | 특징 |
|------|------|------|------|
| `PARALLEL` | `PARALLEL(table degree)` | 테이블 스캔 병렬화 | DOP(병렬도) 지정 |
| `NO_PARALLEL` | `NO_PARALLEL(table)` | 병렬 처리 금지 | 순차 처리 강제 |
| `PARALLEL_INDEX` | `PARALLEL_INDEX(t idx deg)` | 인덱스 병렬 스캔 | 파티션 인덱스 효과적 |
| `NO_PARALLEL_INDEX` | `NO_PARALLEL_INDEX(t idx)` | 인덱스 병렬 스캔 금지 | — |
| `PQ_DISTRIBUTE` | `PQ_DISTRIBUTE(t outer inner)` | 병렬 조인 데이터 분배 방식 | HASH, BROADCAST, NONE 등 |

```sql
-- 4개 병렬 프로세스로 Full Scan
SELECT /*+ FULL(s) PARALLEL(s 4) */ SUM(amount)
FROM sales s
WHERE sale_year = 2024;

-- 병렬 조인 분배 전략
SELECT /*+ PARALLEL(o 4) PARALLEL(c 4)
           PQ_DISTRIBUTE(o HASH HASH) */
       c.region, SUM(o.amt)
FROM orders o, customers c
WHERE o.cust_id = c.cust_id
GROUP BY c.region;
```

---

## 9. 결과 캐싱(Result Cache) 힌트

| 힌트 | 설명 | 적합한 상황 |
|------|------|------------|
| `RESULT_CACHE` | 쿼리 결과를 SGA의 Result Cache에 저장 | 동일 쿼리 반복 실행, 자주 변경 안 되는 참조 데이터 |
| `NO_RESULT_CACHE` | Result Cache 사용 금지 | 실시간 데이터 필요, 캐시 오염 방지 |

```sql
SELECT /*+ RESULT_CACHE */ dept_code, dept_name
FROM dept_master
WHERE use_yn = 'Y';
```

---

## 10. DML 처리 힌트

| 힌트 | 설명 | 특징 | 주의사항 |
|------|------|------|----------|
| `APPEND` | Direct Path Insert 사용 | 버퍼 캐시 우회, HWM 이후에 기록 | Exclusive Lock, 병렬 가능 |
| `APPEND_VALUES` | VALUES 절 INSERT에 Direct Path | 단건도 Direct Path 가능 | Oracle 11g R2+ |
| `NO_APPEND` | Direct Path Insert 금지 | 일반 INSERT | — |

```sql
-- Direct Path Insert로 대용량 데이터 빠르게 적재
INSERT /*+ APPEND */ INTO sales_backup
SELECT * FROM sales WHERE sale_year = 2023;
COMMIT; -- 반드시 커밋 필요
```

> ⚠️ `APPEND` 힌트 사용 시 테이블 전체 Exclusive Lock이 걸리므로 DML 동시성 주의

---

## 11. 기타 주요 힌트

| 힌트 | 문법 | 설명 | 활용 |
|------|------|------|------|
| `QB_NAME` | `QB_NAME(name)` | 쿼리 블록에 이름 부여 | 다른 블록에서 힌트 참조 |
| `DRIVING_SITE` | `DRIVING_SITE(table)` | 분산 DB에서 조인 수행 사이트 지정 | DB Link 사용 시 |
| `CARDINALITY` | `CARDINALITY(table n)` | 예상 카디널리티 수동 지정 | 통계 부정확 시 |
| `OPT_PARAM` | `OPT_PARAM('param' value)` | 옵티마이저 파라미터 쿼리 레벨 변경 | 세션 변경 없이 제어 |
| `DYNAMIC_SAMPLING` | `DYNAMIC_SAMPLING(t level)` | 동적 샘플링 레벨 지정 | 통계 없는 임시 테이블 |
| `GATHER_PLAN_STATISTICS` | `GATHER_PLAN_STATISTICS` | 실행 계획 통계 수집 | 튜닝 분석용 |
| `MONITOR` | `MONITOR` | SQL Monitor 활성화 | 실시간 모니터링 |
| `NO_MONITOR` | `NO_MONITOR` | SQL Monitor 비활성화 | 오버헤드 감소 |
| `CACHE` | `CACHE(table)` | 전체 테이블 캐시 유지 | 소형 참조 테이블 |
| `NOCACHE` | `NOCACHE(table)` | LRU 리스트 끝에 배치 | 풀스캔 후 빠른 버퍼 해제 |

---

## 12. 전체 힌트 요약 테이블

| 분류 | 힌트명 | 핵심 기능 | 키워드 |
|------|--------|-----------|--------|
| **목표** | `ALL_ROWS` | 전체 처리량 최적화 | 배치, DW |
| **목표** | `FIRST_ROWS(n)` | 첫 n행 응답 최적화 | OLTP, 페이징 |
| **액세스** | `FULL` | 풀 테이블 스캔 | 대량, 통계 오류 |
| **액세스** | `INDEX` | 인덱스 Range Scan | 소량, 선택도 |
| **액세스** | `INDEX_FFS` | 인덱스 Fast Full Scan | COUNT, 인덱스만 |
| **액세스** | `INDEX_DESC` | 인덱스 역방향 스캔 | MAX, 최신 데이터 |
| **액세스** | `INDEX_SS` | 인덱스 Skip Scan | 선두 컬럼 조건 없음 |
| **액세스** | `NO_INDEX` | 인덱스 사용 금지 | 잘못된 인덱스 배제 |
| **조인순서** | `LEADING` | 조인 순서 지정 | 드라이빙 테이블 |
| **조인순서** | `ORDERED` | FROM 절 순서 조인 | 구버전 호환 |
| **조인방식** | `USE_NL` | Nested Loop Join | OLTP, 소량 |
| **조인방식** | `USE_HASH` | Hash Join | 대용량, 동등 조인 |
| **조인방식** | `USE_MERGE` | Sort Merge Join | 비동등, 정렬된 데이터 |
| **서브쿼리** | `UNNEST` | 서브쿼리 → 조인 변환 | 옵티마이저 자유도 |
| **서브쿼리** | `NO_UNNEST` | 필터 방식 유지 | 서브쿼리 독립 실행 |
| **서브쿼리** | `PUSH_SUBQ` | 서브쿼리 조기 필터 | 결과 집합 축소 |
| **서브쿼리** | `NO_MERGE` | 인라인 뷰 독립 실행 | 뷰 최적화 방지 |
| **서브쿼리** | `PUSH_PRED` | 조건 뷰 내부 밀어넣기 | 뷰 인덱스 활용 |
| **병렬** | `PARALLEL` | 병렬 스캔 | DOP 지정, 배치 |
| **병렬** | `NO_PARALLEL` | 병렬 금지 | OLTP |
| **캐싱** | `RESULT_CACHE` | 결과 집합 캐시 | 반복 쿼리 |
| **DML** | `APPEND` | Direct Path Insert | 대용량 적재 |
| **기타** | `QB_NAME` | 쿼리 블록 명명 | 복잡한 서브쿼리 |
| **기타** | `DYNAMIC_SAMPLING` | 동적 통계 수집 | 임시 테이블 |
| **기타** | `GATHER_PLAN_STATISTICS` | 실행 통계 수집 | 튜닝 분석 |
| **기타** | `CACHE` | 버퍼 캐시 고정 | 소형 참조 테이블 |

---

## 13. 힌트 적용 시 주의사항

### 13.1 힌트가 무시되는 경우

1. **오타**: 힌트명이 틀리면 에러 없이 무시 (`/*+ INDX(t idx) */` → 무시)
2. **존재하지 않는 객체**: 없는 인덱스명 지정
3. **의미 없는 힌트**: `USE_NL`인데 테이블이 하나인 경우
4. **논리적 불가능**: 비동등 조인에 `USE_HASH` 단독 적용 (불가)
5. **충돌**: `FULL`과 `INDEX`를 같은 테이블에 동시 적용

### 13.2 힌트 사용 Best Practice

```
✅ DO
- 알리아스(Alias)가 있으면 테이블명 대신 알리아스를 사용
  → INDEX(e emp_sal_idx) ← e는 emp의 alias
- 조인 힌트와 액세스 힌트를 함께 사용
  → LEADING(d e) USE_NL(e) INDEX(e emp_dept_idx)
- 실행 계획 검증 후 힌트 적용
  → EXPLAIN PLAN 또는 DBMS_XPLAN 활용

❌ DON'T
- 통계 갱신 전 무분별한 힌트 적용
- APPEND 힌트 후 COMMIT 누락
- 과도한 힌트로 유지보수 복잡도 증가
- 운영 환경에 검증 없이 힌트 적용
```

### 13.3 힌트 검증 방법

```sql
-- 1. 실행 계획 확인
EXPLAIN PLAN FOR
SELECT /*+ INDEX(e emp_deptno_idx) */ *
FROM emp e WHERE deptno = 10;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- 2. 실제 실행 통계 확인 (실행 후)
SELECT /*+ GATHER_PLAN_STATISTICS INDEX(e emp_deptno_idx) */
       * FROM emp e WHERE deptno = 10;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));

-- 3. 힌트 리포트 확인 (Oracle 19c+)
-- Note 섹션에 "hint report" 포함 여부 확인
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'HINT_REPORT'));
```

---

## 14. DAP 시험 핵심 포인트 요약

| 구분 | 암기 포인트 |
|------|------------|
| 힌트 기본 | `/*+ */` 형식, 무효 힌트는 에러 없이 무시 |
| 조인 방식 | NL=소량/인덱스, Hash=대용량/동등, Merge=비동등/정렬 |
| 액세스 | INDEX → Range Scan, INDEX_FFS → Multiblock 인덱스 전체 |
| 드라이빙 | 선택도 높은 테이블을 선행 / NL의 루프 횟수 = 드라이빙 건수 |
| APPEND | Direct Path = HWM 이후 기록, Exclusive Lock, 커밋 필수 |
| 서브쿼리 | UNNEST=조인변환, NO_UNNEST=필터유지, PUSH_SUBQ=조기 필터 |
| Result Cache | SGA 내 공유, 동일 쿼리 반복 시 효과적 |

---
*본 문서는 Oracle 12c 이상 기준으로 작성되었습니다. 일부 힌트는 버전에 따라 동작이 다를 수 있습니다.*
