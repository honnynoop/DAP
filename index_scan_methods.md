# 인덱스 스캔 방식 완전 정리

---

## 0. 인덱스 기본 구조 (B-Tree)

모든 스캔 방식을 이해하려면 **B-Tree 인덱스 구조**를 먼저 알아야 한다.

```
                    [ Root Block ]
                    [ 20 | 40 ]
                   /      |      \
          [Branch]    [Branch]    [Branch]
          [10|15]     [20|30]     [40|50]
         /   |  \      |    \      |    \
       [L1] [L2] [L3] [L4] [L5]  [L6] [L7]   ← Leaf Block
        ↕    ↕    ↕    ↕    ↕     ↕    ↕
       (Leaf Block은 양방향 연결 리스트로 연결됨)

각 Leaf Block 내부:
  ┌────────────────────────────────┐
  │ [키값] → [ROWID]               │
  │ 10     → AAA.001.001           │
  │ 12     → AAA.001.003           │
  │ 15     → AAA.002.001           │
  └────────────────────────────────┘
  ROWID = 데이터 파일번호.블록번호.행번호 (테이블 행의 물리적 주소)
```

**B-Tree 3계층 구조:**
- **Root Block**: 탐색의 시작점, Branch를 가리킴
- **Branch Block**: 범위를 나눠 Leaf를 가리킴
- **Leaf Block**: 실제 키값 + ROWID 저장, 양방향 연결

---

## 1. Index Range Scan (인덱스 범위 스캔)

### 개념
**가장 일반적인 인덱스 스캔 방식.** Root → Branch → Leaf 순으로 내려가 **탐색 시작점**을 찾은 뒤, Leaf Block을 **순방향으로 스캔**하고 각 ROWID로 테이블을 랜덤 접근한다.

### 동작 원리

```
[예시] WHERE deptno = 20

Step 1. Root에서 20이 있는 Branch 탐색
Step 2. Branch에서 20이 있는 Leaf 탐색
Step 3. Leaf에서 deptno=20인 첫 번째 키 발견
Step 4. 해당 ROWID로 테이블 블록 접근 (랜덤 I/O)
Step 5. 같은 Leaf에서 다음 deptno=20 키 확인 → ROWID로 테이블 접근
Step 6. deptno=21이 나오면 스캔 종료

         Root
          |
        Branch
          |
   Leaf → Leaf → Leaf
  [20,R1][20,R2][20,R3] [21,...] ← 여기서 중단
     ↓      ↓      ↓
  Table  Table  Table   ← 각각 랜덤 I/O 발생
```

### 특징 및 적합한 상황

| 항목 | 내용 |
|------|------|
| I/O 방식 | **Single Block I/O** (한 번에 1블록) |
| 정렬 보장 | ✅ 인덱스 순서대로 결과 반환 |
| 조건 유형 | `=`, `BETWEEN`, `>=`, `<=`, `LIKE 'A%'` |
| 적합한 상황 | 소량 데이터 조회, 선택도 높은 조건 |
| 주의사항 | 대량 반환 시 랜덤 I/O 폭증 → FTS가 유리 |

```sql
-- Range Scan 유도
SELECT /*+ INDEX(e emp_deptno_idx) */ empno, ename
FROM emp e
WHERE deptno = 20;                     -- 동등 조건

SELECT * FROM orders
WHERE order_date BETWEEN DATE '2024-01-01' AND DATE '2024-12-31'; -- 범위 조건
```

---

## 2. Index Full Scan (인덱스 전체 스캔)

### 개념
Root → Branch → **첫 번째 Leaf Block**부터 **마지막 Leaf Block까지** 인덱스 전체를 **순차적으로** 스캔한다. 단, Multiblock I/O가 아닌 **Single Block I/O**를 사용한다.

### 동작 원리

```
[예시] SELECT ename FROM emp ORDER BY ename; (IDX on ename 존재 시)

         Root
          |
        Branch
          |
   Leaf → Leaf → Leaf → Leaf → Leaf  ← 전체 순차 스캔
  [A...] [B...] [C...] [...] [Z...]
    ↓      ↓                          ← 필요한 경우만 Table 접근
  Table  Table

※ 인덱스가 이미 ename 순으로 정렬되어 있으므로
  결과도 정렬된 상태로 반환 → ORDER BY의 Sort 연산 생략!
```

### 특징 및 적합한 상황

| 항목 | 내용 |
|------|------|
| I/O 방식 | **Single Block I/O** |
| 정렬 보장 | ✅ 인덱스 컬럼 순서대로 정렬 보장 |
| 조건 유형 | 조건 없거나 전체 조회 |
| 적합한 상황 | ORDER BY, MIN/MAX 빠른 조회, Sort 연산 제거 목적 |
| Index FFS와 차이 | 정렬은 보장되지만 속도는 FFS보다 느림 |

```sql
-- Full Scan으로 Sort 제거 (인덱스가 이미 sal 순 정렬)
SELECT /*+ INDEX(e emp_sal_idx) */ ename, sal
FROM emp e
ORDER BY sal;  -- Sort 연산 없이 인덱스 순서로 반환

-- MIN/MAX 즉시 조회 (첫 번째 / 마지막 Leaf만 접근)
SELECT MIN(sal), MAX(sal) FROM emp;
-- → Index Full Scan (MIN/MAX) : Leaf의 양 끝만 읽음 → 극히 빠름
```

---

## 3. Index Fast Full Scan (인덱스 고속 전체 스캔)

### 개념
인덱스 전체를 **Multiblock I/O**로 한 번에 여러 블록씩 읽는다. 테이블을 접근하지 않고 **인덱스만으로 결과를 반환**한다. 대신 **정렬 순서를 보장하지 않는다.**

### 동작 원리

```
[예시] SELECT COUNT(*) FROM emp; (IDX on empno 존재 시)

일반 Full Scan:
  [L1] → [L2] → [L3] → [L4]  (Single Block씩 순차)
  I/O: 4번

Fast Full Scan:
  [L1][L2][L3][L4]  (Multiblock I/O: 한 번에 여러 블록)
  I/O: 1번 (db_file_multiblock_read_count 설정값만큼 묶음)

※ 테이블 접근 없음! 인덱스 블록 안에 있는 값만 사용
  → empno의 COUNT → 테이블 안 가도 됨
```

### 특징 및 적합한 상황

| 항목 | 내용 |
|------|------|
| I/O 방식 | **Multiblock I/O** (한 번에 여러 블록) |
| 정렬 보장 | ❌ 물리적 저장 순서대로 반환 (정렬 무보장) |
| 테이블 접근 | ❌ 없음 (인덱스만으로 처리) |
| 적합한 상황 | `COUNT(*)`, `SUM(인덱스컬럼)`, 인덱스 컬럼만 SELECT |
| 병렬 처리 | ✅ 가능 |

```sql
-- Covering Index: 인덱스 컬럼(deptno, empno, sal)만 조회
-- IDX(deptno, empno, sal) 존재 시
SELECT /*+ INDEX_FFS(e emp_dept_empno_sal_idx) */
       deptno, COUNT(*), SUM(sal)
FROM emp e
GROUP BY deptno;
-- → 테이블 접근 없이 인덱스만으로 집계 완료
```

---

## 4. Index Skip Scan (인덱스 스킵 스캔)

### 개념
복합 인덱스에서 **선두 컬럼 조건이 없을 때**, 선두 컬럼의 **Distinct 값마다 서브 인덱스가 있다고 간주**하고 해당 영역을 건너뛰며(Skip) 스캔한다.

### 동작 원리

```
[예시] IDX(gender, sal) 복합 인덱스, WHERE sal = 3000

인덱스 저장 구조:
  [F, 800]  [F,1100]  [F,3000]  [M,2450]  [M,3000]  [M,5000]
   ↑선두 컬럼(F/M) 2가지 Distinct 값

Skip Scan 동작:
  Step 1. gender = 'F' 구간에서 sal = 3000 탐색
           → [F, 3000] 발견 → ROWID로 테이블 접근
  Step 2. gender = 'M' 구간으로 건너뜀(Skip)
           → [M, 3000] 발견 → ROWID로 테이블 접근
  Step 3. 종료

  = 마치 WHERE gender = 'F' AND sal = 3000
    UNION ALL
    WHERE gender = 'M' AND sal = 3000 처럼 동작
```

### 선두 컬럼 Distinct 수에 따른 효율성

```
선두 Distinct = 2 (F/M):
  구간 2개만 탐색 → 매우 효율적 ✅

선두 Distinct = 1000 (지역코드):
  구간 1000개 탐색 → 인덱스 Range Scan 1000번
  → FTS보다 오히려 느릴 수 있음 ❌
```

### 특징 및 적합한 상황

| 항목 | 내용 |
|------|------|
| I/O 방식 | Single Block I/O |
| 정렬 보장 | △ (선두 컬럼 순서로는 정렬) |
| 조건 유형 | 선두 컬럼 조건 없고, 이후 컬럼 조건 있음 |
| 적합한 상황 | 선두 컬럼 Distinct 수가 **매우 적을 때** |
| 부적합한 상황 | 선두 Distinct 많으면 오히려 비효율 |

```sql
-- Skip Scan 힌트 사용
SELECT /*+ INDEX_SS(e idx_gender_sal) */ empno, ename, sal
FROM emp e
WHERE sal = 3000;  -- 선두 컬럼(gender) 조건 없음
```

---

## 5. Index Unique Scan (인덱스 유일 스캔)

### 개념
**Unique 인덱스(PK, Unique 제약)**에서 `=` 조건으로 **딱 한 건만** 찾는 방식. 찾는 즉시 스캔 종료.

### 동작 원리

```
[예시] WHERE empno = 7369 (PK 인덱스)

         Root
          |
        Branch
          |
        Leaf
    [...][7369, ROWID][...]
              ↓
           Table (단 1번 접근)
              ↓
           종료! (더 이상 스캔 불필요)

Range Scan과 차이:
- Range Scan: 동일값 끝까지 스캔
- Unique Scan: 1건 찾으면 즉시 종료 → 더 빠름
```

### 특징 및 적합한 상황

| 항목 | 내용 |
|------|------|
| I/O 방식 | Single Block I/O, 최소 횟수 |
| 정렬 보장 | ✅ (1건이므로 의미 없음) |
| 조건 유형 | `=` 조건 + Unique 인덱스 |
| 적합한 상황 | PK 조회, Unique 컬럼 동등 조건 |
| 실행 계획 표시 | `INDEX UNIQUE SCAN` |

```sql
-- PK 조회 → Unique Scan 자동 선택
SELECT * FROM emp WHERE empno = 7369;  -- PK
SELECT * FROM dept WHERE deptno = 10; -- PK
```

---

## 6. Index Range Scan Descending (역방향 범위 스캔)

### 개념
Index Range Scan의 역방향 버전. Leaf Block을 **역순(내림차순)으로** 스캔한다. Leaf Block이 양방향 연결 리스트이므로 가능하다.

### 동작 원리

```
[예시] 최신 주문 10건 조회 (IDX on order_date)

정방향 Leaf:  [2024-01] → [2024-06] → [2024-12]
역방향 Leaf:  [2024-12] → [2024-06] → [2024-01]

WHERE ROWNUM <= 10 ORDER BY order_date DESC
→ INDEX_DESC 힌트: 가장 오른쪽 Leaf부터 역순 스캔
→ 10건 찾으면 즉시 종료 → 전체 스캔 불필요!
```

### 특징 및 적합한 상황

| 항목 | 내용 |
|------|------|
| I/O 방식 | Single Block I/O |
| 정렬 보장 | ✅ 내림차순 정렬 보장 |
| 조건 유형 | `ORDER BY DESC`, `MAX` 조회 |
| 적합한 상황 | 최신 데이터 조회, 내림차순 페이징 |

```sql
-- 최신 주문 10건 (정렬 없이 인덱스 역방향 스캔)
SELECT /*+ INDEX_DESC(o ord_date_idx) */ *
FROM orders o
WHERE ROWNUM <= 10;
-- ORDER BY order_date DESC 없어도 역순 반환!
```

---

## 7. 전체 비교 요약 테이블

| 스캔 방식 | I/O 방식 | 정렬 보장 | 테이블 접근 | 적합 조건 | 대표 사용 목적 |
|-----------|----------|-----------|------------|-----------|--------------|
| **Range Scan** | Single Block | ✅ 오름차순 | ✅ 있음 | `=`, 범위 | 소량 조회, 인덱스 조건 있을 때 |
| **Full Scan** | Single Block | ✅ 오름차순 | △ 있을 수 있음 | 전체 or 정렬 | ORDER BY, MIN/MAX |
| **Fast Full Scan** | **Multi Block** | ❌ 없음 | ❌ 없음 | 인덱스 컬럼만 필요 | COUNT, SUM, 집계 |
| **Skip Scan** | Single Block | △ 부분적 | ✅ 있음 | 선두 컬럼 조건 없음 | 복합 인덱스 선두 미사용 |
| **Unique Scan** | Single Block (최소) | ✅ (1건) | ✅ 있음 (1회) | `=` + Unique | PK/UK 단건 조회 |
| **Descending** | Single Block | ✅ 내림차순 | ✅ 있음 | DESC 정렬 | 최신 데이터, DESC 페이징 |

---

## 8. 스캔 방식 선택 흐름도

```
인덱스 스캔 방식 결정 흐름

조건이 있는가?
  ├─ YES → Unique 인덱스 + = 조건?
  │          ├─ YES → [Unique Scan]
  │          └─ NO  → 선두 컬럼 조건 있는가?
  │                    ├─ YES → 범위/동등 조건?
  │                    │          ├─ = → [Range Scan]
  │                    │          ├─ 범위 → [Range Scan]
  │                    │          └─ DESC → [Range Scan Descending]
  │                    └─ NO  → 선두 Distinct 적은가?
  │                              ├─ YES → [Skip Scan]
  │                              └─ NO  → 다른 인덱스 / FTS 검토
  └─ NO  → 인덱스 컬럼만 조회하는가?
             ├─ YES, 정렬 불필요 → [Fast Full Scan]  ← COUNT, SUM 등
             └─ YES, 정렬 필요  → [Full Scan]        ← ORDER BY
             └─ NO              → FTS
```

---

## 9. 핵심 비유로 이해하기

```
📚 인덱스를 책의 목차라고 비유하면:

Range Scan     = 목차에서 "3장 시작 페이지"를 찾아 3장 끝까지 읽기
Full Scan      = 목차 전체를 처음부터 끝까지 순서대로 읽기
Fast Full Scan = 목차 페이지들을 한 번에 사진 찍듯 빠르게 스캔
                 (순서는 엉망이지만 빠름, 본문 안 봄)
Skip Scan      = 저자별 목차에서 저자는 모르고 제목만 알 때
                 저자 A 목차 → 해당 제목 탐색, 저자 B 목차 → 탐색...
Unique Scan    = 목차에서 딱 하나만 있는 항목 찾고 즉시 책 덮기
Descending     = 목차 맨 뒤에서부터 거꾸로 읽기
```

---

> 💡 **결론**: 스캔 방식은 `상황에 맞게 선택`하는 것이 중요하다.
> - 소량 단건 → **Unique / Range Scan**
> - 집계, COUNT → **Fast Full Scan** (테이블 접근 Zero)
> - 정렬 필요, 소량 → **Full Scan** (Sort 제거 목적)
> - 최신 데이터 → **Descending**
> - 선두 없는 복합 인덱스 → **Skip Scan** (Distinct 적을 때만)
