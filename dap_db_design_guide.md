# 데이터베이스 설계 - DAP 시험 완벽 가이드

> **DAP (Data Architecture Professional) 시험 대비 종합 가이드**  
> 저장공간, 무결성, 인덱스, 분산, 보안 설계 완전 정복

---

## 📑 목차

1. [저장공간 설계](#1-저장공간-설계)
2. [무결성 설계](#2-무결성-설계)
3. [인덱스 설계](#3-인덱스-설계)
4. [분산 설계](#4-분산-설계)
5. [보안 설계](#5-보안-설계)
6. [시험 대비 핵심 요약](#6-시험-대비-핵심-요약)

---

## 1. 저장공간 설계

### 1.1 테이블스페이스(Tablespace) 설계

#### 개념
테이블스페이스는 데이터베이스 객체들이 물리적으로 저장되는 **논리적 저장 공간**입니다.

#### 분리 원칙 (SUTI 원칙)

| 구분 | 테이블스페이스 | 용도 | 분리 이유 |
|------|----------------|------|-----------|
| **S**ystem | 시스템 TS | 데이터 딕셔너리, 메타데이터 | 시스템 데이터 보호 |
| **U**ser | 사용자 데이터 TS | 업무 테이블 | 데이터 격리 |
| **T**emporary | 임시 TS | 정렬, 해시 조인 작업 | 성능 향상 |
| **I**ndex | 인덱스 TS | 인덱스 객체 | I/O 분산 |

**💡 암기 팁**: "**수**능 **유**니폼 **테**스트 **인**증서" (SUTI)

#### 크기 산정 공식

```
초기 크기 = 현재 데이터 크기 × 1.5 ~ 2.0
자동 확장 = 초기 크기의 10% ~ 20%
최대 크기 = 3~5년 예상 증가량 고려
```

**예제 계산**:
- 현재 데이터: 100GB
- 연간 증가율: 20%
- 예상 기간: 3년

```
3년 후 예상 = 100 × (1.2)³ = 172.8GB
초기 크기 = 172.8 × 1.5 = 259.2GB → 260GB
자동 확장 = 26GB (10%)
최대 크기 = 260 + (26 × 10) = 520GB
```

#### 설계 시 고려사항

| 항목 | OLTP | OLAP/DW |
|------|------|---------|
| 블록 크기 | 4KB ~ 8KB | 16KB ~ 32KB |
| 초기 할당 | 작게 시작 | 크게 할당 |
| 자동 확장 | 빈번하게 | 대용량 단위로 |
| 파티셔닝 | 선택적 | 필수적 |

---

### 1.2 파티셔닝(Partitioning) 전략

#### 파티셔닝 유형 비교

| 유형 | 분할 기준 | 적용 사례 | 장점 | 단점 |
|------|----------|-----------|------|------|
| **Range** | 범위 (날짜, 숫자) | 주문(월별), 로그(일별) | 범위 검색 효율적 | 데이터 편중 가능 |
| **List** | 값 목록 | 지역(국가별), 상태코드 | 명확한 분류 | 유연성 부족 |
| **Hash** | 해시 함수 | 고객ID, 계좌번호 | 균등 분산 | 범위 검색 불리 |
| **Composite** | 복합 (Range+Hash 등) | 대용량 이력 데이터 | 유연성 높음 | 관리 복잡 |

#### Range 파티셔닝 예제

```sql
-- 주문 테이블 월별 파티셔닝
CREATE TABLE 주문 (
    주문번호 NUMBER,
    주문일자 DATE,
    고객ID NUMBER,
    금액 NUMBER
)
PARTITION BY RANGE (주문일자) (
    PARTITION p_2024_01 VALUES LESS THAN (TO_DATE('2024-02-01', 'YYYY-MM-DD')),
    PARTITION p_2024_02 VALUES LESS THAN (TO_DATE('2024-03-01', 'YYYY-MM-DD')),
    PARTITION p_2024_03 VALUES LESS THAN (TO_DATE('2024-04-01', 'YYYY-MM-DD'))
);
```

#### Hash 파티셔닝 예제

```sql
-- 고객 테이블 Hash 파티셔닝 (균등 분산)
CREATE TABLE 고객 (
    고객ID NUMBER,
    고객명 VARCHAR2(100),
    지역코드 VARCHAR2(10)
)
PARTITION BY HASH (고객ID)
PARTITIONS 8;  -- 8개 파티션으로 균등 분산
```

#### Composite 파티셔닝 예제

```sql
-- Range-Hash 복합 파티셔닝
CREATE TABLE 거래이력 (
    거래ID NUMBER,
    거래일자 DATE,
    고객ID NUMBER,
    거래금액 NUMBER
)
PARTITION BY RANGE (거래일자)
SUBPARTITION BY HASH (고객ID) SUBPARTITIONS 4 (
    PARTITION p_2024_q1 VALUES LESS THAN (TO_DATE('2024-04-01', 'YYYY-MM-DD')),
    PARTITION p_2024_q2 VALUES LESS THAN (TO_DATE('2024-07-01', 'YYYY-MM-DD'))
);
```

#### 파티셔닝 효과

| 효과 | 설명 | 성능 개선 |
|------|------|-----------|
| **Partition Pruning** | 필요한 파티션만 스캔 | 50% ~ 90% 성능 향상 |
| **병렬 처리** | 각 파티션 독립 처리 | N배 (파티션 수) |
| **관리 용이성** | 파티션 단위 백업/삭제 | 운영 시간 단축 |
| **가용성 향상** | 일부 파티션 장애 격리 | 서비스 연속성 |

---

### 1.3 블록 크기 및 저장 매개변수

#### 블록 크기 결정

| 시스템 유형 | 권장 블록 크기 | 이유 |
|------------|---------------|------|
| OLTP | 4KB ~ 8KB | 랜덤 액세스, 작은 트랜잭션 |
| OLAP/DW | 16KB ~ 32KB | 순차 스캔, 대량 데이터 |
| 혼합 환경 | 8KB | 절충안 |

#### PCTFREE와 PCTUSED

**PCTFREE (Percent Free)**
- **정의**: 블록 내에서 향후 UPDATE를 위해 예약할 공간 비율
- **설정 기준**:

| 테이블 특성 | PCTFREE 값 | 사례 |
|------------|-----------|------|
| INSERT 위주, UPDATE 거의 없음 | 5% ~ 10% | 로그 테이블, 이력 테이블 |
| UPDATE 보통 | 10% ~ 20% | 일반 업무 테이블 |
| UPDATE 빈번, 컬럼 크기 증가 | 20% ~ 30% | 고객정보, 상품설명 |
| LOB 데이터 포함 | 30% ~ 40% | 문서, 이미지 메타데이터 |

**PCTUSED (Percent Used)**
- **정의**: 블록이 다시 INSERT 가능한 상태가 되는 사용률 임계값
- **공식**: PCTFREE + PCTUSED < 100 (권장: 합계 80~90)

```sql
-- 예제: 로그 테이블 (INSERT 위주)
CREATE TABLE 접속로그 (
    로그ID NUMBER,
    접속시간 TIMESTAMP,
    사용자ID VARCHAR2(50),
    IP주소 VARCHAR2(50)
)
PCTFREE 5    -- UPDATE 거의 없음
PCTUSED 80;  -- 빠른 공간 재사용

-- 예제: 고객 테이블 (UPDATE 빈번)
CREATE TABLE 고객정보 (
    고객ID NUMBER,
    고객명 VARCHAR2(100),
    주소 VARCHAR2(500),   -- 변경 가능성 높음
    비고 VARCHAR2(2000)   -- 자주 수정됨
)
PCTFREE 25   -- UPDATE 공간 충분히 확보
PCTUSED 70;  -- 적절한 재사용 임계값
```

#### Row Chaining과 Row Migration 방지

| 현상 | 원인 | 해결 방법 |
|------|------|-----------|
| **Row Chaining** | 행 크기가 블록 크기보다 큼 | 블록 크기 증가, 컬럼 정규화 |
| **Row Migration** | UPDATE로 행 크기 증가 → 다른 블록으로 이동 | PCTFREE 증가, 테이블 재구성 |

---

## 2. 무결성 설계

### 2.1 무결성 제약조건 체계

```
무결성 계층 구조
├── 도메인 무결성 (Domain Integrity) - 컬럼 수준
├── 개체 무결성 (Entity Integrity) - 행 수준
├── 참조 무결성 (Referential Integrity) - 테이블 간
└── 사용자 정의 무결성 (User-Defined Integrity) - 비즈니스 규칙
```

---

### 2.2 도메인 무결성 (Domain Integrity)

#### 제약조건 유형

| 제약조건 | 문법 | 사례 | 효과 |
|---------|------|------|------|
| **NOT NULL** | `컬럼명 NOT NULL` | 고객명, 주문일자 | 필수 입력 강제 |
| **DEFAULT** | `컬럼명 DEFAULT 값` | 등록일자 DEFAULT SYSDATE | 자동 값 할당 |
| **CHECK** | `CHECK (조건)` | 나이 >= 0 | 값 범위 제한 |
| **데이터 타입** | `NUMBER(10,2)` | 금액, 수량 | 형식 제한 |

#### 실전 예제

```sql
CREATE TABLE 회원 (
    회원ID NUMBER PRIMARY KEY,
    회원명 VARCHAR2(100) NOT NULL,
    이메일 VARCHAR2(200) NOT NULL UNIQUE,
    나이 NUMBER CHECK (나이 >= 0 AND 나이 <= 150),
    성별 CHAR(1) CHECK (성별 IN ('M', 'F')),
    가입일자 DATE DEFAULT SYSDATE NOT NULL,
    상태코드 VARCHAR2(10) DEFAULT 'ACTIVE' CHECK (상태코드 IN ('ACTIVE', 'INACTIVE', 'SUSPENDED')),
    포인트 NUMBER DEFAULT 0 CHECK (포인트 >= 0),
    등급 VARCHAR2(10) CHECK (등급 IN ('BRONZE', 'SILVER', 'GOLD', 'PLATINUM'))
);
```

#### CHECK 제약조건 고급 패턴

```sql
-- 1. 날짜 범위 검증
CHECK (종료일자 >= 시작일자)

-- 2. 조건부 필수 입력
CHECK ((배송방법 = '택배' AND 수령인명 IS NOT NULL) OR 배송방법 != '택배')

-- 3. 상호 배타적 컬럼
CHECK ((개인고객번호 IS NOT NULL AND 법인고객번호 IS NULL) OR 
       (개인고객번호 IS NULL AND 법인고객번호 IS NOT NULL))

-- 4. 정규표현식 검증 (Oracle)
CHECK (REGEXP_LIKE(전화번호, '^[0-9]{2,3}-[0-9]{3,4}-[0-9]{4}$'))

-- 5. 이메일 형식 검증
CHECK (REGEXP_LIKE(이메일, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'))
```

---

### 2.3 개체 무결성 (Entity Integrity)

#### 기본키(Primary Key) 설계 원칙

| 원칙 | 설명 | 예제 |
|------|------|------|
| **유일성** | 중복 불가 | 주민등록번호, 사원번호 |
| **최소성** | 최소한의 컬럼 조합 | 단일 컬럼 > 복합 컬럼 |
| **불변성** | 값 변경 불가 | 자동 증가 ID |
| **NOT NULL** | NULL 허용 안 함 | 모든 PK |

#### 자연키 vs 대리키 비교

| 구분 | 자연키 (Natural Key) | 대리키 (Surrogate Key) |
|------|---------------------|----------------------|
| **정의** | 업무적 의미가 있는 속성 | 인위적으로 생성한 식별자 |
| **예시** | 주민번호, 사업자번호, 학번 | SEQUENCE, IDENTITY, UUID |
| **장점** | 비즈니스 의미 명확 | 안정적, 변경 없음 |
| **단점** | 변경 가능성, 개인정보 이슈 | 비즈니스 의미 없음 |
| **권장** | 변경 가능성 없는 경우 | 대부분의 경우 권장 ✅ |

#### 대리키 생성 방법

```sql
-- 1. Oracle SEQUENCE
CREATE SEQUENCE 고객_SEQ
    START WITH 1
    INCREMENT BY 1
    CACHE 20;

INSERT INTO 고객 (고객ID, 고객명) 
VALUES (고객_SEQ.NEXTVAL, '홍길동');

-- 2. MySQL AUTO_INCREMENT
CREATE TABLE 고객 (
    고객ID INT AUTO_INCREMENT PRIMARY KEY,
    고객명 VARCHAR(100)
);

-- 3. PostgreSQL SERIAL
CREATE TABLE 고객 (
    고객ID SERIAL PRIMARY KEY,
    고객명 VARCHAR(100)
);

-- 4. UUID (범용 고유 식별자)
CREATE TABLE 고객 (
    고객ID VARCHAR2(36) DEFAULT SYS_GUID() PRIMARY KEY,
    고객명 VARCHAR2(100)
);
```

---

### 2.4 참조 무결성 (Referential Integrity)

#### 외래키(Foreign Key) 제약조건

**기본 문법**:
```sql
CREATE TABLE 주문 (
    주문번호 NUMBER PRIMARY KEY,
    고객ID NUMBER NOT NULL,
    주문일자 DATE,
    CONSTRAINT fk_주문_고객 FOREIGN KEY (고객ID) 
        REFERENCES 고객(고객ID)
);
```

#### 연쇄 작업 (CASCADE) 옵션

| 옵션 | 동작 | 사용 시나리오 | 주의사항 |
|------|------|--------------|----------|
| **ON DELETE CASCADE** | 부모 삭제 시 자식도 삭제 | 주문-주문상세, 게시글-댓글 | 데이터 소실 위험 ⚠️ |
| **ON DELETE SET NULL** | 부모 삭제 시 자식 FK를 NULL로 | 담당자 퇴사 시 | FK는 NULL 허용해야 함 |
| **ON DELETE SET DEFAULT** | 부모 삭제 시 기본값으로 | 기본 카테고리로 변경 | DEFAULT 값 필요 |
| **ON DELETE NO ACTION** | 자식 존재 시 부모 삭제 거부 | 고객-주문 (기본값) | 가장 안전 ✅ |
| **ON UPDATE CASCADE** | 부모 PK 변경 시 자식 FK도 변경 | 코드 체계 변경 | 성능 영향 큼 |

#### 연쇄 작업 예제

```sql
-- 예제 1: CASCADE 삭제 (게시글-댓글)
CREATE TABLE 게시글 (
    게시글ID NUMBER PRIMARY KEY,
    제목 VARCHAR2(200),
    내용 CLOB
);

CREATE TABLE 댓글 (
    댓글ID NUMBER PRIMARY KEY,
    게시글ID NUMBER,
    댓글내용 VARCHAR2(500),
    CONSTRAINT fk_댓글_게시글 FOREIGN KEY (게시글ID)
        REFERENCES 게시글(게시글ID)
        ON DELETE CASCADE  -- 게시글 삭제 시 댓글도 자동 삭제
);

-- 예제 2: SET NULL (직원-부서)
CREATE TABLE 부서 (
    부서코드 VARCHAR2(10) PRIMARY KEY,
    부서명 VARCHAR2(100)
);

CREATE TABLE 직원 (
    직원ID NUMBER PRIMARY KEY,
    직원명 VARCHAR2(100),
    부서코드 VARCHAR2(10),
    CONSTRAINT fk_직원_부서 FOREIGN KEY (부서코드)
        REFERENCES 부서(부서코드)
        ON DELETE SET NULL  -- 부서 삭제 시 직원의 부서코드를 NULL로
);

-- 예제 3: NO ACTION (고객-주문) - 기본값
CREATE TABLE 고객 (
    고객ID NUMBER PRIMARY KEY,
    고객명 VARCHAR2(100)
);

CREATE TABLE 주문 (
    주문번호 NUMBER PRIMARY KEY,
    고객ID NUMBER,
    CONSTRAINT fk_주문_고객 FOREIGN KEY (고객ID)
        REFERENCES 고객(고객ID)
        -- ON DELETE NO ACTION이 기본값
        -- 주문이 있는 고객은 삭제 불가
);
```

#### 외래키 인덱스 생성 필수!

```sql
-- ✅ 반드시 외래키 컬럼에 인덱스 생성
CREATE INDEX idx_주문_고객ID ON 주문(고객ID);
CREATE INDEX idx_댓글_게시글ID ON 댓글(게시글ID);
```

**이유**:
1. **조인 성능 향상**: FK 기반 조인이 빈번함
2. **참조 무결성 체크 성능**: 부모 삭제 시 자식 존재 여부 확인
3. **잠금 경합 감소**: 인덱스 없으면 테이블 풀 스캔 → 전체 테이블 락

---

### 2.5 사용자 정의 무결성

#### 트리거(Trigger) 활용

```sql
-- 예제 1: 재고 감소 시 최소 재고량 검증
CREATE OR REPLACE TRIGGER trg_재고검증
BEFORE UPDATE OF 재고수량 ON 상품
FOR EACH ROW
BEGIN
    IF :NEW.재고수량 < :NEW.최소재고량 THEN
        RAISE_APPLICATION_ERROR(-20001, '재고가 최소 재고량보다 적습니다.');
    END IF;
END;
/

-- 예제 2: 급여 변경 이력 자동 기록
CREATE OR REPLACE TRIGGER trg_급여이력
AFTER UPDATE OF 급여 ON 직원
FOR EACH ROW
BEGIN
    INSERT INTO 급여이력 (직원ID, 변경전급여, 변경후급여, 변경일시)
    VALUES (:OLD.직원ID, :OLD.급여, :NEW.급여, SYSDATE);
END;
/

-- 예제 3: 업무 시간 외 데이터 변경 방지
CREATE OR REPLACE TRIGGER trg_업무시간체크
BEFORE INSERT OR UPDATE OR DELETE ON 중요테이블
BEGIN
    IF TO_CHAR(SYSDATE, 'HH24') NOT BETWEEN 9 AND 18 OR
       TO_CHAR(SYSDATE, 'DY') IN ('토', '일') THEN
        RAISE_APPLICATION_ERROR(-20002, '업무 시간(평일 09:00~18:00)에만 수정 가능합니다.');
    END IF;
END;
/
```

#### 저장 프로시저로 무결성 보장

```sql
CREATE OR REPLACE PROCEDURE sp_주문등록 (
    p_고객ID IN NUMBER,
    p_상품ID IN NUMBER,
    p_수량 IN NUMBER,
    p_주문번호 OUT NUMBER
)
IS
    v_재고수량 NUMBER;
    v_고객등급 VARCHAR2(10);
    v_최대주문수량 NUMBER;
BEGIN
    -- 1. 재고 확인
    SELECT 재고수량 INTO v_재고수량
    FROM 상품
    WHERE 상품ID = p_상품ID
    FOR UPDATE;  -- 동시성 제어
    
    IF v_재고수량 < p_수량 THEN
        RAISE_APPLICATION_ERROR(-20003, '재고가 부족합니다.');
    END IF;
    
    -- 2. 고객 등급별 최대 주문 수량 확인
    SELECT 고객등급 INTO v_고객등급
    FROM 고객
    WHERE 고객ID = p_고객ID;
    
    v_최대주문수량 := CASE v_고객등급
        WHEN 'VIP' THEN 1000
        WHEN 'GOLD' THEN 500
        ELSE 100
    END;
    
    IF p_수량 > v_최대주문수량 THEN
        RAISE_APPLICATION_ERROR(-20004, '주문 수량이 고객 등급 한도를 초과합니다.');
    END IF;
    
    -- 3. 주문 등록
    INSERT INTO 주문 (주문번호, 고객ID, 상품ID, 수량, 주문일시)
    VALUES (주문번호_SEQ.NEXTVAL, p_고객ID, p_상품ID, p_수량, SYSDATE)
    RETURNING 주문번호 INTO p_주문번호;
    
    -- 4. 재고 차감
    UPDATE 상품
    SET 재고수량 = 재고수량 - p_수량
    WHERE 상품ID = p_상품ID;
    
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
/
```

---

## 3. 인덱스 설계

### 3.1 인덱스 종류와 특징

| 인덱스 유형 | 구조 | 적합한 상황 | 부적합한 상황 |
|-----------|------|-----------|--------------|
| **B-Tree** | 균형 트리 | 범위 검색, 정렬, 대부분의 경우 | 카디널리티 매우 낮을 때 |
| **Bitmap** | 비트맵 | 카디널리티 낮음, DW, 복합 조건 | OLTP, DML 빈번 |
| **Function-Based** | 함수 결과 | UPPER(), SUBSTR() 등 함수 사용 | 함수 미사용 |
| **Unique** | 중복 불가 | PK, UK 자동 생성 | 중복 값 존재 |
| **Composite** | 여러 컬럼 | 복합 조건 검색 | 단일 컬럼 검색 |

#### B-Tree 인덱스 구조

```
                    [50]
                   /    \
              [30]        [70]
             /    \      /    \
         [10,20] [40] [60]  [80,90]
            ↓     ↓    ↓      ↓
         RowID  RowID RowID  RowID
```

**특징**:
- 루트 → 브랜치 → 리프 노드 (모두 같은 깊이)
- 리프 노드는 이중 연결 리스트 (범위 검색 효율적)
- 검색 시간: O(log N)

---

### 3.2 인덱스 설계 원칙

#### 1. 선택도(Selectivity) 기준

```
선택도 = 고유값 개수 / 전체 행 개수

- 선택도 > 0.05 (5% 이상) : 인덱스 효과 좋음 ✅
- 선택도 < 0.01 (1% 이하) : 인덱스 효과 낮음 ⚠️
```

| 컬럼 | 전체 행 | 고유값 | 선택도 | 인덱스 권장 |
|------|--------|-------|--------|------------|
| 주민등록번호 | 1,000,000 | 1,000,000 | 1.0 | ✅ 매우 좋음 |
| 이메일 | 1,000,000 | 980,000 | 0.98 | ✅ 매우 좋음 |
| 전화번호 | 1,000,000 | 850,000 | 0.85 | ✅ 좋음 |
| 생년월일 | 1,000,000 | 30,000 | 0.03 | ⚠️ 보통 |
| 성별 | 1,000,000 | 2 | 0.000002 | ❌ 비효율 |
| 도시코드 | 1,000,000 | 50 | 0.00005 | ❌ 비효율 |

#### 2. 복합 인덱스 컬럼 순서 (중요!)

**원칙**: 선택도 높은 컬럼 → 선택도 낮은 컬럼

```sql
-- ❌ 잘못된 순서
CREATE INDEX idx_bad ON 주문(주문상태, 고객ID, 주문일자);
-- 주문상태 (5가지) → 선택도 낮음

-- ✅ 올바른 순서
CREATE INDEX idx_good ON 주문(고객ID, 주문일자, 주문상태);
-- 고객ID (고유) → 주문일자 (많음) → 주문상태 (적음)
```

**예외**: WHERE 절에서 '=' 조건이 먼저 오는 컬럼을 앞에

```sql
-- 쿼리: WHERE 주문상태 = '완료' AND 주문일자 BETWEEN ...
-- 인덱스: (주문상태, 주문일자) ✅
-- 이유: '=' 조건으로 범위를 먼저 좁힌 후, BETWEEN 처리
```

#### 3. 인덱스 생성 기준 (WJOS 원칙)

| 항목 | 설명 | 예시 |
|------|------|------|
| **W**HERE | WHERE 절 조건 | WHERE 고객ID = 100 |
| **J**OIN | JOIN 조건 (FK) | JOIN ON 주문.고객ID = 고객.고객ID |
| **O**RDER BY | 정렬 조건 | ORDER BY 주문일자 DESC |
| **S**ELECT | 커버링 인덱스 | SELECT 주문번호, 주문일자 (인덱스에 포함) |

💡 암기: "**위**기 **조**작 **오**리 **세**일" (WJOS)

---

### 3.3 커버링 인덱스 (Covering Index)

#### 개념
쿼리에 필요한 모든 컬럼을 인덱스에 포함시켜 **테이블 액세스 없이** 인덱스만으로 결과 반환

#### 예제

```sql
-- 쿼리
SELECT 주문번호, 주문일자, 주문금액
FROM 주문
WHERE 고객ID = 12345
ORDER BY 주문일자 DESC;

-- ❌ 일반 인덱스
CREATE INDEX idx_주문_고객ID ON 주문(고객ID);
-- 결과: 인덱스 스캔 → 테이블 액세스 (주문일자, 주문금액 조회)

-- ✅ 커버링 인덱스
CREATE INDEX idx_주문_커버링 ON 주문(고객ID, 주문일자 DESC, 주문금액);
-- 결과: 인덱스만 스캔 → 테이블 액세스 없음!
```

#### 성능 비교

| 방식 | I/O | 성능 |
|------|-----|------|
| 테이블 풀 스캔 | 10,000 블록 | 100% |
| 일반 인덱스 + 테이블 액세스 | 50 + 50 = 100 블록 | 10배 빠름 |
| 커버링 인덱스 | 50 블록 | 20배 빠름 |

---

### 3.4 인덱스 설계 안티패턴

| 안티패턴 | 문제점 | 해결 방법 |
|---------|--------|----------|
| **과도한 인덱스** | DML 성능 저하, 저장공간 낭비 | 사용하지 않는 인덱스 삭제 |
| **중복 인덱스** | (컬럼A), (컬럼A, 컬럼B) 동시 존재 | 단일 컬럼 인덱스 삭제 |
| **넓은 인덱스** | 5개 이상 컬럼 포함 | 필수 컬럼만 선별 |
| **함수 미사용** | WHERE UPPER(컬럼) 에 일반 인덱스 | Function-Based Index 생성 |
| **선택도 무시** | 성별, 상태코드 등에 인덱스 | Bitmap Index 또는 삭제 |

---

### 3.5 인덱스 관리

#### 인덱스 재구성 (Rebuild)

```sql
-- 단편화 확인
SELECT INDEX_NAME, BLEVEL, LEAF_BLOCKS, NUM_ROWS, 
       (DEL_LF_ROWS / NULLIF(LF_ROWS, 0)) * 100 AS 단편화율
FROM USER_INDEXES
WHERE TABLE_NAME = '주문';

-- 단편화율 30% 이상 시 재구성
ALTER INDEX idx_주문_고객ID REBUILD ONLINE;
-- ONLINE 옵션: 재구성 중에도 DML 가능
```

#### 사용되지 않는 인덱스 찾기

```sql
-- Oracle
SELECT INDEX_NAME, TABLE_NAME
FROM USER_INDEXES
WHERE INDEX_NAME NOT IN (
    SELECT INDEX_NAME
    FROM V$OBJECT_USAGE
    WHERE USED = 'TRUE'
);

-- 모니터링 시작
ALTER INDEX idx_주문_상태 MONITORING USAGE;

-- 일정 기간 후 확인
SELECT * FROM V$OBJECT_USAGE WHERE INDEX_NAME = 'IDX_주문_상태';
```

---

## 4. 분산 설계

### 4.1 데이터 분산 전략 비교

| 전략 | 설명 | 장점 | 단점 | 사용 사례 |
|------|------|------|------|-----------|
| **수평 분할** | 같은 스키마, 다른 서버 | 확장성 우수 | 크로스 샤드 쿼리 복잡 | 대용량 사용자 데이터 |
| **수직 분할** | 컬럼 기준 분리 | 접근 패턴 최적화 | 조인 필요 | 자주/덜 사용 컬럼 분리 |
| **복제** | 동일 데이터 복사 | 읽기 성능 향상 | 쓰기 복잡도 증가 | 읽기 위주 서비스 |

---

### 4.2 샤딩(Sharding) 상세

#### 샤딩 방식 비교

| 방식 | 샤드 결정 방법 | 장점 | 단점 | 적합한 경우 |
|------|---------------|------|------|------------|
| **Range** | 범위 (1~100만: S1) | 구현 간단, 범위 검색 쉬움 | 핫스팟 발생 가능 | 시계열 데이터 |
| **Hash** | 해시(키) mod N | 균등 분산 | 범위 검색 어려움 | 균등 분산 필요 |
| **Directory** | 조회 테이블 | 유연성 최고 | SPOF, 복잡도 증가 | 동적 샤드 재배치 |
| **Geo** | 지역 | 지연 시간 최소화 | 지역별 불균형 | 글로벌 서비스 |

#### Range-Based Sharding 예제

```
사용자 ID 범위로 샤딩:

Shard 1: 1 ~ 1,000,000
Shard 2: 1,000,001 ~ 2,000,000
Shard 3: 2,000,001 ~ 3,000,000

장점:
✅ 구현 간단
✅ 사용자 ID로 범위 조회 쉬움

단점:
⚠️ 최근 가입 사용자가 Shard 3에 집중
⚠️ 핫스팟 발생 (부하 불균형)
```

#### Hash-Based Sharding 예제

```python
# 샤드 결정 로직
def get_shard(user_id, num_shards=4):
    return hash(user_id) % num_shards

# 예시
get_shard(12345) → 1
get_shard(67890) → 3
get_shard(11111) → 2

장점:
✅ 균등 분산
✅ 핫스팟 없음

단점:
⚠️ 범위 검색 불가 (모든 샤드 조회 필요)
⚠️ 샤드 추가 시 데이터 재배치 필요 (Consistent Hashing으로 해결)
```

#### Consistent Hashing

```
일반 해시:
- 샤드 4개 → 5개 증가 시 80% 데이터 재배치

Consistent Hashing:
- 가상 노드 사용
- 샤드 추가 시 평균 1/N 데이터만 재배치
- Netflix, Cassandra 사용
```

---

### 4.3 복제(Replication) 전략

#### Master-Slave 복제

```
[Master] ─────┐
 (Write)      │
              ├─→ [Slave 1] (Read)
              ├─→ [Slave 2] (Read)
              └─→ [Slave 3] (Read)

장점:
✅ 읽기 부하 분산 (1 : N)
✅ 백업 자동화
✅ 장애 복구 (Slave → Master 승격)

단점:
⚠️ 복제 지연 (Replication Lag)
⚠️ Master가 SPOF
```

**복제 지연 처리 방법**:

| 방법 | 설명 | 사용 시나리오 |
|------|------|--------------|
| **동기 복제** | Master 커밋 후 Slave 확인 대기 | 강한 일관성 필요 (금융) |
| **비동기 복제** | Master 즉시 커밋, Slave 비동기 | 성능 우선 (소셜 미디어) |
| **반동기 복제** | 최소 1개 Slave만 확인 | 절충안 |
| **Sticky Session** | 같은 사용자는 같은 Slave | 세션 일관성 |

#### Master-Master 복제

```
[Master 1] ←──→ [Master 2]
(Region A)      (Region B)

장점:
✅ 양방향 쓰기 가능
✅ 지역별 지연 시간 최소화
✅ 고가용성

단점:
⚠️ 충돌 해결 복잡 (Conflict Resolution)
⚠️ 순환 복제 주의
```

**충돌 해결 전략**:
- Last Write Wins (LWW): 최신 타임스탬프 우선
- Version Vector: 버전 벡터로 인과 관계 추적
- Application-Level Resolution: 애플리케이션에서 처리

---

### 4.4 분산 트랜잭션

#### 2단계 커밋 (2PC: Two-Phase Commit)

```
[Coordinator]
     │
     ├─→ [Participant 1]
     ├─→ [Participant 2]
     └─→ [Participant 3]

Phase 1: Prepare (준비)
  Coordinator → All: "커밋 가능?"
  All → Coordinator: "예" 또는 "아니오"

Phase 2: Commit (커밋)
  Coordinator → All: "커밋" (모두 예) 또는 "롤백" (하나라도 아니오)
  All: 실행 및 완료 응답
```

| 장점 | 단점 |
|------|------|
| ✅ 강한 일관성 보장 | ⚠️ Coordinator가 SPOF |
| ✅ ACID 트랜잭션 | ⚠️ 블로킹 프로토콜 (성능 저하) |
| | ⚠️ 네트워크 장애 시 데드락 |

#### Saga 패턴

```
주문 프로세스 예제:

Saga 1: 주문 생성 → 성공
  보상: 주문 취소

Saga 2: 재고 차감 → 성공
  보상: 재고 복구

Saga 3: 결제 처리 → 실패!
  → Saga 2 보상 실행 (재고 복구)
  → Saga 1 보상 실행 (주문 취소)
```

| 장점 | 단점 |
|------|------|
| ✅ 비블로킹 (성능 우수) | ⚠️ 최종 일관성 (Eventual Consistency) |
| ✅ 확장성 좋음 | ⚠️ 보상 트랜잭션 구현 복잡 |
| ✅ 마이크로서비스 적합 | ⚠️ 격리 수준 낮음 |

**Saga 구현 방식**:

| 방식 | 설명 | 도구 |
|------|------|------|
| **Choreography** | 각 서비스가 이벤트 발행/구독 | Kafka, RabbitMQ |
| **Orchestration** | 중앙 조정자가 순서 제어 | Temporal, Camunda |

---

## 5. 보안 설계

### 5.1 접근 제어 계층

```
계층적 접근 제어 구조:

Level 1: 시스템 수준 (OS 인증)
    └─→ Level 2: 데이터베이스 수준 (DB 계정)
            └─→ Level 3: 스키마 수준 (스키마 소유권)
                    └─→ Level 4: 객체 수준 (테이블, 뷰)
                            └─→ Level 5: 행 수준 (RLS)
                                    └─→ Level 6: 컬럼 수준 (VPD)
```

---

### 5.2 권한 설계 (RBAC)

#### 역할 기반 접근 제어 예제

```sql
-- 1. 역할 생성
CREATE ROLE 영업사원;
CREATE ROLE 영업관리자;
CREATE ROLE DBA;

-- 2. 역할에 권한 부여
GRANT SELECT, INSERT ON 고객 TO 영업사원;
GRANT SELECT ON 주문 TO 영업사원;

GRANT SELECT, INSERT, UPDATE, DELETE ON 고객 TO 영업관리자;
GRANT SELECT, UPDATE ON 주문 TO 영업관리자;

GRANT ALL PRIVILEGES TO DBA;

-- 3. 사용자에게 역할 할당
GRANT 영업사원 TO 홍길동;
GRANT 영업관리자 TO 김철수;

-- 4. 역할 활성화
SET ROLE 영업사원;
```

#### 권한 매트릭스

| 역할 \ 객체 | 고객 | 주문 | 상품 | 재고 | 급여 |
|-----------|------|------|------|------|------|
| 영업사원 | R, C | R | R | - | - |
| 영업관리자 | All | R, U | R, U | R | - |
| 재고담당 | R | R | R, U | All | - |
| 인사담당 | R | - | - | - | All |
| DBA | All | All | All | All | All |

(R: Read, C: Create, U: Update, D: Delete, All: CRUD)

---

### 5.3 행 수준 보안 (RLS: Row-Level Security)

#### Oracle VPD (Virtual Private Database)

```sql
-- 1. 보안 정책 함수 생성
CREATE OR REPLACE FUNCTION fn_영업사원_정책 (
    p_schema VARCHAR2,
    p_object VARCHAR2
)
RETURN VARCHAR2
IS
    v_predicate VARCHAR2(2000);
    v_사원번호 NUMBER;
BEGIN
    -- 현재 로그인 사용자의 사원번호 조회
    SELECT 사원번호 INTO v_사원번호
    FROM 사원
    WHERE 사용자ID = USER;
    
    -- 자신이 담당하는 고객만 조회 가능
    v_predicate := '담당사원번호 = ' || v_사원번호;
    
    RETURN v_predicate;
END;
/

-- 2. 정책 적용
BEGIN
    DBMS_RLS.ADD_POLICY (
        object_schema   => 'HR',
        object_name     => '고객',
        policy_name     => '영업사원_고객_정책',
        function_schema => 'HR',
        policy_function => 'fn_영업사원_정책',
        statement_types => 'SELECT, INSERT, UPDATE, DELETE'
    );
END;
/

-- 결과: 영업사원이 "SELECT * FROM 고객"을 실행하면
-- 자동으로 "WHERE 담당사원번호 = [자신의 번호]"가 추가됨
```

#### PostgreSQL RLS

```sql
-- 1. RLS 활성화
ALTER TABLE 고객 ENABLE ROW LEVEL SECURITY;

-- 2. 정책 생성
CREATE POLICY 영업사원_고객_정책 ON 고객
    FOR ALL
    TO 영업사원_role
    USING (담당사원번호 = current_setting('app.current_사원번호')::INTEGER);

-- 3. 세션 변수 설정
SET app.current_사원번호 = 12345;
```

---

### 5.4 데이터 암호화

#### 암호화 계층

| 계층 | 방법 | 도구 | 보호 대상 |
|------|------|------|-----------|
| **애플리케이션** | 컬럼 암호화 | AES-256, RSA | 특정 민감 컬럼 |
| **데이터베이스** | TDE | Oracle TDE, SQL Server TDE | 전체 DB 파일 |
| **파일 시스템** | OS 암호화 | LUKS, BitLocker | 디스크 |
| **네트워크** | SSL/TLS | TLS 1.3 | 전송 데이터 |

#### TDE (Transparent Data Encryption) 예제

```sql
-- Oracle TDE 설정

-- 1. Wallet 생성
ALTER SYSTEM SET ENCRYPTION KEY IDENTIFIED BY "복잡한비밀번호";

-- 2. 테이블스페이스 암호화
CREATE TABLESPACE secure_ts
    DATAFILE '/data/secure01.dbf' SIZE 100M
    ENCRYPTION USING 'AES256'
    DEFAULT STORAGE(ENCRYPT);

-- 3. 기존 테이블 암호화
ALTER TABLE 고객 MOVE TABLESPACE secure_ts;

-- 4. 특정 컬럼만 암호화
ALTER TABLE 고객 MODIFY (주민번호 ENCRYPT USING 'AES256');
```

#### 컬럼 암호화 (Application-Level)

```sql
-- 암호화 함수 생성
CREATE OR REPLACE FUNCTION fn_암호화(p_원본 VARCHAR2)
RETURN RAW
IS
    v_키 RAW(32) := UTL_RAW.CAST_TO_RAW('MySecretKey12345678901234567890');
BEGIN
    RETURN DBMS_CRYPTO.ENCRYPT(
        src => UTL_RAW.CAST_TO_RAW(p_원본),
        typ => DBMS_CRYPTO.ENCRYPT_AES256 + DBMS_CRYPTO.CHAIN_CBC + DBMS_CRYPTO.PAD_PKCS5,
        key => v_키
    );
END;
/

-- 복호화 함수
CREATE OR REPLACE FUNCTION fn_복호화(p_암호문 RAW)
RETURN VARCHAR2
IS
    v_키 RAW(32) := UTL_RAW.CAST_TO_RAW('MySecretKey12345678901234567890');
BEGIN
    RETURN UTL_RAW.CAST_TO_VARCHAR2(
        DBMS_CRYPTO.DECRYPT(
            src => p_암호문,
            typ => DBMS_CRYPTO.ENCRYPT_AES256 + DBMS_CRYPTO.CHAIN_CBC + DBMS_CRYPTO.PAD_PKCS5,
            key => v_키
        )
    );
END;
/

-- 사용 예제
INSERT INTO 고객 (고객ID, 고객명, 주민번호_암호화)
VALUES (1, '홍길동', fn_암호화('123456-1234567'));

SELECT 고객명, fn_복호화(주민번호_암호화) AS 주민번호
FROM 고객
WHERE 고객ID = 1;
```

---

### 5.5 감사(Auditing)

#### 감사 유형

| 유형 | 대상 | 예시 | 목적 |
|------|------|------|------|
| **Statement** | SQL 문 유형 | SELECT, INSERT, DELETE | 일반 모니터링 |
| **Privilege** | 권한 사용 | GRANT, REVOKE | 권한 변경 추적 |
| **Object** | 특정 객체 | 급여 테이블 | 민감 데이터 접근 |
| **Fine-Grained** | 조건부 상세 | 급여 > 1억원 조회 | 세밀한 감사 |

#### 감사 설정 예제

```sql
-- 1. Statement Auditing
AUDIT SELECT TABLE, INSERT TABLE, DELETE TABLE BY ACCESS;

-- 2. Object Auditing
AUDIT SELECT, UPDATE, DELETE ON 급여 BY ACCESS;

-- 3. Privilege Auditing
AUDIT CREATE TABLE, DROP TABLE BY ACCESS;

-- 4. Fine-Grained Auditing (FGA)
BEGIN
    DBMS_FGA.ADD_POLICY (
        object_schema   => 'HR',
        object_name     => '급여',
        policy_name     => '고액급여_접근_감사',
        audit_condition => '급여금액 > 100000000',
        audit_column    => '급여금액',
        handler_schema  => 'HR',
        handler_module  => 'pkg_알림.고액급여_알림',
        enable          => TRUE
    );
END;
/

-- 5. 감사 로그 조회
SELECT USERNAME, OBJ_NAME, ACTION_NAME, TIMESTAMP
FROM DBA_AUDIT_TRAIL
WHERE OBJ_NAME = '급여'
ORDER BY TIMESTAMP DESC;

-- 6. FGA 로그 조회
SELECT DB_USER, OBJECT_NAME, SQL_TEXT, TIMESTAMP
FROM DBA_FGA_AUDIT_TRAIL
WHERE POLICY_NAME = '고액급여_접근_감사'
ORDER BY TIMESTAMP DESC;
```

---

### 5.6 SQL 인젝션 방어

#### 취약한 코드 vs 안전한 코드

```java
// ❌ 취약한 코드 (SQL Injection 가능)
String userId = request.getParameter("userId");
String sql = "SELECT * FROM 사용자 WHERE 사용자ID = '" + userId + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);

// 공격 예시:
// userId = "admin' OR '1'='1"
// 실행되는 SQL: SELECT * FROM 사용자 WHERE 사용자ID = 'admin' OR '1'='1'
// 결과: 모든 사용자 조회됨!

// ✅ 안전한 코드 (PreparedStatement)
String userId = request.getParameter("userId");
String sql = "SELECT * FROM 사용자 WHERE 사용자ID = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, userId);
ResultSet rs = pstmt.executeQuery();

// 공격 시도:
// userId = "admin' OR '1'='1"
// 실행되는 SQL: SELECT * FROM 사용자 WHERE 사용자ID = 'admin'' OR ''1''=''1'
// 결과: 문자열로 처리되어 공격 실패
```

#### 입력 검증

```java
// 화이트리스트 방식 검증
public boolean isValidUserId(String userId) {
    // 영문, 숫자, 언더스코어만 허용, 6~20자
    return userId.matches("^[a-zA-Z0-9_]{6,20}$");
}

// 블랙리스트 방식 (권장하지 않음)
public String sanitize(String input) {
    return input.replaceAll("[';\"\\-\\-]", "");
    // 문제: 우회 가능 (e.g., char(39) = ')
}
```

---

## 6. 시험 대비 핵심 요약

### 6.1 저장공간 설계 체크리스트

- [ ] 테이블스페이스 분리 (SUTI 원칙)
- [ ] 파티셔닝 전략 선택 (Range/List/Hash/Composite)
- [ ] 블록 크기 결정 (OLTP: 4~8KB, OLAP: 16~32KB)
- [ ] PCTFREE/PCTUSED 설정
- [ ] Row Migration 방지 설계

### 6.2 무결성 설계 체크리스트

- [ ] 도메인 무결성 (NOT NULL, CHECK, DEFAULT)
- [ ] 개체 무결성 (PK 설정, 대리키 vs 자연키)
- [ ] 참조 무결성 (FK, CASCADE 옵션)
- [ ] 외래키 인덱스 생성 ✅
- [ ] 트리거/프로시저로 비즈니스 규칙 구현

### 6.3 인덱스 설계 체크리스트

- [ ] WJOS 원칙 (WHERE, JOIN, ORDER BY, SELECT)
- [ ] 선택도 5% 이상인 컬럼에 생성
- [ ] 복합 인덱스 컬럼 순서 (선택도 높은 순)
- [ ] 커버링 인덱스 고려
- [ ] 사용하지 않는 인덱스 삭제
- [ ] 주기적 재구성 (단편화율 30% 이상)

### 6.4 분산 설계 체크리스트

- [ ] 샤딩 방식 선택 (Range/Hash/Directory)
- [ ] 샤딩 키 선정 (균등 분산, 크로스 샤드 최소화)
- [ ] 복제 전략 (Master-Slave, Master-Master)
- [ ] 분산 트랜잭션 방법 (2PC vs Saga)
- [ ] 복제 지연 처리 방안

### 6.5 보안 설계 체크리스트

- [ ] 최소 권한 원칙 (RBAC)
- [ ] 행 수준 보안 (RLS/VPD)
- [ ] 데이터 암호화 (TDE, 컬럼 암호화)
- [ ] 전송 암호화 (SSL/TLS)
- [ ] 감사 로그 설정
- [ ] SQL 인젝션 방어 (PreparedStatement)

---

## 7. DAP 시험 빈출 문제 유형

### 유형 1: 시나리오 기반 설계

**문제 예시**:
> 온라인 쇼핑몰 주문 테이블이 일 평균 10만 건씩 증가하고 있습니다. 최근 6개월 데이터는 자주 조회되지만 1년 이상 된 데이터는 거의 조회되지 않습니다. 적절한 저장공간 설계 방법은?

**정답 접근**:
1. Range 파티셔닝 (월별 또는 분기별)
2. 압축 옵션 (1년 이상 파티션)
3. 테이블스페이스 분리 (핫 데이터 vs 콜드 데이터)

### 유형 2: 무결성 제약 조건

**문제 예시**:
> 주문 테이블과 고객 테이블이 있습니다. 고객이 삭제되면 해당 고객의 주문 정보는 유지하되, 고객ID는 NULL로 변경해야 합니다. 적절한 참조 무결성 옵션은?

**정답**: `ON DELETE SET NULL`

### 유형 3: 인덱스 설계

**문제 예시**:
> 다음 쿼리의 성능을 최적화하기 위한 인덱스 설계는?
> ```sql
> SELECT 주문번호, 주문일자, 주문금액
> FROM 주문
> WHERE 고객ID = 12345 AND 주문상태 = '완료'
> ORDER BY 주문일자 DESC;
> ```

**정답**: `CREATE INDEX idx ON 주문(고객ID, 주문상태, 주문일자 DESC, 주문금액);`
- 커버링 인덱스로 테이블 액세스 제거
- WHERE 조건 컬럼을 앞에 배치
- ORDER BY 컬럼 포함

---

## 8. 암기 도구

### 저장공간 (SUTI)
**수**능 **유**니폼 **테**스트 **인**증서
- System, User, Temporary, Index

### 무결성 (DERU)
**도**깨비 **엔**젤 **참**새 **유**니콘
- 도메인, 엔티티(개체), 참조, 유저정의

### 인덱스 (WJOS)
**위**기 **조**작 **오**리 **세**일
- WHERE, JOIN, ORDER BY, SELECT

### 파티셔닝 (RLHC)
**리**얼 **리**스트 **해**시 **컴**포지트
- Range, List, Hash, Composite

### CASCADE 옵션 (CSDN)
**캐**스케이드 **셋**널 **디**폴트 **노**액션
- CASCADE, SET NULL, SET DEFAULT, NO ACTION

---

## 9. 실전 문제 풀이 전략

### 1단계: 요구사항 파악
- 데이터 특성 (OLTP vs OLAP)
- 데이터 증가율
- 조회 패턴
- 보안 요구사항

### 2단계: 설계 원칙 적용
- 정규화 vs 반정규화
- 파티셔닝 필요성
- 인덱스 전략
- 분산 필요성

### 3단계: 트레이드오프 고려
- 성능 vs 일관성
- 복잡도 vs 확장성
- 비용 vs 효과

### 4단계: 검증
- 예상 성능 계산
- 장애 시나리오 검토
- 유지보수 용이성

---

## 10. 참고 자료

### 공식 문서
- Oracle Database Documentation
- PostgreSQL Documentation
- MySQL Reference Manual

### 권장 도서
- "Database System Concepts" - Silberschatz
- "Designing Data-Intensive Applications" - Martin Kleppmann

### 온라인 리소스
- DB-Engines Ranking
- Use The Index, Luke!

---

**최종 점검 포인트** ✅
1. 각 설계 영역의 트레이드오프를 이해하고 있는가?
2. 실무 시나리오에 적용할 수 있는가?
3. 성능과 무결성의 균형을 잡을 수 있는가?
4. 암기 도구를 활용하여 핵심 개념을 기억하는가?

**DAP 시험 합격을 기원합니다!** 🎓