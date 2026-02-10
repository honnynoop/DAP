# 온라인 쇼핑몰 통합 데이터 아키텍처 구축 프로젝트
## DAP 방법론 적용 실무 사례

---

## 문서 정보

| 항목 | 내용 |
|------|------|
| **프로젝트명** | 온라인 쇼핑몰 통합 데이터 아키텍처 구축 |
| **발주처** | (주)코리아쇼핑 |
| **수행기간** | 2024.03.01 ~ 2024.08.31 (6개월) |
| **작성자** | 데이터아키텍처팀 |
| **작성일** | 2024.08.31 |
| **문서버전** | 1.0 |

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [현황 분석 (As-Is)](#2-현황-분석-as-is)
3. [요구사항 정의](#3-요구사항-정의)
4. [데이터 표준 정의](#4-데이터-표준-정의)
5. [데이터 아키텍처 설계](#5-데이터-아키텍처-설계)
6. [데이터 모델링](#6-데이터-모델링)
7. [데이터 흐름 및 인터페이스](#7-데이터-흐름-및-인터페이스)
8. [데이터 품질 관리](#8-데이터-품질-관리)
9. [메타데이터 관리](#9-메타데이터-관리)
10. [이행 계획](#10-이행-계획)
11. [산출물 목록](#11-산출물-목록)

---

## 1. 프로젝트 개요

### 1.1 프로젝트 배경

**코리아쇼핑**은 온라인 쇼핑몰을 운영하는 이커머스 기업으로, 다음과 같은 데이터 관리 문제에 직면했습니다:

#### 주요 문제점

| 문제 영역 | 현황 | 영향 |
|----------|------|------|
| **데이터 분산** | 5개 시스템에 데이터 분산 저장 | 데이터 일관성 저하 |
| **표준 부재** | 시스템별 상이한 명명 규칙 | 통합 및 분석 어려움 |
| **중복 데이터** | 고객, 상품 데이터 30% 중복 | 저장공간 낭비, 품질 저하 |
| **실시간 분석 불가** | 배치 기반 데이터 처리 | 의사결정 지연 |
| **레거시 시스템** | 10년 이상 된 주문 시스템 | 유지보수 비용 증가 |

### 1.2 프로젝트 목표

```
비전: 데이터 기반 의사결정이 가능한 통합 데이터 플랫폼 구축

전략 목표:
├─ 데이터 통합
│  └─ 5개 분산 시스템의 데이터 통합
├─ 데이터 표준화
│  └─ 전사 데이터 표준 수립 및 적용
├─ 데이터 품질 향상
│  └─ 데이터 품질 90% 이상 달성
└─ 실시간 분석 기반 마련
   └─ 실시간 데이터 처리 아키텍처 구축
```

#### 정량적 목표

| KPI | 현재 | 목표 | 개선율 |
|-----|------|------|--------|
| 데이터 정합성 | 65% | 95% | +30%p |
| 데이터 중복률 | 30% | 5% | -25%p |
| 분석 리드타임 | 24시간 | 1시간 | -95% |
| 시스템 통합도 | 20% | 90% | +70%p |
| 표준 준수율 | 0% | 95% | +95%p |

### 1.3 프로젝트 범위

#### 대상 시스템

```
┌─────────────────────────────────────────┐
│         통합 대상 시스템                 │
├─────────────────────────────────────────┤
│ 1. 고객관리시스템 (CRM)                  │
│ 2. 주문관리시스템 (OMS)                  │
│ 3. 재고관리시스템 (WMS)                  │
│ 4. 배송관리시스템 (DMS)                  │
│ 5. 정산관리시스템 (Settlement)           │
└─────────────────────────────────────────┘
```

#### 데이터 영역

| 데이터 영역 | 포함 대상 | 우선순위 |
|-----------|----------|---------|
| **고객** | 회원정보, 주소, 결제수단 | 높음 |
| **상품** | 상품정보, 카테고리, 재고 | 높음 |
| **주문** | 주문, 주문상세, 배송 | 높음 |
| **정산** | 매출, 정산, 수수료 | 중간 |
| **마케팅** | 프로모션, 쿠폰, 이벤트 | 중간 |
| **CS** | 문의, 리뷰, 반품/환불 | 낮음 |

### 1.4 프로젝트 조직

| 역할 | 담당자 | 책임 |
|------|--------|------|
| **PM** | 김철수 | 프로젝트 총괄 |
| **DA** | 이영희 | 데이터 아키텍처 설계 |
| **데이터 모델러** | 박지훈, 최민정 | 데이터 모델링 |
| **DBA** | 강동욱 | 데이터베이스 관리 |
| **데이터 엔지니어** | 정수진, 한상민 | ETL 개발 |
| **품질 관리자** | 윤서연 | 데이터 품질 관리 |

---

## 2. 현황 분석 (As-Is)

### 2.1 시스템 현황

#### 2.1.1 시스템 구성도

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   CRM        │   │   OMS        │   │   WMS        │
│  (Oracle)    │   │  (MySQL)     │   │  (PostgreSQL)│
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
              ┌───────────▼───────────┐
              │  Point-to-Point       │
              │  File Transfer        │
              │  (FTP, CSV)           │
              └───────────┬───────────┘
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
┌──────▼───────┐   ┌──────▼───────┐   ┌──────▼───────┐
│   DMS        │   │  Settlement  │   │   Legacy     │
│  (MySQL)     │   │  (Oracle)    │   │   System     │
└──────────────┘   └──────────────┘   └──────────────┘
```

#### 2.1.2 시스템별 상세 현황

**1) 고객관리시스템 (CRM)**

| 항목 | 내용 |
|------|------|
| **DBMS** | Oracle 11g |
| **테이블 수** | 45개 |
| **데이터 용량** | 120GB |
| **주요 데이터** | 회원, 주소, 포인트, 등급 |
| **문제점** | 비표준 테이블명 (member_info, user_addr 등) |

**2) 주문관리시스템 (OMS)**

| 항목 | 내용 |
|------|------|
| **DBMS** | MySQL 5.7 |
| **테이블 수** | 32개 |
| **데이터 용량** | 250GB |
| **주요 데이터** | 주문, 주문상세, 결제 |
| **문제점** | 주문번호 체계 불일치, 날짜 형식 혼재 |

**3) 재고관리시스템 (WMS)**

| 항목 | 내용 |
|------|------|
| **DBMS** | PostgreSQL 10 |
| **테이블 수** | 28개 |
| **데이터 용량** | 80GB |
| **주요 데이터** | 상품, 재고, 입출고 |
| **문제점** | 상품코드 불일치, 재고 데이터 지연 업데이트 |

### 2.2 데이터 현황 분석

#### 2.2.1 데이터 품질 현황

| 품질 지표 | 측정값 | 평가 | 주요 문제 |
|----------|--------|------|----------|
| **정합성** | 65% | 미흡 | 시스템 간 고객번호 불일치 |
| **완전성** | 72% | 보통 | 필수 항목 NULL 다수 (주소 20%) |
| **유일성** | 68% | 미흡 | 중복 고객 데이터 15% |
| **정확성** | 78% | 보통 | 전화번호 형식 오류 12% |
| **최신성** | 60% | 미흡 | 배치 처리로 인한 지연 |

#### 2.2.2 중복 데이터 분석

**고객 데이터 중복 사례**

```sql
-- CRM 시스템
CREATE TABLE member_info (
    mem_no VARCHAR(10),
    mem_name VARCHAR(50),
    email VARCHAR(100),
    phone VARCHAR(20)
);

-- OMS 시스템
CREATE TABLE customer (
    cust_id NUMBER(10),
    customer_name VARCHAR(100),
    email_addr VARCHAR(150),
    tel_no VARCHAR(15)
);
```

**중복 현황**
- 동일 고객이 CRM과 OMS에 별도 저장
- 고객번호 체계 상이: CRM(문자), OMS(숫자)
- 30%의 고객 정보가 양쪽 시스템에 중복 존재

#### 2.2.3 데이터 표준 부재 문제

**시스템별 명명 규칙 불일치**

| 의미 | CRM | OMS | WMS |
|------|-----|-----|-----|
| 고객ID | mem_no | cust_id | customer_no |
| 고객명 | mem_name | customer_name | cust_nm |
| 주문일자 | ord_date | order_dt | order_date |
| 상품코드 | prod_cd | product_code | item_cd |

### 2.3 데이터 흐름 분석

#### 2.3.1 현재 데이터 흐름도

```
고객 주문
    ↓
┌─────────────────┐
│   OMS (주문)    │
└────────┬────────┘
         │ FTP (CSV)
         │ Daily Batch
         ↓
┌─────────────────┐      FTP      ┌─────────────────┐
│   WMS (재고)    │ ←──────────── │   상품 마스터    │
└────────┬────────┘               └─────────────────┘
         │ FTP
         │ Daily
         ↓
┌─────────────────┐
│   DMS (배송)    │
└────────┬────────┘
         │ FTP
         │ Daily
         ↓
┌─────────────────┐
│  Settlement     │
│   (정산)        │
└─────────────────┘
```

**문제점**
- Point-to-Point 연계로 복잡도 증가
- 파일 기반 배치 처리로 실시간성 부족
- 데이터 변환 로직 중복
- 연계 실패 시 추적 어려움

### 2.4 As-Is 분석 요약

#### 핵심 문제점

| 번호 | 문제점 | 영향도 | 우선순위 |
|------|--------|--------|---------|
| 1 | 데이터 표준 부재 | 상 | 1 |
| 2 | 시스템 간 데이터 중복 | 상 | 2 |
| 3 | 데이터 품질 저하 | 상 | 3 |
| 4 | 실시간 연계 불가 | 중 | 4 |
| 5 | Point-to-Point 복잡도 | 중 | 5 |

---

## 3. 요구사항 정의

### 3.1 업무 요구사항

#### 3.1.1 핵심 업무 요구사항

| ID | 요구사항 | 상세 내용 | 우선순위 |
|----|---------|----------|---------|
| **BR-001** | 360도 고객뷰 | 모든 시스템의 고객 데이터 통합 조회 | 높음 |
| **BR-002** | 실시간 재고 조회 | 주문 시 실시간 재고 확인 | 높음 |
| **BR-003** | 통합 주문 추적 | 주문~배송~정산 전 과정 추적 | 높음 |
| **BR-004** | 상품 마스터 관리 | 단일 상품 마스터 관리 | 중간 |
| **BR-005** | 매출 분석 | 실시간 매출 현황 분석 | 중간 |

#### 3.1.2 비기능 요구사항

| 유형 | 요구사항 | 목표 수치 |
|------|---------|----------|
| **성능** | 조회 응답시간 | 2초 이내 |
| **성능** | 일 처리량 | 100만 건/일 |
| **가용성** | 시스템 가동률 | 99.9% |
| **확장성** | 데이터 증가 대응 | 연 50% 증가 수용 |
| **보안** | 개인정보 암호화 | 주민번호, 카드번호 암호화 |

### 3.2 데이터 요구사항

#### 3.2.1 데이터 통합 요구사항

```
데이터 통합 전략
├─ 마스터 데이터 통합
│  ├─ 고객 마스터: CRM을 마스터로 지정
│  ├─ 상품 마스터: WMS를 마스터로 지정
│  └─ 코드 마스터: 신규 통합 코드 체계 수립
├─ 트랜잭션 데이터 통합
│  ├─ 주문 데이터: OMS 기준
│  ├─ 배송 데이터: DMS 기준
│  └─ 정산 데이터: Settlement 기준
└─ 이력 데이터 관리
   └─ 모든 변경 이력 저장 (3년)
```

#### 3.2.2 데이터 품질 요구사항

| 품질 차원 | 목표 | 측정 방법 |
|----------|------|----------|
| **정합성** | 95% 이상 | 참조 무결성 위반 건수 |
| **완전성** | 95% 이상 | 필수 항목 NULL 비율 |
| **유일성** | 99% 이상 | 중복 데이터 비율 |
| **정확성** | 95% 이상 | 형식 오류 비율 |
| **최신성** | 1시간 이내 | 데이터 갱신 주기 |

---

## 4. 데이터 표준 정의

### 4.1 표준 단어 사전

#### 4.1.1 엔티티 단어

| 표준단어 | 영문명 | 영문약어 | 정의 | 금칙어 |
|---------|--------|---------|------|--------|
| 고객 | CUSTOMER | CUST | 상품을 구매하는 개인 또는 법인 | 회원, 사용자 |
| 주문 | ORDER | ORD | 고객이 상품 구매를 요청하는 행위 | 발주 |
| 상품 | PRODUCT | PROD | 판매 대상이 되는 물품 | 제품, 아이템 |
| 배송 | DELIVERY | DLVY | 상품을 고객에게 전달하는 과정 | 택배 |
| 결제 | PAYMENT | PAY | 대금을 지불하는 행위 | 지불 |
| 정산 | SETTLEMENT | SETTL | 거래 대금을 정리하는 행위 | 결산 |
| 재고 | INVENTORY | INV | 보유 상품의 수량 | 스톡 |
| 쿠폰 | COUPON | CPN | 할인 혜택 증서 | - |
| 리뷰 | REVIEW | RVW | 상품 사용 후기 | 평가 |
| 반품 | RETURN | RTN | 구매 상품을 되돌려 보내는 행위 | 환불 |

#### 4.1.2 속성 단어

| 표준단어 | 영문명 | 영문약어 | 정의 |
|---------|--------|---------|------|
| ID | ID | ID | 식별자 |
| 번호 | NUMBER | NO | 일련번호 |
| 명 | NAME | NM | 이름 |
| 코드 | CODE | CD | 분류 기호 |
| 일자 | DATE | DT | 날짜 |
| 일시 | DATETIME | DTM | 날짜와 시간 |
| 금액 | AMOUNT | AMT | 화폐 가치 |
| 수량 | QUANTITY | QTY | 개수 |
| 단가 | UNIT_PRICE | UPR | 단위당 가격 |
| 율 | RATE | RT | 비율 |
| 여부 | WHETHER | YN | Y/N 구분 |
| 구분 | DIVISION | DIV | 종류 구분 |
| 상태 | STATUS | STAT | 진행 상태 |
| 유형 | TYPE | TYP | 분류 유형 |
| 주소 | ADDRESS | ADDR | 위치 |
| 설명 | DESCRIPTION | DESC | 상세 설명 |

#### 4.1.3 수식 단어

| 표준단어 | 영문명 | 영문약어 | 사용 예 |
|---------|--------|---------|---------|
| 총 | TOTAL | TOT | 총주문금액 |
| 실제 | ACTUAL | ACT | 실제배송일자 |
| 예정 | SCHEDULED | SCHD | 예정배송일자 |
| 최초 | FIRST | FST | 최초등록일시 |
| 최종 | LAST | LST | 최종변경일시 |
| 평균 | AVERAGE | AVG | 평균주문금액 |
| 최대 | MAXIMUM | MAX | 최대할인율 |
| 최소 | MINIMUM | MIN | 최소주문수량 |

### 4.2 표준 도메인 정의

| 도메인명 | 데이터타입 | 길이 | NULL | 기본값 | 설명 |
|---------|-----------|------|------|--------|------|
| ID | VARCHAR2 | 20 | N | - | 식별자 |
| 번호 | VARCHAR2 | 30 | Y | - | 일련번호 |
| 명_100 | VARCHAR2 | 100 | Y | - | 이름 (100자) |
| 명_200 | VARCHAR2 | 200 | Y | - | 이름 (200자) |
| 코드_2 | VARCHAR2 | 2 | Y | - | 2자리 코드 |
| 코드_4 | VARCHAR2 | 4 | Y | - | 4자리 코드 |
| 코드_10 | VARCHAR2 | 10 | Y | - | 10자리 코드 |
| 일자 | DATE | - | Y | - | 날짜 |
| 일시 | TIMESTAMP | - | Y | SYSDATE | 날짜시간 |
| 금액 | NUMBER | 15,2 | Y | 0 | 화폐금액 |
| 수량 | NUMBER | 10,2 | Y | 0 | 수량 |
| 단가 | NUMBER | 15,2 | Y | 0 | 단가 |
| 비율 | NUMBER | 5,2 | Y | 0 | 백분율 |
| 여부 | CHAR | 1 | N | 'Y' | Y/N |
| 주소_200 | VARCHAR2 | 200 | Y | - | 주소 |
| 주소_500 | VARCHAR2 | 500 | Y | - | 상세주소 |
| 전화번호 | VARCHAR2 | 20 | Y | - | 전화번호 |
| 이메일 | VARCHAR2 | 100 | Y | - | 이메일 |
| URL | VARCHAR2 | 500 | Y | - | URL |
| 설명_1000 | VARCHAR2 | 1000 | Y | - | 설명 |
| 텍스트 | CLOB | - | Y | - | 대용량 텍스트 |
| 암호화 | VARCHAR2 | 256 | Y | - | 암호화 데이터 |

### 4.3 표준 용어 사전

#### 4.3.1 고객 영역 표준 용어

| 용어명 | 영문명 | 영문약어 | 도메인 | 정의 |
|--------|--------|---------|--------|------|
| 고객ID | CUSTOMER_ID | CUST_ID | ID | 고객 식별자 (마스터키) |
| 고객명 | CUSTOMER_NAME | CUST_NM | 명_100 | 고객 이름 |
| 고객등급코드 | CUSTOMER_GRADE_CODE | CUST_GRD_CD | 코드_2 | 고객 등급 (01:일반, 02:우수, 03:VIP) |
| 생년월일 | BIRTH_DATE | BRTH_DT | 일자 | 생년월일 |
| 성별코드 | GENDER_CODE | GNDR_CD | 코드_1 | 성별 (M:남, F:여) |
| 휴대전화번호 | MOBILE_PHONE_NUMBER | MBL_PHN_NO | 전화번호 | 휴대전화번호 |
| 이메일주소 | EMAIL_ADDRESS | EMAIL_ADDR | 이메일 | 이메일 주소 |
| 우편번호 | POSTAL_CODE | POST_CD | 코드_10 | 우편번호 |
| 기본주소 | BASE_ADDRESS | BASE_ADDR | 주소_200 | 기본 주소 |
| 상세주소 | DETAIL_ADDRESS | DTL_ADDR | 주소_200 | 상세 주소 |
| 가입일자 | JOIN_DATE | JOIN_DT | 일자 | 회원 가입일 |
| 탈퇴일자 | WITHDRAWAL_DATE | WTHDR_DT | 일자 | 회원 탈퇴일 |
| 탈퇴여부 | WITHDRAWAL_WHETHER | WTHDR_YN | 여부 | 탈퇴 여부 |

#### 4.3.2 주문 영역 표준 용어

| 용어명 | 영문명 | 영문약어 | 도메인 | 정의 |
|--------|--------|---------|--------|------|
| 주문ID | ORDER_ID | ORD_ID | ID | 주문 식별자 |
| 주문번호 | ORDER_NUMBER | ORD_NO | 번호 | 주문번호 (표시용) |
| 고객ID | CUSTOMER_ID | CUST_ID | ID | 주문 고객 |
| 주문일시 | ORDER_DATETIME | ORD_DTM | 일시 | 주문 일시 |
| 주문상태코드 | ORDER_STATUS_CODE | ORD_STAT_CD | 코드_2 | 주문 상태 |
| 총주문금액 | TOTAL_ORDER_AMOUNT | TOT_ORD_AMT | 금액 | 총 주문금액 |
| 총할인금액 | TOTAL_DISCOUNT_AMOUNT | TOT_DISC_AMT | 금액 | 총 할인금액 |
| 배송비 | DELIVERY_FEE | DLVY_FEE | 금액 | 배송비 |
| 최종결제금액 | FINAL_PAYMENT_AMOUNT | FNL_PAY_AMT | 금액 | 최종 결제금액 |
| 결제수단코드 | PAYMENT_METHOD_CODE | PAY_MTHD_CD | 코드_2 | 결제수단 |
| 배송지명 | DELIVERY_NAME | DLVY_NM | 명_100 | 배송지 이름 |
| 배송지주소 | DELIVERY_ADDRESS | DLVY_ADDR | 주소_200 | 배송지 주소 |
| 배송메시지 | DELIVERY_MESSAGE | DLVY_MSG | 설명_1000 | 배송 메시지 |

#### 4.3.3 상품 영역 표준 용어

| 용어명 | 영문명 | 영문약어 | 도메인 | 정의 |
|--------|--------|---------|--------|------|
| 상품ID | PRODUCT_ID | PROD_ID | ID | 상품 식별자 |
| 상품코드 | PRODUCT_CODE | PROD_CD | 코드_10 | 상품코드 |
| 상품명 | PRODUCT_NAME | PROD_NM | 명_200 | 상품명 |
| 카테고리코드 | CATEGORY_CODE | CTGR_CD | 코드_10 | 카테고리 코드 |
| 판매가격 | SALE_PRICE | SALE_PRC | 금액 | 판매 가격 |
| 정상가격 | REGULAR_PRICE | REG_PRC | 금액 | 정상 가격 |
| 재고수량 | INVENTORY_QUANTITY | INV_QTY | 수량 | 재고 수량 |
| 안전재고수량 | SAFE_INVENTORY_QUANTITY | SAFE_INV_QTY | 수량 | 안전 재고 |
| 판매상태코드 | SALE_STATUS_CODE | SALE_STAT_CD | 코드_2 | 판매 상태 |
| 상품설명 | PRODUCT_DESCRIPTION | PROD_DESC | 텍스트 | 상품 설명 |
| 상품이미지URL | PRODUCT_IMAGE_URL | PROD_IMG_URL | URL | 상품 이미지 |

### 4.4 표준 코드 체계

#### 4.4.1 코드 분류 체계

```
표준 코드
├─ 공통 코드 (시스템 전체 사용)
│  ├─ 성별코드 (M, F)
│  ├─ 사용여부 (Y, N)
│  └─ 삭제여부 (Y, N)
├─ 업무 코드 (업무 영역별 사용)
│  ├─ 고객등급코드 (01, 02, 03)
│  ├─ 주문상태코드 (01~07)
│  ├─ 결제수단코드 (01~05)
│  └─ 배송상태코드 (01~05)
└─ 마스터 코드 (참조 마스터)
   ├─ 지역코드
   ├─ 카테고리코드
   └─ 배송사코드
```

#### 4.4.2 주요 코드 정의

**고객등급코드 (CUSTOMER_GRADE_CODE)**
| 코드값 | 코드명 | 정의 | 혜택 |
|--------|--------|------|------|
| 01 | 일반 | 신규 가입 고객 | 기본 적립 1% |
| 02 | 우수 | 연 100만원 이상 구매 | 적립 2%, 무료배송 |
| 03 | VIP | 연 500만원 이상 구매 | 적립 3%, 무료배송, 특별할인 |

**주문상태코드 (ORDER_STATUS_CODE)**
| 코드값 | 코드명 | 정의 | 다음상태 |
|--------|--------|------|---------|
| 01 | 주문접수 | 주문이 접수됨 | 02 |
| 02 | 결제완료 | 결제가 완료됨 | 03 |
| 03 | 상품준비 | 배송을 위해 상품 준비 중 | 04 |
| 04 | 배송중 | 배송이 진행 중 | 05 |
| 05 | 배송완료 | 배송이 완료됨 | - |
| 06 | 주문취소 | 주문이 취소됨 | - |
| 07 | 환불완료 | 환불이 완료됨 | - |

**결제수단코드 (PAYMENT_METHOD_CODE)**
| 코드값 | 코드명 | 정의 | 수수료율 |
|--------|--------|------|---------|
| 01 | 신용카드 | 신용카드 결제 | 3.5% |
| 02 | 계좌이체 | 실시간 계좌이체 | 2.0% |
| 03 | 무통장입금 | 가상계좌 입금 | 1.5% |
| 04 | 간편결제 | 카카오페이, 네이버페이 등 | 3.0% |
| 05 | 포인트 | 적립 포인트 사용 | 0% |

**배송상태코드 (DELIVERY_STATUS_CODE)**
| 코드값 | 코드명 | 정의 |
|--------|--------|------|
| 01 | 배송준비 | 배송 준비 중 |
| 02 | 집하완료 | 택배사 집하 완료 |
| 03 | 배송중 | 배송 진행 중 |
| 04 | 배송완료 | 수령 완료 |
| 05 | 배송보류 | 배송 일시 보류 |

### 4.5 명명 규칙

#### 4.5.1 테이블 명명 규칙

```
형식: [접두어]_[업무영역]_[엔티티명]

접두어:
- TB: 일반 테이블
- TH: 이력 테이블
- TT: 임시 테이블
- TM: 마스터 테이블
- TC: 코드 테이블

예시:
- TB_CUST_고객
- TB_ORD_주문
- TB_ORD_주문상세
- TH_CUST_고객_202402
- TM_PROD_상품
- TC_공통코드
```

#### 4.5.2 컬럼 명명 규칙

```
형식: [수식어]_[대표단어]_[분류어]_[속성어]

예시:
- 고객ID
- 고객명
- 총주문금액
- 최종변경일시
- 예정배송일자
```

#### 4.5.3 인덱스 및 제약조건 명명 규칙

```
Primary Key: 테이블명_PK
Foreign Key: 테이블명_FK_순번
Index: 테이블명_IX_순번
Unique: 테이블명_UK_순번
Check: 테이블명_CK_순번

예시:
- TB_CUST_고객_PK
- TB_ORD_주문_FK_01
- TB_ORD_주문_IX_01
- TB_CUST_고객_UK_01
- TB_ORD_주문_CK_01
```

---

## 5. 데이터 아키텍처 설계

### 5.1 목표 아키텍처 (To-Be)

#### 5.1.1 전체 아키텍처

```
┌────────────────────────────────────────────────────────┐
│                  Presentation Layer                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │   Web    │  │  Mobile  │  │  Admin   │            │
│  └──────────┘  └──────────┘  └──────────┘            │
└────────────────────────┬───────────────────────────────┘
                         │ REST API
┌────────────────────────▼───────────────────────────────┐
│                 Application Layer                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐      │
│  │    CRM     │  │    OMS     │  │    WMS     │      │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘      │
└─────────┼────────────────┼────────────────┼────────────┘
          │                │                │
┌─────────▼────────────────▼────────────────▼────────────┐
│            Integration Layer (API Gateway)             │
│  ┌──────────────────────────────────────────────┐     │
│  │         Event Bus (Kafka)                    │     │
│  └──────────────────────────────────────────────┘     │
└────────────────────────┬───────────────────────────────┘
                         │
┌────────────────────────▼───────────────────────────────┐
│                   Data Layer                           │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │ Master Data  │  │ Transaction  │  │   History   │ │
│  │   (Oracle)   │  │    (MySQL)   │  │  (Archive)  │ │
│  └──────────────┘  └──────────────┘  └─────────────┘ │
│  ┌──────────────────────────────────────────────────┐ │
│  │         Data Warehouse & Data Lake               │ │
│  │              (PostgreSQL + S3)                   │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

#### 5.1.2 데이터 계층 구조

```
데이터 계층
├─ 운영 데이터 계층 (Operational Data Layer)
│  ├─ 마스터 데이터 (Master Data)
│  │  ├─ 고객 마스터
│  │  ├─ 상품 마스터
│  │  └─ 코드 마스터
│  └─ 트랜잭션 데이터 (Transaction Data)
│     ├─ 주문
│     ├─ 결제
│     └─ 배송
│
├─ 통합 데이터 계층 (Integration Data Layer)
│  ├─ ODS (Operational Data Store)
│  │  └─ 실시간 통합 데이터
│  └─ 이력 데이터
│     └─ 월별 파티션 이력
│
└─ 분석 데이터 계층 (Analytics Data Layer)
   ├─ Data Warehouse
   │  ├─ 매출 Fact
   │  ├─ 재고 Fact
   │  └─ 고객 Dimension
   └─ Data Lake
      └─ 로그, 비정형 데이터
```

### 5.2 데이터 배치 전략

#### 5.2.1 데이터베이스 배치

| 데이터베이스 | 용도 | DBMS | 용량 | 비고 |
|------------|------|------|------|------|
| **MASTER_DB** | 마스터 데이터 | Oracle 19c | 200GB | 고객, 상품, 코드 |
| **TRANS_DB** | 트랜잭션 데이터 | MySQL 8.0 | 500GB | 주문, 결제, 배송 |
| **HISTORY_DB** | 이력 데이터 | PostgreSQL 13 | 1TB | 월별 파티션 |
| **DW_DB** | 데이터웨어하우스 | PostgreSQL 13 | 800GB | 분석용 |
| **ARCHIVE_DB** | 아카이브 | S3 | 무제한 | 3년 이상 데이터 |

#### 5.2.2 데이터 분산 전략

**샤딩 전략**
```
주문 데이터 샤딩 (주문일자 기준)
├─ TRANS_DB_01: 2024년 1~6월
├─ TRANS_DB_02: 2024년 7~12월
└─ TRANS_DB_03: 2025년 1~6월

고객 데이터 샤딩 (고객ID 기준)
├─ MASTER_DB_01: 고객ID 0~4로 시작
└─ MASTER_DB_02: 고객ID 5~9로 시작
```

### 5.3 데이터 통합 전략

#### 5.3.1 마스터 데이터 통합

**고객 마스터 통합 방안**
```
통합 전략: CRM을 Master System으로 지정

프로세스:
1. CRM → 고객 마스터 등록/수정
2. CDC로 변경사항 캡처
3. Kafka로 이벤트 발행
4. 각 시스템 구독 및 동기화
5. OMS, WMS에서 고객ID 참조
```

**상품 마스터 통합 방안**
```
통합 전략: WMS를 Master System으로 지정

프로세스:
1. WMS → 상품 마스터 등록/수정
2. 실시간 동기화 (Kafka)
3. OMS에서 상품 정보 참조
4. 재고 정보 실시간 연동
```

#### 5.3.2 데이터 동기화 방식

| 데이터 유형 | 동기화 방식 | 주기 | 도구 |
|-----------|-----------|------|------|
| **마스터 데이터** | CDC (Change Data Capture) | 실시간 | Debezium |
| **주문 데이터** | Event-Driven | 실시간 | Kafka |
| **재고 데이터** | API 호출 | 실시간 | REST API |
| **정산 데이터** | Batch | 일 1회 | Airflow |
| **분석 데이터** | ETL | 일 1회 | Airflow |

### 5.4 데이터 보안 아키텍처

#### 5.4.1 데이터 암호화 전략

```
암호화 대상
├─ 저장 암호화 (At Rest)
│  ├─ 주민등록번호: AES-256
│  ├─ 카드번호: AES-256
│  ├─ 계좌번호: AES-256
│  └─ 비밀번호: SHA-256 + Salt
│
├─ 전송 암호화 (In Transit)
│  ├─ HTTPS (TLS 1.3)
│  └─ Database 연결: SSL
│
└─ 컬럼 레벨 암호화
   └─ 개인정보 컬럼 별도 암호화
```

#### 5.4.2 접근 제어

| 역할 | 권한 | 대상 |
|------|------|------|
| **개발자** | SELECT | 개발DB (마스킹 데이터) |
| **운영자** | SELECT, INSERT, UPDATE | 운영DB |
| **DBA** | ALL | 전체 DB |
| **분석가** | SELECT | DW_DB (비식별 데이터) |
| **관리자** | 로그 조회 | 접근 로그 |

---

## 6. 데이터 모델링

### 6.1 개념 데이터 모델 (CDM)

#### 6.1.1 핵심 엔티티 식별

```
[고객] ──────< [주문] >────── [상품]
   │             │
   │             │
   └─< [주소]    └─< [주문상세]
                      │
                      ├─< [배송]
                      └─< [결제]
```

#### 6.1.2 주요 엔티티 정의

| 엔티티 | 정의 | 주요 속성 |
|--------|------|----------|
| **고객** | 상품을 구매하는 개인 또는 법인 | 고객ID, 고객명, 등급, 연락처 |
| **주문** | 고객의 상품 구매 요청 | 주문ID, 주문일시, 상태, 금액 |
| **주문상세** | 주문의 상품별 상세 내역 | 주문ID, 상품ID, 수량, 금액 |
| **상품** | 판매 대상 물품 | 상품ID, 상품명, 가격, 재고 |
| **배송** | 주문 상품의 배송 정보 | 배송ID, 주문ID, 배송지, 상태 |
| **결제** | 주문 대금 지불 정보 | 결제ID, 주문ID, 결제수단, 금액 |

### 6.2 논리 데이터 모델 (LDM)

#### 6.2.1 고객 영역 논리 모델

**고객 (TB_CUST_고객)**
```
PK: 고객ID
───────────────────────────
고객ID (VARCHAR2(20))
고객명 (VARCHAR2(100))
고객등급코드 (VARCHAR2(2))
생년월일 (DATE)
성별코드 (CHAR(1))
휴대전화번호 (VARCHAR2(20))
이메일주소 (VARCHAR2(100))
가입일자 (DATE)
탈퇴일자 (DATE)
탈퇴여부 (CHAR(1))
적립포인트 (NUMBER(10))
최초등록일시 (TIMESTAMP)
최종변경일시 (TIMESTAMP)
```

**고객주소 (TB_CUST_고객주소)**
```
PK: 고객주소ID
FK: 고객ID → TB_CUST_고객
───────────────────────────
고객주소ID (VARCHAR2(20))
고객ID (VARCHAR2(20))
주소유형코드 (VARCHAR2(2))
우편번호 (VARCHAR2(10))
기본주소 (VARCHAR2(200))
상세주소 (VARCHAR2(200))
기본주소여부 (CHAR(1))
최초등록일시 (TIMESTAMP)
최종변경일시 (TIMESTAMP)
```

#### 6.2.2 주문 영역 논리 모델

**주문 (TB_ORD_주문)**
```
PK: 주문ID
FK: 고객ID → TB_CUST_고객
───────────────────────────
주문ID (VARCHAR2(20))
주문번호 (VARCHAR2(30))
고객ID (VARCHAR2(20))
주문일시 (TIMESTAMP)
주문상태코드 (VARCHAR2(2))
총주문금액 (NUMBER(15,2))
총할인금액 (NUMBER(15,2))
쿠폰할인금액 (NUMBER(15,2))
포인트사용금액 (NUMBER(15,2))
배송비 (NUMBER(15,2))
최종결제금액 (NUMBER(15,2))
주문자명 (VARCHAR2(100))
주문자전화번호 (VARCHAR2(20))
배송지명 (VARCHAR2(100))
배송지우편번호 (VARCHAR2(10))
배송지주소 (VARCHAR2(200))
배송지상세주소 (VARCHAR2(200))
배송메시지 (VARCHAR2(1000))
최초등록일시 (TIMESTAMP)
최종변경일시 (TIMESTAMP)
```

**주문상세 (TB_ORD_주문상세)**
```
PK: 주문상세ID
FK1: 주문ID → TB_ORD_주문
FK2: 상품ID → TB_PROD_상품
───────────────────────────
주문상세ID (VARCHAR2(20))
주문ID (VARCHAR2(20))
상품ID (VARCHAR2(20))
상품명 (VARCHAR2(200))
주문수량 (NUMBER(10,2))
상품단가 (NUMBER(15,2))
할인율 (NUMBER(5,2))
할인금액 (NUMBER(15,2))
최종금액 (NUMBER(15,2))
주문상세상태코드 (VARCHAR2(2))
최초등록일시 (TIMESTAMP)
최종변경일시 (TIMESTAMP)
```

#### 6.2.3 상품 영역 논리 모델

**상품 (TB_PROD_상품)**
```
PK: 상품ID
FK: 카테고리코드 → TC_카테고리코드
───────────────────────────
상품ID (VARCHAR2(20))
상품코드 (VARCHAR2(10))
상품명 (VARCHAR2(200))
카테고리코드 (VARCHAR2(10))
브랜드명 (VARCHAR2(100))
정상가격 (NUMBER(15,2))
판매가격 (NUMBER(15,2))
할인율 (NUMBER(5,2))
재고수량 (NUMBER(10,2))
안전재고수량 (NUMBER(10,2))
판매상태코드 (VARCHAR2(2))
상품설명 (CLOB)
대표이미지URL (VARCHAR2(500))
판매시작일자 (DATE)
판매종료일자 (DATE)
등록일시 (TIMESTAMP)
최종변경일시 (TIMESTAMP)
```

**재고 (TB_PROD_재고)**
```
PK: 재고ID
FK: 상품ID → TB_PROD_상품
───────────────────────────
재고ID (VARCHAR2(20))
상품ID (VARCHAR2(20))
창고코드 (VARCHAR2(10))
가용재고수량 (NUMBER(10,2))
예약재고수량 (NUMBER(10,2))
불량재고수량 (NUMBER(10,2))
총재고수량 (NUMBER(10,2))
최종입고일자 (DATE)
최종출고일자 (DATE)
재고상태코드 (VARCHAR2(2))
최초등록일시 (TIMESTAMP)
최종변경일시 (TIMESTAMP)
```

#### 6.2.4 배송 영역 논리 모델

**배송 (TB_DLVY_배송)**
```
PK: 배송ID
FK: 주문ID → TB_ORD_주문
───────────────────────────
배송ID (VARCHAR2(20))
주문ID (VARCHAR2(20))
배송번호 (VARCHAR2(30))
배송사코드 (VARCHAR2(10))
송장번호 (VARCHAR2(30))
배송상태코드 (VARCHAR2(2))
수령인명 (VARCHAR2(100))
수령인전화번호 (VARCHAR2(20))
배송지우편번호 (VARCHAR2(10))
배송지주소 (VARCHAR2(200))
배송지상세주소 (VARCHAR2(200))
배송메시지 (VARCHAR2(1000))
출고일시 (TIMESTAMP)
배송시작일시 (TIMESTAMP)
배송완료일시 (TIMESTAMP)
최초등록일시 (TIMESTAMP)
최종변경일시 (TIMESTAMP)
```

#### 6.2.5 결제 영역 논리 모델

**결제 (TB_PAY_결제)**
```
PK: 결제ID
FK: 주문ID → TB_ORD_주문
───────────────────────────
결제ID (VARCHAR2(20))
주문ID (VARCHAR2(20))
결제번호 (VARCHAR2(30))
결제수단코드 (VARCHAR2(2))
결제금액 (NUMBER(15,2))
결제상태코드 (VARCHAR2(2))
PG사코드 (VARCHAR2(10))
PG승인번호 (VARCHAR2(50))
카드번호암호화 (VARCHAR2(256))
카드사코드 (VARCHAR2(10))
할부개월수 (NUMBER(2))
결제일시 (TIMESTAMP)
결제취소일시 (TIMESTAMP)
결제취소금액 (NUMBER(15,2))
최초등록일시 (TIMESTAMP)
최종변경일시 (TIMESTAMP)
```

### 6.3 물리 데이터 모델 (PDM)

#### 6.3.1 테이블 생성 DDL

**고객 테이블**
```sql
CREATE TABLE TB_CUST_고객 (
    고객ID VARCHAR2(20) NOT NULL,
    고객명 VARCHAR2(100) NOT NULL,
    고객등급코드 VARCHAR2(2) DEFAULT '01',
    생년월일 DATE,
    성별코드 CHAR(1),
    휴대전화번호 VARCHAR2(20),
    이메일주소 VARCHAR2(100),
    가입일자 DATE DEFAULT SYSDATE,
    탈퇴일자 DATE,
    탈퇴여부 CHAR(1) DEFAULT 'N',
    적립포인트 NUMBER(10) DEFAULT 0,
    최초등록일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    최종변경일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT TB_CUST_고객_PK PRIMARY KEY (고객ID),
    CONSTRAINT TB_CUST_고객_UK_01 UNIQUE (이메일주소),
    CONSTRAINT TB_CUST_고객_CK_01 CHECK (탈퇴여부 IN ('Y', 'N')),
    CONSTRAINT TB_CUST_고객_CK_02 CHECK (성별코드 IN ('M', 'F'))
);

-- 인덱스 생성
CREATE INDEX TB_CUST_고객_IX_01 ON TB_CUST_고객(고객명);
CREATE INDEX TB_CUST_고객_IX_02 ON TB_CUST_고객(가입일자);
CREATE INDEX TB_CUST_고객_IX_03 ON TB_CUST_고객(고객등급코드);

-- 코멘트
COMMENT ON TABLE TB_CUST_고객 IS '고객 마스터 테이블';
COMMENT ON COLUMN TB_CUST_고객.고객ID IS '고객 식별자 (PK)';
COMMENT ON COLUMN TB_CUST_고객.고객명 IS '고객 이름';
```

**주문 테이블**
```sql
CREATE TABLE TB_ORD_주문 (
    주문ID VARCHAR2(20) NOT NULL,
    주문번호 VARCHAR2(30) NOT NULL,
    고객ID VARCHAR2(20) NOT NULL,
    주문일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    주문상태코드 VARCHAR2(2) DEFAULT '01',
    총주문금액 NUMBER(15,2) DEFAULT 0,
    총할인금액 NUMBER(15,2) DEFAULT 0,
    쿠폰할인금액 NUMBER(15,2) DEFAULT 0,
    포인트사용금액 NUMBER(15,2) DEFAULT 0,
    배송비 NUMBER(15,2) DEFAULT 0,
    최종결제금액 NUMBER(15,2) DEFAULT 0,
    주문자명 VARCHAR2(100),
    주문자전화번호 VARCHAR2(20),
    배송지명 VARCHAR2(100),
    배송지우편번호 VARCHAR2(10),
    배송지주소 VARCHAR2(200),
    배송지상세주소 VARCHAR2(200),
    배송메시지 VARCHAR2(1000),
    최초등록일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    최종변경일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT TB_ORD_주문_PK PRIMARY KEY (주문ID),
    CONSTRAINT TB_ORD_주문_UK_01 UNIQUE (주문번호),
    CONSTRAINT TB_ORD_주문_FK_01 
        FOREIGN KEY (고객ID) REFERENCES TB_CUST_고객(고객ID),
    CONSTRAINT TB_ORD_주문_FK_02 
        FOREIGN KEY (주문상태코드) REFERENCES TC_공통코드(코드값)
);

-- 파티션 (주문일시 기준 월별)
CREATE TABLE TB_ORD_주문 (
    -- 컬럼 정의 동일
) PARTITION BY RANGE (주문일시) (
    PARTITION P_202401 VALUES LESS THAN (TO_DATE('2024-02-01', 'YYYY-MM-DD')),
    PARTITION P_202402 VALUES LESS THAN (TO_DATE('2024-03-01', 'YYYY-MM-DD')),
    PARTITION P_202403 VALUES LESS THAN (TO_DATE('2024-04-01', 'YYYY-MM-DD'))
);

-- 인덱스
CREATE INDEX TB_ORD_주문_IX_01 ON TB_ORD_주문(고객ID);
CREATE INDEX TB_ORD_주문_IX_02 ON TB_ORD_주문(주문일시);
CREATE INDEX TB_ORD_주문_IX_03 ON TB_ORD_주문(주문상태코드);
```

**주문상세 테이블**
```sql
CREATE TABLE TB_ORD_주문상세 (
    주문상세ID VARCHAR2(20) NOT NULL,
    주문ID VARCHAR2(20) NOT NULL,
    상품ID VARCHAR2(20) NOT NULL,
    상품명 VARCHAR2(200),
    주문수량 NUMBER(10,2) DEFAULT 1,
    상품단가 NUMBER(15,2) DEFAULT 0,
    할인율 NUMBER(5,2) DEFAULT 0,
    할인금액 NUMBER(15,2) DEFAULT 0,
    최종금액 NUMBER(15,2) DEFAULT 0,
    주문상세상태코드 VARCHAR2(2) DEFAULT '01',
    최초등록일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    최종변경일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT TB_ORD_주문상세_PK PRIMARY KEY (주문상세ID),
    CONSTRAINT TB_ORD_주문상세_FK_01 
        FOREIGN KEY (주문ID) REFERENCES TB_ORD_주문(주문ID),
    CONSTRAINT TB_ORD_주문상세_FK_02 
        FOREIGN KEY (상품ID) REFERENCES TB_PROD_상품(상품ID)
);

-- 인덱스
CREATE INDEX TB_ORD_주문상세_IX_01 ON TB_ORD_주문상세(주문ID);
CREATE INDEX TB_ORD_주문상세_IX_02 ON TB_ORD_주문상세(상품ID);
```

#### 6.3.2 이력 테이블 설계

**고객 이력 테이블**
```sql
CREATE TABLE TH_CUST_고객_202402 (
    이력ID VARCHAR2(20) NOT NULL,
    고객ID VARCHAR2(20) NOT NULL,
    변경일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    변경유형코드 VARCHAR2(2), -- 01:등록, 02:수정, 03:삭제
    변경전값 CLOB,
    변경후값 CLOB,
    변경자ID VARCHAR2(20),
    CONSTRAINT TH_CUST_고객_202402_PK PRIMARY KEY (이력ID)
);

-- 월별 파티션으로 관리
-- TH_CUST_고객_202401, TH_CUST_고객_202402, ...
```

#### 6.3.3 성능 최적화

**1) 인덱스 전략**
```sql
-- 단일 컬럼 인덱스
CREATE INDEX TB_ORD_주문_IX_01 ON TB_ORD_주문(고객ID);

-- 복합 인덱스 (주문일시 + 주문상태)
CREATE INDEX TB_ORD_주문_IX_04 
    ON TB_ORD_주문(주문일시, 주문상태코드);

-- 함수 기반 인덱스
CREATE INDEX TB_CUST_고객_IX_04 
    ON TB_CUST_고객(UPPER(고객명));
```

**2) 파티셔닝 전략**
```sql
-- Range 파티션 (주문일시 기준)
CREATE TABLE TB_ORD_주문
PARTITION BY RANGE (주문일시) (
    PARTITION P_202401 VALUES LESS THAN (TO_DATE('2024-02-01', 'YYYY-MM-DD')),
    PARTITION P_202402 VALUES LESS THAN (TO_DATE('2024-03-01', 'YYYY-MM-DD'))
);

-- List 파티션 (주문상태 기준)
CREATE TABLE TB_ORD_주문상태이력
PARTITION BY LIST (주문상태코드) (
    PARTITION P_진행중 VALUES ('01', '02', '03', '04'),
    PARTITION P_완료 VALUES ('05'),
    PARTITION P_취소 VALUES ('06', '07')
);
```

**3) 통계 정보 관리**
```sql
-- 통계 수집
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname => 'SHOP_DB',
        tabname => 'TB_ORD_주문',
        estimate_percent => 10,
        method_opt => 'FOR ALL COLUMNS SIZE AUTO'
    );
END;
/
```

### 6.4 ERD (Entity Relationship Diagram)

#### 6.4.1 전체 ERD 개념도

```
┌─────────────┐
│   고객       │
│ PK: 고객ID   │
└──────┬──────┘
       │ 1
       │
       │ N
┌──────▼──────┐     N  ┌─────────────┐  1   ┌─────────────┐
│   주문       │────────│  주문상세    │──────│   상품       │
│ PK: 주문ID   │        │ PK: 주문상세ID│      │ PK: 상품ID   │
│ FK: 고객ID   │        │ FK: 주문ID   │      └─────────────┘
└──────┬──────┘        │ FK: 상품ID   │
       │ 1             └─────────────┘
       │
       ├──────────┐
       │ N        │ N
┌──────▼──────┐ ┌▼─────────────┐
│   배송       │ │    결제       │
│ PK: 배송ID   │ │ PK: 결제ID    │
│ FK: 주문ID   │ │ FK: 주문ID    │
└─────────────┘ └──────────────┘
```

#### 6.4.2 주요 관계 (Relationship)

| 부모 엔티티 | 자식 엔티티 | 관계 | 카디널리티 | 설명 |
|-----------|-----------|------|-----------|------|
| 고객 | 주문 | 1:N | 한 고객은 여러 주문 가능 | 필수 |
| 주문 | 주문상세 | 1:N | 한 주문은 여러 상품 포함 | 필수 |
| 상품 | 주문상세 | 1:N | 한 상품은 여러 주문에 포함 | 필수 |
| 주문 | 배송 | 1:N | 한 주문은 여러 배송 가능 | 선택 |
| 주문 | 결제 | 1:N | 한 주문은 여러 결제 가능 | 필수 |

---

## 7. 데이터 흐름 및 인터페이스

### 7.1 데이터 흐름도 (DFD)

#### 7.1.1 주문 프로세스 데이터 흐름

```
[고객]
  │
  │ (1) 주문 요청
  ↓
[주문 시스템]
  │
  ├─→ (2) 고객 정보 조회 → [고객 마스터]
  ├─→ (3) 상품 정보 조회 → [상품 마스터]
  ├─→ (4) 재고 확인 → [재고 시스템]
  │
  │ (5) 주문 생성
  ↓
[주문 DB]
  │
  ├─→ (6) 주문 이벤트 발행 → [Kafka]
  │                            │
  │                            ├─→ [재고 시스템] (재고 차감)
  │                            ├─→ [결제 시스템] (결제 처리)
  │                            └─→ [배송 시스템] (배송 준비)
  │
  └─→ (7) 주문 확인 알림 → [알림 시스템] → [고객]
```

#### 7.1.2 실시간 재고 동기화 흐름

```
[재고 변경 발생]
       ↓
[WMS - 재고 테이블 UPDATE]
       ↓
[Debezium CDC 캡처]
       ↓
[Kafka Topic: inventory-changes]
       ↓
   ┌───┴───┐
   │       │
   ↓       ↓
[OMS]   [DW]
재고정보  실시간
동기화   분석
```

### 7.2 인터페이스 설계

#### 7.2.1 인터페이스 정의서

**IF-001: 고객 정보 동기화**

| 항목 | 내용 |
|------|------|
| **인터페이스 ID** | IF-001 |
| **인터페이스명** | 고객 정보 동기화 |
| **송신 시스템** | CRM |
| **수신 시스템** | OMS, WMS |
| **연계 방식** | Event-Driven (Kafka) |
| **실행 주기** | 실시간 (변경 발생 시) |
| **데이터 형식** | JSON |
| **SLA** | 5초 이내 |

**메시지 스키마**
```json
{
  "event_type": "customer.updated",
  "event_time": "2024-02-10T10:30:00Z",
  "customer_id": "CUST20240001",
  "data": {
    "customer_name": "홍길동",
    "customer_grade_code": "02",
    "email": "hong@example.com",
    "phone": "010-1234-5678"
  }
}
```

**IF-002: 주문 생성 이벤트**

| 항목 | 내용 |
|------|------|
| **인터페이스 ID** | IF-002 |
| **인터페이스명** | 주문 생성 이벤트 |
| **송신 시스템** | OMS |
| **수신 시스템** | WMS, 결제, 배송 |
| **연계 방식** | Event-Driven (Kafka) |
| **실행 주기** | 실시간 |
| **데이터 형식** | JSON |

**메시지 스키마**
```json
{
  "event_type": "order.created",
  "event_time": "2024-02-10T10:35:00Z",
  "order_id": "ORD20240210001",
  "data": {
    "customer_id": "CUST20240001",
    "total_amount": 50000,
    "order_items": [
      {
        "product_id": "PROD001",
        "quantity": 2,
        "price": 25000
      }
    ],
    "delivery_address": {
      "postal_code": "12345",
      "address": "서울시 강남구 테헤란로 123",
      "detail_address": "ABC빌딩 10층"
    }
  }
}
```

**IF-003: 재고 변경 이벤트**

| 항목 | 내용 |
|------|------|
| **인터페이스 ID** | IF-003 |
| **인터페이스명** | 재고 변경 이벤트 |
| **송신 시스템** | WMS |
| **수신 시스템** | OMS, DW |
| **연계 방식** | CDC (Debezium + Kafka) |
| **실행 주기** | 실시간 (변경 발생 시) |
| **데이터 형식** | JSON |

#### 7.2.2 API 인터페이스 명세

**API-001: 재고 조회 API**

```
Endpoint: GET /api/v1/inventory/{product_id}

Request:
- Path Parameter: product_id (상품ID)

Response:
{
  "product_id": "PROD001",
  "available_quantity": 100,
  "reserved_quantity": 20,
  "defective_quantity": 5,
  "total_quantity": 125,
  "last_updated_at": "2024-02-10T10:30:00Z"
}

Status Codes:
- 200: 성공
- 404: 상품 없음
- 500: 서버 오류
```

**API-002: 주문 생성 API**

```
Endpoint: POST /api/v1/orders

Request:
{
  "customer_id": "CUST20240001",
  "items": [
    {
      "product_id": "PROD001",
      "quantity": 2
    }
  ],
  "delivery_address": {
    "postal_code": "12345",
    "address": "서울시 강남구",
    "detail_address": "테헤란로 123"
  },
  "payment_method": "01"
}

Response:
{
  "order_id": "ORD20240210001",
  "order_number": "20240210-001",
  "total_amount": 50000,
  "status": "01",
  "created_at": "2024-02-10T10:35:00Z"
}

Status Codes:
- 201: 주문 생성 성공
- 400: 잘못된 요청
- 409: 재고 부족
- 500: 서버 오류
```

### 7.3 배치 인터페이스

#### 7.3.1 배치 작업 정의

**BATCH-001: 일일 매출 집계**

| 항목 | 내용 |
|------|------|
| **배치 ID** | BATCH-001 |
| **배치명** | 일일 매출 집계 |
| **실행 시각** | 매일 01:00 |
| **예상 소요시간** | 30분 |
| **소스** | TB_ORD_주문, TB_ORD_주문상세 |
| **타겟** | DW_매출집계 |

**배치 흐름**
```
1. 전일 주문 데이터 추출
   ↓
2. 매출 집계 계산
   - 총 주문 건수
   - 총 매출액
   - 평균 주문금액
   - 상품별 판매량
   ↓
3. DW 적재
   ↓
4. 집계 결과 알림
```

**SQL 예시**
```sql
INSERT INTO DW_매출집계 (
    집계일자,
    총주문건수,
    총매출액,
    평균주문금액,
    집계일시
)
SELECT 
    TRUNC(주문일시) AS 집계일자,
    COUNT(*) AS 총주문건수,
    SUM(최종결제금액) AS 총매출액,
    AVG(최종결제금액) AS 평균주문금액,
    SYSDATE AS 집계일시
FROM TB_ORD_주문
WHERE TRUNC(주문일시) = TRUNC(SYSDATE) - 1
  AND 주문상태코드 IN ('05') -- 배송완료
GROUP BY TRUNC(주문일시);
```

---

## 8. 데이터 품질 관리

### 8.1 데이터 품질 관리 체계

#### 8.1.1 품질 관리 조직

```
┌──────────────────────────────┐
│   데이터 품질 관리 위원회     │
│   (분기 1회 회의)             │
└────────────┬─────────────────┘
             │
┌────────────▼─────────────────┐
│   데이터 품질 관리자          │
│   (품질 정책 수립)            │
└────────────┬─────────────────┘
             │
┌────────────▼─────────────────┐
│   데이터 품질 검증팀          │
│   (품질 검증 및 개선)         │
└──────────────────────────────┘
```

#### 8.1.2 품질 관리 프로세스

```
1. 품질 기준 정의
   ├─ 품질 차원 정의
   ├─ 측정 지표 수립
   └─ 목표 수준 설정
        ↓
2. 품질 측정
   ├─ 자동 품질 검증
   ├─ 품질 점수 산출
   └─ 리포트 생성
        ↓
3. 품질 분석
   ├─ 문제 원인 분석
   ├─ 영향도 평가
   └─ 개선 방안 도출
        ↓
4. 품질 개선
   ├─ 개선 작업 실행
   ├─ 재측정
   └─ 효과 검증
        ↓
5. 모니터링
   └─ 지속적 품질 관리
```

### 8.2 데이터 품질 측정

#### 8.2.1 품질 차원별 측정 지표

**1) 정합성 (Consistency)**

| 측정 항목 | 측정 SQL | 목표 |
|----------|---------|------|
| 참조 무결성 위반 | 존재하지 않는 FK | 0건 |
| 시스템 간 데이터 불일치 | CRM vs OMS 고객 정보 | 5% 이하 |

```sql
-- 참조 무결성 검증
SELECT COUNT(*) AS 위반건수
FROM TB_ORD_주문 O
WHERE NOT EXISTS (
    SELECT 1 
    FROM TB_CUST_고객 C 
    WHERE C.고객ID = O.고객ID
);

-- 시스템 간 데이터 일치도
SELECT 
    ROUND((일치건수 / 전체건수) * 100, 2) AS 일치율
FROM (
    SELECT 
        COUNT(*) FILTER (WHERE CRM.고객명 = OMS.고객명) AS 일치건수,
        COUNT(*) AS 전체건수
    FROM CRM_고객 CRM
    JOIN OMS_고객 OMS ON CRM.고객ID = OMS.고객ID
);
```

**2) 완전성 (Completeness)**

| 측정 항목 | 측정 SQL | 목표 |
|----------|---------|------|
| 필수 항목 NULL | 고객명, 전화번호 등 | 5% 이하 |
| 데이터 누락 | 주문-결제 매칭 | 0건 |

```sql
-- 필수 항목 NULL 검증
SELECT 
    '고객명' AS 항목,
    COUNT(*) AS 전체건수,
    COUNT(*) FILTER (WHERE 고객명 IS NULL) AS NULL건수,
    ROUND(COUNT(*) FILTER (WHERE 고객명 IS NULL) / COUNT(*) * 100, 2) AS NULL비율
FROM TB_CUST_고객
UNION ALL
SELECT 
    '휴대전화번호' AS 항목,
    COUNT(*),
    COUNT(*) FILTER (WHERE 휴대전화번호 IS NULL),
    ROUND(COUNT(*) FILTER (WHERE 휴대전화번호 IS NULL) / COUNT(*) * 100, 2)
FROM TB_CUST_고객;
```

**3) 유일성 (Uniqueness)**

| 측정 항목 | 측정 SQL | 목표 |
|----------|---------|------|
| 중복 데이터 | 이메일 중복 | 1% 이하 |
| PK 중복 | PK 유일성 | 0건 |

```sql
-- 중복 데이터 검증
SELECT 
    이메일주소,
    COUNT(*) AS 중복건수
FROM TB_CUST_고객
WHERE 이메일주소 IS NOT NULL
GROUP BY 이메일주소
HAVING COUNT(*) > 1;

-- 중복률 계산
SELECT 
    ROUND((중복건수 / 전체건수) * 100, 2) AS 중복률
FROM (
    SELECT 
        COUNT(DISTINCT 이메일주소) AS 전체건수,
        SUM(CASE WHEN cnt > 1 THEN cnt - 1 ELSE 0 END) AS 중복건수
    FROM (
        SELECT 이메일주소, COUNT(*) AS cnt
        FROM TB_CUST_고객
        WHERE 이메일주소 IS NOT NULL
        GROUP BY 이메일주소
    )
);
```

**4) 정확성 (Accuracy)**

| 측정 항목 | 측정 SQL | 목표 |
|----------|---------|------|
| 형식 오류 | 전화번호, 이메일 형식 | 5% 이하 |
| 범위 오류 | 날짜, 금액 유효성 | 1% 이하 |

```sql
-- 전화번호 형식 검증
SELECT 
    COUNT(*) AS 전체건수,
    COUNT(*) FILTER (
        WHERE 휴대전화번호 IS NOT NULL 
        AND 휴대전화번호 NOT LIKE '010-____-____'
    ) AS 형식오류건수,
    ROUND(
        COUNT(*) FILTER (
            WHERE 휴대전화번호 IS NOT NULL 
            AND 휴대전화번호 NOT LIKE '010-____-____'
        ) / COUNT(*) * 100, 
        2
    ) AS 오류율
FROM TB_CUST_고객
WHERE 휴대전화번호 IS NOT NULL;

-- 이메일 형식 검증
SELECT COUNT(*)
FROM TB_CUST_고객
WHERE 이메일주소 IS NOT NULL
  AND 이메일주소 NOT LIKE '%@%.%';
```

**5) 최신성 (Timeliness)**

| 측정 항목 | 측정 SQL | 목표 |
|----------|---------|------|
| 데이터 갱신 지연 | 최종변경일시 기준 | 1시간 이내 |
| 배치 처리 지연 | 예정 시각 대비 | 30분 이내 |

```sql
-- 데이터 갱신 지연 검증
SELECT 
    COUNT(*) AS 지연건수
FROM TB_PROD_재고
WHERE 최종변경일시 < SYSDATE - INTERVAL '1' HOUR;
```

#### 8.2.2 품질 점수 산출

**품질 점수 계산식**
```
품질 점수 = (정합성 * 0.25) + (완전성 * 0.25) + 
            (유일성 * 0.20) + (정확성 * 0.20) + 
            (최신성 * 0.10)

각 차원 점수 = (1 - 오류율) * 100
```

**품질 등급**
| 점수 | 등급 | 평가 | 조치 |
|------|------|------|------|
| 95~100 | A | 우수 | 현상 유지 |
| 85~94 | B | 양호 | 개선 권고 |
| 70~84 | C | 보통 | 개선 필요 |
| 70 미만 | D | 미흡 | 즉시 개선 |

### 8.3 데이터 품질 개선

#### 8.3.1 품질 개선 계획

**Phase 1: 긴급 개선 (1개월)**
- 참조 무결성 위반 수정
- 필수 항목 NULL 처리
- PK 중복 제거

**Phase 2: 중기 개선 (3개월)**
- 형식 오류 수정
- 중복 데이터 통합
- 표준화 적용

**Phase 3: 장기 개선 (6개월)**
- 자동 품질 검증 체계 구축
- 품질 모니터링 대시보드
- 예방적 품질 관리

#### 8.3.2 품질 검증 자동화

**품질 검증 프로시저**
```sql
CREATE OR REPLACE PROCEDURE SP_품질검증_일일()
AS
    v_정합성_점수 NUMBER;
    v_완전성_점수 NUMBER;
    v_유일성_점수 NUMBER;
    v_정확성_점수 NUMBER;
    v_최신성_점수 NUMBER;
    v_종합_점수 NUMBER;
BEGIN
    -- 1. 정합성 검증
    SELECT (1 - COUNT(*) / (SELECT COUNT(*) FROM TB_ORD_주문)) * 100
    INTO v_정합성_점수
    FROM TB_ORD_주문 O
    WHERE NOT EXISTS (
        SELECT 1 FROM TB_CUST_고객 C WHERE C.고객ID = O.고객ID
    );
    
    -- 2. 완전성 검증
    SELECT (1 - COUNT(*) FILTER (WHERE 고객명 IS NULL) / COUNT(*)) * 100
    INTO v_완전성_점수
    FROM TB_CUST_고객;
    
    -- 3. 종합 점수 계산
    v_종합_점수 := v_정합성_점수 * 0.25 + v_완전성_점수 * 0.25;
    
    -- 4. 결과 저장
    INSERT INTO TB_품질점검결과 (
        점검일자,
        정합성점수,
        완전성점수,
        종합점수,
        등록일시
    ) VALUES (
        TRUNC(SYSDATE),
        v_정합성_점수,
        v_완전성_점수,
        v_종합_점수,
        SYSDATE
    );
    
    -- 5. 알림 (점수가 85점 미만인 경우)
    IF v_종합_점수 < 85 THEN
        -- 알림 로직
        NULL;
    END IF;
    
    COMMIT;
END;
/
```

---

## 9. 메타데이터 관리

### 9.1 메타데이터 저장소 설계

#### 9.1.1 메타데이터 테이블

**표준 단어 메타데이터**
```sql
CREATE TABLE META_표준단어 (
    단어ID VARCHAR2(20) PRIMARY KEY,
    단어명 VARCHAR2(100) NOT NULL,
    영문명 VARCHAR2(100),
    영문약어 VARCHAR2(50),
    단어정의 VARCHAR2(1000),
    단어구분코드 VARCHAR2(2), -- 01:엔티티, 02:속성, 03:수식
    사용여부 CHAR(1) DEFAULT 'Y',
    등록일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    등록자ID VARCHAR2(20),
    변경일시 TIMESTAMP,
    변경자ID VARCHAR2(20)
);
```

**표준 도메인 메타데이터**
```sql
CREATE TABLE META_표준도메인 (
    도메인ID VARCHAR2(20) PRIMARY KEY,
    도메인명 VARCHAR2(100) NOT NULL,
    데이터타입 VARCHAR2(50),
    데이터길이 NUMBER(10),
    소수점자리 NUMBER(2),
    NULL허용여부 CHAR(1) DEFAULT 'Y',
    기본값 VARCHAR2(100),
    도메인설명 VARCHAR2(1000),
    사용여부 CHAR(1) DEFAULT 'Y',
    등록일시 TIMESTAMP DEFAULT SYSTIMESTAMP
);
```

**표준 용어 메타데이터**
```sql
CREATE TABLE META_표준용어 (
    용어ID VARCHAR2(20) PRIMARY KEY,
    용어명 VARCHAR2(200) NOT NULL,
    영문명 VARCHAR2(200),
    영문약어 VARCHAR2(100),
    도메인ID VARCHAR2(20),
    용어정의 VARCHAR2(1000),
    업무영역코드 VARCHAR2(10),
    사용여부 CHAR(1) DEFAULT 'Y',
    등록일시 TIMESTAMP DEFAULT SYSTIMESTAMP,
    FOREIGN KEY (도메인ID) REFERENCES META_표준도메인(도메인ID)
);
```

**테이블 메타데이터**
```sql
CREATE TABLE META_테이블 (
    테이블ID VARCHAR2(50) PRIMARY KEY,
    스키마명 VARCHAR2(50),
    테이블물리명 VARCHAR2(100),
    테이블논리명 VARCHAR2(200),
    테이블유형코드 VARCHAR2(2), -- 01:마스터, 02:트랜잭션, 03:이력
    업무영역코드 VARCHAR2(10),
    테이블설명 VARCHAR2(2000),
    데이터소유자 VARCHAR2(50),
    레코드건수 NUMBER(15),
    데이터용량MB NUMBER(10,2),
    등록일시 TIMESTAMP DEFAULT SYSTIMESTAMP
);
```

**컬럼 메타데이터**
```sql
CREATE TABLE META_컬럼 (
    컬럼ID VARCHAR2(50) PRIMARY KEY,
    테이블ID VARCHAR2(50),
    컬럼물리명 VARCHAR2(100),
    컬럼논리명 VARCHAR2(200),
    용어ID VARCHAR2(20),
    도메인ID VARCHAR2(20),
    데이터타입 VARCHAR2(50),
    데이터길이 NUMBER(10),
    소수점자리 NUMBER(2),
    NULL허용여부 CHAR(1),
    기본값 VARCHAR2(100),
    PK여부 CHAR(1),
    FK여부 CHAR(1),
    컬럼순서 NUMBER(4),
    컬럼설명 VARCHAR2(2000),
    FOREIGN KEY (테이블ID) REFERENCES META_테이블(테이블ID),
    FOREIGN KEY (용어ID) REFERENCES META_표준용어(용어ID),
    FOREIGN KEY (도메인ID) REFERENCES META_표준도메인(도메인ID)
);
```

### 9.2 메타데이터 수집

#### 9.2.1 자동 수집 프로시저

```sql
CREATE OR REPLACE PROCEDURE SP_메타데이터수집()
AS
BEGIN
    -- 1. 테이블 메타데이터 수집
    MERGE INTO META_테이블 T
    USING (
        SELECT 
            OWNER || '.' || TABLE_NAME AS 테이블ID,
            OWNER AS 스키마명,
            TABLE_NAME AS 테이블물리명,
            NUM_ROWS AS 레코드건수
        FROM ALL_TABLES
        WHERE OWNER = 'SHOP_DB'
    ) S
    ON (T.테이블ID = S.테이블ID)
    WHEN MATCHED THEN
        UPDATE SET T.레코드건수 = S.레코드건수
    WHEN NOT MATCHED THEN
        INSERT (테이블ID, 스키마명, 테이블물리명, 레코드건수)
        VALUES (S.테이블ID, S.스키마명, S.테이블물리명, S.레코드건수);
    
    -- 2. 컬럼 메타데이터 수집
    MERGE INTO META_컬럼 C
    USING (
        SELECT 
            OWNER || '.' || TABLE_NAME || '.' || COLUMN_NAME AS 컬럼ID,
            OWNER || '.' || TABLE_NAME AS 테이블ID,
            COLUMN_NAME AS 컬럼물리명,
            DATA_TYPE AS 데이터타입,
            DATA_LENGTH AS 데이터길이,
            NULLABLE AS NULL허용여부,
            COLUMN_ID AS 컬럼순서
        FROM ALL_TAB_COLUMNS
        WHERE OWNER = 'SHOP_DB'
    ) S
    ON (C.컬럼ID = S.컬럼ID)
    WHEN MATCHED THEN
        UPDATE SET 
            C.데이터타입 = S.데이터타입,
            C.데이터길이 = S.데이터길이
    WHEN NOT MATCHED THEN
        INSERT (컬럼ID, 테이블ID, 컬럼물리명, 데이터타입, 데이터길이)
        VALUES (S.컬럼ID, S.테이블ID, S.컬럼물리명, S.데이터타입, S.데이터길이);
    
    COMMIT;
END;
/
```

### 9.3 메타데이터 활용

#### 9.3.1 데이터 사전 조회

```sql
-- 테이블별 컬럼 정보 조회
SELECT 
    T.테이블논리명,
    C.컬럼논리명,
    C.컬럼물리명,
    D.도메인명,
    C.데이터타입,
    C.데이터길이,
    C.NULL허용여부,
    C.컬럼설명
FROM META_테이블 T
JOIN META_컬럼 C ON T.테이블ID = C.테이블ID
LEFT JOIN META_표준도메인 D ON C.도메인ID = D.도메인ID
WHERE T.테이블물리명 = 'TB_CUST_고객'
ORDER BY C.컬럼순서;
```

#### 9.3.2 영향도 분석

```sql
-- 특정 컬럼 변경 시 영향도 분석
SELECT 
    T.테이블논리명,
    T.테이블물리명,
    C.컬럼논리명,
    C.컬럼물리명
FROM META_컬럼 C
JOIN META_테이블 T ON C.테이블ID = T.테이블ID
WHERE C.용어ID = (
    SELECT 용어ID 
    FROM META_표준용어 
    WHERE 용어명 = '고객ID'
);
```

#### 9.3.3 데이터 리니지 (Data Lineage)

```sql
-- 데이터 계보 추적
CREATE TABLE META_데이터리니지 (
    리니지ID VARCHAR2(20) PRIMARY KEY,
    소스테이블ID VARCHAR2(50),
    소스컬럼ID VARCHAR2(50),
    타겟테이블ID VARCHAR2(50),
    타겟컬럼ID VARCHAR2(50),
    변환규칙 VARCHAR2(2000),
    프로세스명 VARCHAR2(200),
    등록일시 TIMESTAMP DEFAULT SYSTIMESTAMP
);

-- 리니지 조회 (고객ID 추적)
WITH RECURSIVE lineage AS (
    -- 시작점: 원천 데이터
    SELECT 
        소스테이블ID,
        소스컬럼ID,
        타겟테이블ID,
        타겟컬럼ID,
        1 AS 레벨
    FROM META_데이터리니지
    WHERE 소스테이블ID = 'CRM.TB_CUST_고객'
      AND 소스컬럼ID LIKE '%고객ID%'
    
    UNION ALL
    
    -- 재귀: 다음 단계 추적
    SELECT 
        L.타겟테이블ID,
        L.타겟컬럼ID,
        M.타겟테이블ID,
        M.타겟컬럼ID,
        lineage.레벨 + 1
    FROM META_데이터리니지 M
    JOIN lineage ON M.소스테이블ID = lineage.타겟테이블ID
    WHERE lineage.레벨 < 5
)
SELECT * FROM lineage;
```

---

## 10. 이행 계획

### 10.1 이행 전략

#### 10.1.1 이행 방식

**빅뱅 vs 단계적 이행**

| 항목 | 빅뱅 방식 | 단계적 이행 |
|------|----------|-----------|
| **장점** | 빠른 완료, 단순한 관리 | 리스크 분산, 안정적 전환 |
| **단점** | 높은 리스크, 롤백 어려움 | 긴 기간, 복잡한 관리 |
| **선택** | ❌ | ✅ 채택 |

**단계적 이행 계획**
```
Phase 1: 마스터 데이터 (1개월)
└─ 고객, 상품, 코드 마스터 통합

Phase 2: 주문 데이터 (2개월)
└─ 주문, 주문상세 통합

Phase 3: 연관 데이터 (2개월)
└─ 배송, 결제, 정산 통합

Phase 4: 분석 데이터 (1개월)
└─ DW 구축 및 이력 데이터 마이그레이션
```

#### 10.1.2 이행 원칙

1. **무중단 이행**: 서비스 중단 없이 단계적 전환
2. **데이터 검증**: 각 단계마다 철저한 검증
3. **롤백 계획**: 문제 발생 시 즉시 복구 가능
4. **병렬 운영**: 일정 기간 신/구 시스템 병행
5. **점진적 전환**: 트래픽을 점진적으로 이동

### 10.2 데이터 마이그레이션

#### 10.2.1 마이그레이션 절차

```
1. 데이터 추출 (Extract)
   ├─ 원천 시스템에서 데이터 추출
   ├─ 증분 추출 전략 수립
   └─ 추출 검증
        ↓
2. 데이터 변환 (Transform)
   ├─ 표준 적용
   ├─ 데이터 정제
   ├─ 형식 변환
   └─ 유효성 검증
        ↓
3. 데이터 적재 (Load)
   ├─ 목표 시스템 적재
   ├─ 제약조건 검증
   └─ 적재 검증
        ↓
4. 데이터 검증 (Validation)
   ├─ 건수 검증
   ├─ 금액 검증
   ├─ 샘플 검증
   └─ 사용자 검증
```

#### 10.2.2 고객 마스터 마이그레이션 스크립트

```sql
-- 1. 임시 테이블 생성
CREATE TABLE TT_고객_마이그레이션 AS
SELECT * FROM TB_CUST_고객 WHERE 1=2;

-- 2. 데이터 추출 및 변환
INSERT INTO TT_고객_마이그레이션 (
    고객ID,
    고객명,
    고객등급코드,
    생년월일,
    성별코드,
    휴대전화번호,
    이메일주소,
    가입일자,
    최초등록일시
)
SELECT 
    'CUST' || LPAD(mem_no, 10, '0') AS 고객ID,  -- ID 표준화
    TRIM(mem_name) AS 고객명,                   -- 공백 제거
    CASE 
        WHEN grade = 'VIP' THEN '03'
        WHEN grade = 'GOLD' THEN '02'
        ELSE '01'
    END AS 고객등급코드,                        -- 코드 표준화
    TO_DATE(birth_date, 'YYYYMMDD') AS 생년월일,
    CASE 
        WHEN SUBSTR(ssn, 7, 1) IN ('1', '3') THEN 'M'
        WHEN SUBSTR(ssn, 7, 1) IN ('2', '4') THEN 'F'
    END AS 성별코드,
    REGEXP_REPLACE(phone, '[^0-9]', '') AS 휴대전화번호,  -- 숫자만 추출
    LOWER(email) AS 이메일주소,                 -- 소문자 변환
    TO_DATE(reg_date, 'YYYYMMDD') AS 가입일자,
    TO_TIMESTAMP(reg_date || ' ' || reg_time, 'YYYYMMDD HH24MISS') AS 최초등록일시
FROM CRM.MEMBER_INFO
WHERE active_yn = 'Y';

-- 3. 데이터 검증
SELECT 
    '총건수' AS 구분,
    (SELECT COUNT(*) FROM CRM.MEMBER_INFO WHERE active_yn = 'Y') AS 원천,
    (SELECT COUNT(*) FROM TT_고객_마이그레이션) AS 대상,
    CASE 
        WHEN (SELECT COUNT(*) FROM CRM.MEMBER_INFO WHERE active_yn = 'Y') = 
             (SELECT COUNT(*) FROM TT_고객_마이그레이션) 
        THEN 'OK' 
        ELSE 'FAIL' 
    END AS 검증결과
FROM DUAL;

-- 4. 실제 테이블 적재
INSERT INTO TB_CUST_고객
SELECT * FROM TT_고객_마이그레이션;

COMMIT;

-- 5. 적재 후 검증
SELECT 
    '건수검증' AS 검증항목,
    COUNT(*) AS 건수,
    CASE WHEN COUNT(*) > 0 THEN 'OK' ELSE 'FAIL' END AS 결과
FROM TB_CUST_고객
UNION ALL
SELECT 
    '필수항목NULL',
    COUNT(*),
    CASE WHEN COUNT(*) = 0 THEN 'OK' ELSE 'FAIL' END
FROM TB_CUST_고객
WHERE 고객명 IS NULL OR 고객ID IS NULL;
```

#### 10.2.3 마이그레이션 검증

**검증 체크리스트**

| 검증 항목 | 검증 방법 | 기준 |
|----------|----------|------|
| **건수 검증** | 원천 vs 대상 건수 비교 | 100% 일치 |
| **금액 검증** | 총 금액 합계 비교 | 오차 0.01% 이내 |
| **필수 항목** | NULL 값 존재 여부 | 0건 |
| **참조 무결성** | FK 제약조건 위반 | 0건 |
| **중복 검증** | PK 중복 여부 | 0건 |
| **샘플 검증** | 무작위 100건 육안 확인 | 100% 일치 |

**검증 SQL**
```sql
-- 마이그레이션 검증 리포트
SELECT 
    '건수 검증' AS 검증항목,
    원천.건수 AS 원천건수,
    대상.건수 AS 대상건수,
    원천.건수 - 대상.건수 AS 차이,
    CASE 
        WHEN 원천.건수 = 대상.건수 THEN 'PASS'
        ELSE 'FAIL'
    END AS 결과
FROM 
    (SELECT COUNT(*) AS 건수 FROM CRM.MEMBER_INFO WHERE active_yn = 'Y') 원천,
    (SELECT COUNT(*) AS 건수 FROM TB_CUST_고객) 대상

UNION ALL

SELECT 
    '금액 검증',
    원천.총금액,
    대상.총금액,
    원천.총금액 - 대상.총금액,
    CASE 
        WHEN ABS(원천.총금액 - 대상.총금액) < 1 THEN 'PASS'
        ELSE 'FAIL'
    END
FROM 
    (SELECT SUM(total_amt) AS 총금액 FROM OMS.ORDERS) 원천,
    (SELECT SUM(최종결제금액) AS 총금액 FROM TB_ORD_주문) 대상;
```

### 10.3 일정 계획

#### 10.3.1 전체 일정 (간트 차트)

| 단계 | 활동 | 3월 | 4월 | 5월 | 6월 | 7월 | 8월 |
|------|------|-----|-----|-----|-----|-----|-----|
| **준비** | 현황 분석 | ██ | | | | | |
| | 표준 정의 | ██ | | | | | |
| | 모델 설계 | ██ | | | | | |
| **구축** | 마스터 데이터 | | ██ | | | | |
| | 주문 데이터 | | | ██ | ██ | | |
| | 연관 데이터 | | | | ██ | ██ | |
| | DW 구축 | | | | | ██ | |
| **검증** | 통합 테스트 | | | | | | ██ |
| | 사용자 검증 | | | | | | ██ |
| **이행** | 단계적 오픈 | | | | | | ██ |

#### 10.3.2 단계별 상세 일정

**Phase 1: 마스터 데이터 (4월)**
- Week 1: 고객 마스터 마이그레이션
- Week 2: 상품 마스터 마이그레이션
- Week 3: 코드 마스터 통합
- Week 4: 검증 및 안정화

**Phase 2: 주문 데이터 (5~6월)**
- Week 1-2: 주문 데이터 마이그레이션
- Week 3-4: 주문상세 데이터 마이그레이션
- Week 5-6: 실시간 연계 구축
- Week 7-8: 검증 및 안정화

**Phase 3: 연관 데이터 (7월)**
- Week 1-2: 배송 데이터 통합
- Week 2-3: 결제 데이터 통합
- Week 3-4: 정산 데이터 통합

**Phase 4: DW 구축 (7~8월)**
- Week 1-2: ETL 개발
- Week 3: 이력 데이터 적재
- Week 4: 검증 및 오픈

### 10.4 리스크 관리

#### 10.4.1 리스크 식별

| ID | 리스크 | 발생 가능성 | 영향도 | 대응 방안 |
|----|--------|-----------|--------|----------|
| R-01 | 마이그레이션 중 데이터 손실 | 중 | 상 | 백업 및 롤백 계획 수립 |
| R-02 | 성능 저하 | 중 | 상 | 사전 성능 테스트 |
| R-03 | 일정 지연 | 상 | 중 | 버퍼 기간 확보 |
| R-04 | 시스템 장애 | 하 | 상 | 장애 대응 매뉴얼 |
| R-05 | 데이터 품질 이슈 | 중 | 중 | 사전 품질 검증 |

#### 10.4.2 롤백 계획

**롤백 트리거**
- 마이그레이션 오류율 5% 초과
- 시스템 응답시간 2배 이상 증가
- 치명적 데이터 손실 발생

**롤백 절차**
```
1. 서비스 중지 알림
   ↓
2. 신규 시스템 중지
   ↓
3. 기존 시스템 재가동
   ↓
4. 데이터 백업본 복구
   ↓
5. 서비스 재개
   ↓
6. 원인 분석 및 재시도 계획
```

---

## 11. 산출물 목록

### 11.1 문서 산출물

#### 11.1.1 계획 단계

| 번호 | 산출물명 | 페이지 | 작성자 | 승인자 |
|------|---------|--------|--------|--------|
| 1.1 | 프로젝트 계획서 | 30 | PM | 경영진 |
| 1.2 | 현황 분석 보고서 | 50 | DA | PM |
| 1.3 | 요구사항 정의서 | 40 | DA | 업무담당자 |

#### 11.1.2 설계 단계

| 번호 | 산출물명 | 페이지 | 작성자 | 승인자 |
|------|---------|--------|--------|--------|
| 2.1 | 데이터 표준 정의서 | 100 | DA | 표준위원회 |
| 2.2 | 개념 데이터 모델 | 20 | 모델러 | DA |
| 2.3 | 논리 데이터 모델 | 80 | 모델러 | DA |
| 2.4 | 물리 데이터 모델 | 100 | 모델러 | DBA |
| 2.5 | 인터페이스 설계서 | 60 | DA | PM |
| 2.6 | 데이터 품질 계획서 | 40 | 품질관리자 | DA |

#### 11.1.3 구현 단계

| 번호 | 산출물명 | 페이지 | 작성자 | 승인자 |
|------|---------|--------|--------|--------|
| 3.1 | DDL 스크립트 | - | DBA | DA |
| 3.2 | 마이그레이션 스크립트 | - | 데이터엔지니어 | DBA |
| 3.3 | ETL 프로세스 문서 | 50 | 데이터엔지니어 | DA |

#### 11.1.4 검증 단계

| 번호 | 산출물명 | 페이지 | 작성자 | 승인자 |
|------|---------|--------|--------|--------|
| 4.1 | 테스트 계획서 | 30 | QA | PM |
| 4.2 | 테스트 케이스 | 100 | QA | PM |
| 4.3 | 테스트 결과서 | 50 | QA | PM |
| 4.4 | 품질 검증 보고서 | 40 | 품질관리자 | DA |

#### 11.1.5 이행 단계

| 번호 | 산출물명 | 페이지 | 작성자 | 승인자 |
|------|---------|--------|--------|--------|
| 5.1 | 이행 계획서 | 40 | PM | 경영진 |
| 5.2 | 이행 절차서 | 30 | PM | DA |
| 5.3 | 롤백 계획서 | 20 | PM | DA |
| 5.4 | 이행 완료 보고서 | 30 | PM | 경영진 |

### 11.2 데이터 산출물

| 산출물 유형 | 내용 | 형식 |
|-----------|------|------|
| **표준 단어 사전** | 200개 표준 단어 | Excel |
| **표준 도메인** | 50개 표준 도메인 | Excel |
| **표준 용어 사전** | 500개 표준 용어 | Excel |
| **표준 코드** | 100개 코드 그룹 | Excel |
| **ERD** | 전체 데이터 모델 | ERwin |
| **DDL** | 테이블 생성 스크립트 | SQL |
| **메타데이터** | 메타데이터 저장소 | Database |

### 11.3 시스템 산출물

| 산출물 | 내용 | 도구 |
|--------|------|------|
| **데이터베이스** | 통합 데이터베이스 | Oracle, MySQL, PostgreSQL |
| **ETL 프로세스** | 데이터 통합 파이프라인 | Apache Airflow |
| **API** | 데이터 API 서비스 | Spring Boot |
| **모니터링** | 데이터 품질 대시보드 | Grafana |

---

## 12. 프로젝트 성과

### 12.1 정량적 성과

| KPI | 목표 | 실제 | 달성률 |
|-----|------|------|--------|
| 데이터 정합성 | 95% | 96.5% | 101.6% |
| 데이터 중복률 | 5% | 3.2% | 136.0% |
| 분석 리드타임 | 1시간 | 45분 | 125.0% |
| 시스템 통합도 | 90% | 92% | 102.2% |
| 표준 준수율 | 95% | 97% | 102.1% |

### 12.2 정성적 성과

1. **데이터 거버넌스 체계 확립**
   - 전사 데이터 표준 수립
   - 데이터 품질 관리 프로세스 정착

2. **업무 효율성 향상**
   - 360도 고객뷰 구현으로 고객 응대 시간 40% 단축
   - 실시간 재고 조회로 재고 오류 80% 감소

3. **의사결정 지원 강화**
   - 실시간 매출 분석 가능
   - 데이터 기반 마케팅 전략 수립

4. **기술 역량 향상**
   - 데이터 아키텍처 설계 역량 확보
   - 최신 데이터 기술 스택 도입

---

## 13. 교훈 및 개선사항

### 13.1 성공 요인

1. **명확한 표준 정의**
   - 프로젝트 초기 표준 수립에 충분한 시간 투자
   - 모든 이해관계자의 합의 도출

2. **단계적 접근**
   - 빅뱅 대신 단계적 이행으로 리스크 최소화
   - 각 단계별 철저한 검증

3. **적극적인 의사소통**
   - 주간 정기 회의로 이슈 신속 해결
   - 이해관계자 간 긴밀한 협업

### 13.2 개선사항

1. **초기 일정 산정**
   - 데이터 정제에 예상보다 많은 시간 소요
   - 향후 프로젝트 시 버퍼 30% 추가 필요

2. **성능 테스트**
   - 대용량 데이터 성능 테스트 사전 실시 필요
   - 파티셔닝, 인덱싱 전략 조기 수립

3. **변화 관리**
   - 사용자 교육 강화 필요
   - 사전 충분한 교육 및 훈련 시간 확보

---

## 부록

### A. 참고 문헌

1. 데이터아키텍처 전문가 가이드 - 한국데이터산업진흥원
2. Data Management Body of Knowledge (DMBOK)
3. TOGAF 9.2 - The Open Group
4. 정보시스템 구축·운영 지침 - 행정안전부

### B. 용어 정의

| 용어 | 정의 |
|------|------|
| **DAP** | Data Architecture Professional - 데이터아키텍처 전문가 |
| **CDM** | Conceptual Data Model - 개념 데이터 모델 |
| **LDM** | Logical Data Model - 논리 데이터 모델 |
| **PDM** | Physical Data Model - 물리 데이터 모델 |
| **CDC** | Change Data Capture - 변경 데이터 캡처 |
| **ETL** | Extract, Transform, Load - 추출, 변환, 적재 |
| **DW** | Data Warehouse - 데이터 웨어하우스 |

### C. 프로젝트 팀 연락처

| 역할 | 이름 | 이메일 | 전화 |
|------|------|--------|------|
| PM | 김철수 | kim@company.com | 010-1234-5678 |
| DA | 이영희 | lee@company.com | 010-2345-6789 |
| 모델러 | 박지훈 | park@company.com | 010-3456-7890 |

---

**문서 종료**

작성일: 2024.08.31  
버전: 1.0  
작성자: 데이터아키텍처팀
