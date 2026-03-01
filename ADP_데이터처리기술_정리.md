# ADP 데이터 처리 기술 완전 정리

> **ADP(Advanced Data Analytics Professional)** - 데이터 처리 기술 영역 심화 학습 자료  
> 한국데이터산업진흥원(K-DATA) 공식 출제 범위 기준

---

## 목차

1. [데이터 수집 기술](#1-데이터-수집-기술)
2. [데이터 저장 기술](#2-데이터-저장-기술)
3. [데이터 처리 및 변환 기술](#3-데이터-처리-및-변환-기술)
4. [데이터 통합 기술 (EAI/ETL/ELT)](#4-데이터-통합-기술-eaietlelt)
5. [빅데이터 처리 프레임워크](#5-빅데이터-처리-프레임워크)
6. [스트리밍 데이터 처리](#6-스트리밍-데이터-처리)
7. [데이터 품질 관리](#7-데이터-품질-관리)
8. [데이터 보안 및 거버넌스](#8-데이터-보안-및-거버넌스)
9. [요약 테이블](#9-요약-테이블)

---

## 1. 데이터 수집 기술

### 1.1 크롤링 (Web Crawling)

웹 크롤링은 자동화된 프로그램(봇)이 웹 페이지를 순회하며 데이터를 수집하는 기술이다.

- **정적 크롤링**: HTML 파싱 방식. `BeautifulSoup`, `lxml` 사용
- **동적 크롤링**: JavaScript 렌더링 필요 시 `Selenium`, `Playwright` 사용
- **API 기반 수집**: REST API, GraphQL을 통한 구조화된 데이터 수집
- **RSS/Atom 피드**: 뉴스, 블로그 등 구독형 데이터 수집

```python
# 정적 크롤링 예시
import requests
from bs4 import BeautifulSoup

response = requests.get("https://example.com")
soup = BeautifulSoup(response.text, 'html.parser')
data = soup.find_all('div', class_='content')
```

### 1.2 로그 수집

- **Fluentd / Fluentbit**: 다양한 소스의 로그를 수집·전송하는 오픈소스 데이터 수집기
- **Logstash**: ELK Stack의 구성 요소, 로그 수집 및 필터링
- **Syslog**: Unix/Linux 시스템 표준 로그 프로토콜
- **SNMP**: 네트워크 장비 모니터링 및 데이터 수집

### 1.3 CDC (Change Data Capture)

데이터베이스의 변경사항(INSERT, UPDATE, DELETE)을 실시간으로 감지하여 캡처하는 기술이다.

| CDC 방식 | 설명 | 도구 |
|----------|------|------|
| 로그 기반 | DB 트랜잭션 로그를 파싱 | Debezium, Maxwell |
| 트리거 기반 | DB 트리거로 변경 감지 | 직접 구현 |
| 타임스탬프 기반 | 변경 시간 컬럼 조회 | 직접 쿼리 |
| 스냅샷 비교 | 이전/이후 스냅샷 비교 | 배치 처리 |

```
CDC 흐름: DB 변경 → Debezium → Kafka Topic → Consumer → Target DB
```

### 1.4 IoT 데이터 수집

- **MQTT**: 경량 메시지 프로토콜, IoT 기기 간 통신 표준
- **AMQP**: 고급 메시지 큐 프로토콜 (RabbitMQ)
- **CoAP**: 제약된 환경(저전력 기기)을 위한 REST 프로토콜

---

## 2. 데이터 저장 기술

### 2.1 관계형 데이터베이스 (RDBMS)

ACID(원자성, 일관성, 고립성, 내구성) 특성을 보장하는 전통적 데이터 저장소이다.

| 구성 요소 | 설명 |
|-----------|------|
| 정규화 | 데이터 중복 최소화 (1NF~BCNF) |
| 인덱스 | B-Tree, Hash, Bitmap, Full-Text |
| 파티셔닝 | Range, List, Hash, Composite |
| 샤딩 | 수평 분할을 통한 분산 저장 |

**주요 RDBMS**: MySQL, PostgreSQL, Oracle, MS SQL Server

### 2.2 NoSQL 데이터베이스

비정형/반정형 데이터 처리에 최적화된 데이터베이스다. BASE 특성(Basically Available, Soft state, Eventually consistent)을 따른다.

| 유형 | 특징 | 대표 제품 | 사용 사례 |
|------|------|-----------|-----------|
| Key-Value | 단순 키-값 쌍 저장 | Redis, DynamoDB | 세션, 캐시 |
| Document | JSON/BSON 형태 저장 | MongoDB, CouchDB | 사용자 프로필, 카탈로그 |
| Column-Family | 컬럼 기반 분산 저장 | HBase, Cassandra | 시계열, 이벤트 로그 |
| Graph | 노드-엣지 관계 저장 | Neo4j, Amazon Neptune | SNS, 추천 시스템 |

**CAP 정리**: 분산 시스템에서 일관성(Consistency), 가용성(Availability), 분할 내성(Partition Tolerance) 중 두 가지만 동시에 보장 가능하다.

### 2.3 데이터 웨어하우스 (DW)

의사결정 지원을 위해 통합·주제 중심으로 구성된 데이터 저장소이다.

**특징**: 주제 지향적, 통합적, 시간 가변적, 비휘발적

**스키마 유형**:
- **스타 스키마(Star Schema)**: 팩트 테이블을 중심으로 차원 테이블이 연결된 단순 구조
- **스노우플레이크 스키마(Snowflake Schema)**: 차원 테이블이 정규화된 복잡 구조
- **갤럭시 스키마(Galaxy Schema)**: 다수의 팩트 테이블을 공유 차원으로 연결

```
[스타 스키마 구조]
         [날짜 차원]
              |
[고객 차원] - [매출 팩트] - [제품 차원]
              |
         [지역 차원]
```

**OLAP 연산**:
- **Roll-up**: 상세 데이터를 집계 (시→월→연)
- **Drill-down**: 집계 데이터를 상세화 (연→월→시)
- **Slice**: 특정 차원으로 단면 추출
- **Dice**: 복수 차원으로 큐브 추출
- **Pivot**: 차원 축을 회전

### 2.4 데이터 레이크 (Data Lake)

원시 데이터를 스키마 정의 없이 대규모로 저장하는 저장소이다.

| 구분 | 데이터 웨어하우스 | 데이터 레이크 |
|------|-----------------|--------------|
| 데이터 형태 | 정형 | 정형·반정형·비정형 |
| 스키마 | Schema-on-Write | Schema-on-Read |
| 사용 목적 | BI·리포팅 | 탐색적 분석·ML |
| 저장 비용 | 높음 | 낮음 |
| 처리 속도 | 빠름 | 상대적으로 느림 |

---

## 3. 데이터 처리 및 변환 기술

### 3.1 데이터 전처리 기법

#### 결측값 처리

| 방법 | 설명 | 적합 상황 |
|------|------|-----------|
| 삭제 | 결측 행/열 제거 | 결측 비율 < 5% |
| 평균/중앙값 대체 | 수치형 변수 대체 | 분포가 정규에 가까울 때 |
| 최빈값 대체 | 범주형 변수 대체 | 명목형 데이터 |
| 회귀 대체 | 예측 모델로 추정 | 변수 간 관계 존재 시 |
| KNN 대체 | 유사 샘플 기반 추정 | 고차원 데이터 |
| 다중 대체(MI) | 여러 대체값 생성 후 통합 | 정밀 분석 필요 시 |

#### 이상값 처리

- **Z-Score 방법**: |Z| > 3 인 경우 이상값으로 판별
  - Z = (X - μ) / σ
- **IQR 방법**: Q1 - 1.5×IQR 미만 또는 Q3 + 1.5×IQR 초과
- **Isolation Forest**: 트리 기반 비지도 이상값 탐지
- **LOF (Local Outlier Factor)**: 밀도 기반 이상값 탐지

#### 데이터 정규화 / 표준화

| 방법 | 공식 | 특징 |
|------|------|------|
| Min-Max 정규화 | (x - min) / (max - min) | [0, 1] 범위로 변환 |
| Z-Score 표준화 | (x - μ) / σ | 평균 0, 표준편차 1 |
| Robust Scaling | (x - median) / IQR | 이상값에 강건 |
| Log 변환 | log(x + 1) | 왜도 감소, 오른쪽 꼬리 분포 |
| Box-Cox 변환 | (xλ - 1) / λ | 정규성 확보 |

### 3.2 인코딩 기법

| 방법 | 설명 | 주의사항 |
|------|------|---------|
| Label Encoding | 범주형 → 정수 | 순서 의미 부여 위험 |
| One-Hot Encoding | 이진 더미 변수 생성 | 고차원 문제(차원의 저주) |
| Target Encoding | 타깃 평균값으로 대체 | 데이터 누수 주의 |
| Frequency Encoding | 빈도수로 대체 | - |
| Binary Encoding | 이진수로 인코딩 | One-Hot의 차원 절감 |

### 3.3 피처 엔지니어링

- **다항식 특성**: 상호작용 항, 제곱항 생성
- **날짜 분해**: 연, 월, 일, 요일, 시간대 등 파생 변수
- **텍스트 특성**: TF-IDF, Word2Vec, Bag of Words
- **집계 특성**: 그룹별 통계량(mean, std, max, min)
- **차원 축소**: PCA, t-SNE, UMAP

---

## 4. 데이터 통합 기술 (EAI/ETL/ELT)

### 4.1 EAI (Enterprise Application Integration)

기업 내 이기종 애플리케이션 시스템을 통합하는 미들웨어 기술이다.

**EAI 통합 유형**:

| 유형 | 설명 | 장점 | 단점 |
|------|------|------|------|
| Point-to-Point | 시스템 간 직접 연결 | 단순 구현 | n(n-1)/2 연결 문제 |
| Hub-and-Spoke | 중앙 허브를 통한 연결 | 관리 용이 | 허브 단일 장애점 |
| Bus (ESB) | 메시지 버스로 연결 | 확장성, 유연성 | 초기 구축 복잡 |
| Hybrid | 복합 방식 | 유연한 구성 | 설계 복잡 |

**ESB (Enterprise Service Bus)**:  
서비스 지향 아키텍처(SOA)의 핵심 구성 요소로, 메시지 라우팅·변환·프로토콜 중재를 담당한다.

```
[시스템 A] ──┐
[시스템 B] ──┼── [ESB] ──┬── [시스템 D]
[시스템 C] ──┘           └── [시스템 E]
```

### 4.2 ETL (Extract, Transform, Load)

소스 데이터를 추출하여 변환 후 타깃 저장소에 적재하는 전통적 데이터 통합 프로세스이다.

```
[소스 DB] → [Extract] → [Staging Area] → [Transform] → [Load] → [DW/DM]
```

**ETL 구성 요소**:

| 단계 | 주요 작업 |
|------|----------|
| Extract | 소스 시스템에서 데이터 추출 (Full/Incremental) |
| Transform | 데이터 정제, 표준화, 통합, 집계, 파생 변수 생성 |
| Load | 타깃 시스템에 적재 (Full Load / Incremental Load) |

**ETL vs ELT 비교**:

| 구분 | ETL | ELT |
|------|-----|-----|
| 변환 시점 | 적재 전 | 적재 후 |
| 처리 위치 | 별도 ETL 서버 | 타깃 시스템 내 |
| 적합 환경 | 전통적 DW | 클라우드 DW, 데이터 레이크 |
| 대표 도구 | Informatica, SSIS, Talend | dbt, Spark, Snowflake |
| 처리 속도 | 변환에 시간 소요 | 적재 빠름, 변환은 후처리 |

### 4.3 데이터 파이프라인

**배치 처리 vs 스트림 처리**:

| 구분 | 배치 처리 | 스트림 처리 |
|------|----------|------------|
| 처리 방식 | 일정 시간 누적 후 처리 | 데이터 발생 즉시 처리 |
| 지연 시간 | 분~시간 단위 | 밀리초~초 단위 |
| 처리량 | 대용량 처리 가능 | 상대적으로 소량 |
| 대표 도구 | Spark(batch), Hadoop | Kafka Streams, Flink |
| 사용 사례 | 야간 정산, 월간 리포트 | 실시간 알림, 이상 탐지 |

---

## 5. 빅데이터 처리 프레임워크

### 5.1 Hadoop 에코시스템

분산 저장 및 처리를 위한 오픈소스 프레임워크다.

**핵심 구성 요소**:

| 컴포넌트 | 역할 | 주요 특징 |
|----------|------|----------|
| HDFS | 분산 파일 시스템 | 블록 단위(128MB) 저장, 복제 계수 3 |
| YARN | 리소스 관리 | ResourceManager + NodeManager |
| MapReduce | 분산 처리 모델 | Map→Shuffle→Reduce |
| Hive | SQL on Hadoop | HiveQL, 배치 분석 |
| HBase | 분산 NoSQL DB | Column-Family, 실시간 조회 |
| Pig | 스크립트 처리 | Pig Latin 언어 |
| Sqoop | RDBMS 연동 | Hadoop↔RDBMS 데이터 이동 |
| Oozie | 워크플로우 관리 | Job 스케줄링 |
| ZooKeeper | 분산 코디네이션 | 설정 관리, 동기화 |

**HDFS 아키텍처**:
```
Client
  ↓
NameNode (메타데이터 관리)
  ↓
DataNode1  DataNode2  DataNode3
(블록1,2)  (블록2,3)  (블록1,3)  ← 복제 계수 = 3
```

**MapReduce 처리 흐름**:
```
Input → Split → Map → (Combiner) → Shuffle/Sort → Reduce → Output
```

- **Map**: 입력 데이터를 Key-Value 쌍으로 변환
- **Shuffle**: 동일 Key의 데이터를 동일 Reducer로 전송
- **Reduce**: Key별로 집계·처리

### 5.2 Apache Spark

메모리 기반 고속 분산 처리 프레임워크로, Hadoop MapReduce보다 최대 100배 빠르다.

**핵심 개념**:

| 개념 | 설명 |
|------|------|
| RDD (Resilient Distributed Dataset) | 분산 불변 데이터셋, 결함 허용성 |
| DataFrame | 스키마 있는 분산 데이터, SQL 지원 |
| Dataset | 타입 안전 DataFrame (Scala/Java) |
| DAG (Directed Acyclic Graph) | 연산의 논리적 실행 계획 |
| Lazy Evaluation | Action 호출 시에만 실제 계산 수행 |

**Transformation vs Action**:

| 구분 | 연산 종류 | 예시 |
|------|----------|------|
| Transformation (지연) | map, filter, flatMap, groupBy, join | df.filter(col > 0) |
| Action (즉시 실행) | count, collect, show, save, reduce | df.count() |

**Spark 에코시스템**:
- **Spark SQL**: SQL 쿼리 및 DataFrame API
- **Spark Streaming / Structured Streaming**: 스트림 처리
- **MLlib**: 분산 머신러닝 라이브러리
- **GraphX**: 그래프 처리
- **SparkR / PySpark**: R/Python API

```python
# PySpark 예시
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, avg

spark = SparkSession.builder.appName("example").getOrCreate()
df = spark.read.csv("data.csv", header=True, inferSchema=True)

result = df.filter(col("age") > 25) \
           .groupBy("department") \
           .agg(avg("salary").alias("avg_salary")) \
           .orderBy("avg_salary", ascending=False)

result.show()
```

### 5.3 데이터 처리 성능 최적화

- **파티셔닝**: 데이터를 논리적으로 분할하여 병렬 처리 향상
- **버켓팅(Bucketing)**: 특정 컬럼 해시 기반 분산 저장 (조인 최적화)
- **캐싱**: 자주 사용하는 데이터를 메모리에 저장
- **브로드캐스트 조인**: 소형 테이블을 전체 노드에 복제하여 셔플 방지
- **컬럼형 저장(Parquet, ORC)**: 압축률 향상, 컬럼 프루닝

---

## 6. 스트리밍 데이터 처리

### 6.1 Apache Kafka

분산 메시지 스트리밍 플랫폼으로 고성능·고가용성의 실시간 데이터 파이프라인을 구축한다.

**핵심 구성 요소**:

| 컴포넌트 | 역할 |
|----------|------|
| Broker | 메시지를 저장·전달하는 Kafka 서버 |
| Topic | 메시지를 분류하는 논리적 채널 |
| Partition | Topic을 물리적으로 분할한 단위 |
| Producer | 메시지를 생성하여 Topic에 발행 |
| Consumer | Topic에서 메시지를 구독·소비 |
| Consumer Group | 파티션을 분담하는 Consumer 묶음 |
| ZooKeeper / KRaft | 클러스터 메타데이터 관리 |
| Offset | Consumer의 메시지 읽기 위치 |

**Kafka 아키텍처**:
```
Producer → [Topic: Partition 0] → Consumer Group A
         → [Topic: Partition 1] → Consumer Group B
         → [Topic: Partition 2]
```

**Kafka 특징**:
- 메시지를 디스크에 영속 저장 (기본 7일 보존)
- 컨슈머 그룹별 독립적인 오프셋 관리
- 파티션 수 = 최대 병렬 처리 수

### 6.2 스트리밍 처리 시스템 비교

| 구분 | Kafka Streams | Apache Flink | Spark Structured Streaming |
|------|--------------|--------------|--------------------------|
| 처리 모델 | 이벤트 기반 | 이벤트 기반 | 마이크로배치 / 연속 |
| 지연시간 | 밀리초 | 밀리초 | 초 단위 |
| 정확성 보장 | Exactly-Once | Exactly-Once | At-Least-Once / Exactly-Once |
| 상태 관리 | 로컬 상태 저장소 | 분산 상태 관리 | 체크포인트 |
| 배포 복잡도 | 낮음 | 높음 | 중간 |

### 6.3 Lambda 아키텍처 vs Kappa 아키텍처

**Lambda 아키텍처**:
```
             ┌── [배치 레이어] ──────────────────────┐
데이터 입력 →│                                        ├→ 서빙 레이어 → 쿼리
             └── [속도 레이어(실시간)] ───────────────┘
```
- 배치(정확성)와 스트림(속도) 두 레이어를 병렬 운영
- 복잡성이 높고 코드 중복 발생

**Kappa 아키텍처**:
```
데이터 입력 → [스트리밍 레이어(Kafka+Flink)] → 서빙 레이어 → 쿼리
```
- 스트리밍 레이어만으로 모든 처리 수행
- 단순하지만 재처리 비용 발생

---

## 7. 데이터 품질 관리

### 7.1 데이터 품질 6대 요소 (ISO/IEC 25012)

| 품질 요소 | 설명 | 측정 방법 |
|----------|------|----------|
| **완전성** (Completeness) | 필수 데이터가 누락 없이 존재 | 결측값 비율 |
| **정확성** (Accuracy) | 실제 현실을 정확하게 반영 | 오류율, 참조 데이터 비교 |
| **일관성** (Consistency) | 동일 데이터가 여러 저장소에서 동일 | 교차 검증 |
| **적시성** (Timeliness) | 필요 시점에 최신 데이터 제공 | 업데이트 주기 |
| **유일성** (Uniqueness) | 중복 데이터 없음 | 중복률 |
| **유효성** (Validity) | 비즈니스 규칙·형식 준수 | 도메인 규칙 검사 |

### 7.2 데이터 프로파일링

데이터의 구조, 내용, 관계를 자동으로 분석하여 품질을 파악하는 과정이다.

- **컬럼 프로파일링**: NULL 비율, 최솟값/최댓값, 분포, 패턴
- **관계 프로파일링**: 외래키 관계, 종속성 분석
- **데이터 프로파일링 도구**: Apache Griffin, Great Expectations, Talend DQ

### 7.3 마스터 데이터 관리 (MDM)

핵심 비즈니스 데이터(고객, 제품, 직원 등)의 단일 참조 원천(Single Source of Truth)을 구축·유지하는 관리 체계이다.

**MDM 스타일**:
- **Registry 스타일**: 마스터 레코드는 참조만, 실 데이터는 소스 시스템 유지
- **Consolidation 스타일**: 소스에서 통합 복사본 생성 (읽기 전용)
- **Coexistence 스타일**: 마스터와 소스가 동기화
- **Centralized 스타일**: 하나의 마스터 시스템이 모든 소스를 제어

---

## 8. 데이터 보안 및 거버넌스

### 8.1 데이터 암호화

| 구분 | 알고리즘 | 키 길이 | 용도 |
|------|---------|--------|------|
| 대칭키 | AES | 128/256 bit | 대용량 데이터 암호화 |
| 대칭키 | DES, 3DES | 56/168 bit | 레거시 (비권장) |
| 비대칭키 | RSA | 2048+ bit | 키 교환, 인증서 |
| 비대칭키 | ECC | 256 bit | 모바일, IoT |
| 해시 | SHA-256, SHA-3 | - | 무결성 검증 |

### 8.2 데이터 접근 제어

- **DAC (Discretionary Access Control)**: 소유자가 접근 권한 설정 (파일 시스템)
- **MAC (Mandatory Access Control)**: 시스템이 보안 레벨 기반으로 접근 통제
- **RBAC (Role-Based Access Control)**: 역할 기반 권한 부여 (대부분의 기업 시스템)
- **ABAC (Attribute-Based Access Control)**: 속성(사용자·환경·데이터) 기반 동적 제어

### 8.3 개인정보 보호 기술

| 기술 | 설명 |
|------|------|
| 가명처리 (Pseudonymization) | 식별자를 대체 값으로 변환, 재식별 가능 |
| 익명처리 (Anonymization) | 재식별 불가능하도록 완전 제거 |
| 데이터 마스킹 | 민감 데이터를 *로 대체 |
| k-익명성 | k개 이상의 동일 레코드가 존재하도록 보장 |
| l-다양성 | 민감 속성에 l개 이상의 다양한 값 보장 |
| 차분 프라이버시 | 통계적 노이즈 추가로 개인 정보 보호 |

### 8.4 데이터 거버넌스

데이터의 전체 생명주기에 걸친 관리 정책·프로세스·책임 체계이다.

**주요 구성 요소**:
- **데이터 스튜어드십**: 데이터 품질·정의에 대한 책임 역할
- **데이터 카탈로그**: 데이터 자산 목록, 메타데이터 관리 (Apache Atlas, Alation)
- **데이터 리니지(Lineage)**: 데이터 발생부터 현재까지 흐름 추적
- **메타데이터 관리**: 기술적·비즈니스·운영 메타데이터 통합 관리

---

## 9. 요약 테이블

### 9.1 데이터 처리 기술 전체 요약

| 영역 | 기술/개념 | 핵심 키워드 | 대표 도구 |
|------|----------|------------|----------|
| **데이터 수집** | 크롤링 | 정적/동적, BeautifulSoup, Selenium | BeautifulSoup, Scrapy, Selenium |
| **데이터 수집** | CDC | 로그 기반, 실시간 변경 감지 | Debezium, Maxwell |
| **데이터 수집** | 로그 수집 | Fluentd, Logstash | ELK Stack, Fluentd |
| **데이터 저장** | RDBMS | ACID, 정규화, 인덱스, 파티셔닝 | MySQL, PostgreSQL, Oracle |
| **데이터 저장** | NoSQL | BASE, CAP 정리, 4가지 유형 | MongoDB, Redis, HBase |
| **데이터 저장** | DW | 스타/스노우플레이크 스키마, OLAP | Snowflake, Redshift |
| **데이터 저장** | Data Lake | Schema-on-Read, 원시 데이터 | S3, HDFS, Delta Lake |
| **전처리** | 결측값 처리 | 삭제, 대체, KNN, 다중 대체 | pandas, scikit-learn |
| **전처리** | 이상값 처리 | Z-Score, IQR, Isolation Forest | pandas, sklearn |
| **전처리** | 정규화/표준화 | Min-Max, Z-Score, Box-Cox | sklearn.preprocessing |
| **통합** | EAI | ESB, Hub-and-Spoke, Point-to-Point | MuleSoft, IBM MQ |
| **통합** | ETL | Extract, Transform, Load, Staging | Informatica, Talend, SSIS |
| **통합** | ELT | 적재 후 변환, 클라우드 DW | dbt, Spark |
| **빅데이터** | Hadoop | HDFS, MapReduce, YARN | Hadoop, Hive, HBase |
| **빅데이터** | Spark | RDD, DataFrame, Lazy Evaluation | PySpark, Spark SQL |
| **스트리밍** | Kafka | Topic, Partition, Consumer Group | Apache Kafka |
| **스트리밍** | 아키텍처 | Lambda, Kappa | Kafka+Flink, Spark Streaming |
| **품질** | 데이터 품질 | 완전성·정확성·일관성·적시성·유일성·유효성 | Great Expectations |
| **품질** | MDM | Single Source of Truth, 4가지 스타일 | Informatica MDM |
| **보안** | 암호화 | AES, RSA, SHA-256 | OpenSSL, AWS KMS |
| **보안** | 접근 제어 | DAC, MAC, RBAC, ABAC | IAM, LDAP |
| **보안** | 개인정보 보호 | k-익명성, l-다양성, 차분 프라이버시 | ARX, sdcMicro |

### 9.2 배치 vs 스트리밍 vs 마이크로배치 비교

| 구분 | 배치 | 마이크로배치 | 스트리밍 |
|------|------|------------|---------|
| 지연시간 | 분~시간 | 초~분 | 밀리초~초 |
| 처리량 | 최대 | 높음 | 중간 |
| 복잡도 | 낮음 | 중간 | 높음 |
| 정확도 | 높음 | 높음 | 설계에 따라 다름 |
| 대표 도구 | Spark Batch, Hive | Spark Streaming | Flink, Kafka Streams |
| 사용 사례 | 정산, 리포트 | 실시간 대시보드 | 알림, 이상 탐지 |

### 9.3 주요 NoSQL 유형 비교

| 구분 | Key-Value | Document | Column-Family | Graph |
|------|-----------|----------|---------------|-------|
| 데이터 모델 | 키-값 쌍 | JSON/BSON | 행-열 패밀리 | 노드-엣지 |
| 쿼리 | 키 조회만 | 다양한 필드 조회 | 행 + 컬럼 범위 | 관계 탐색 |
| 확장성 | 수평 확장 용이 | 수평 확장 | 선형 확장 | 제한적 |
| 대표 제품 | Redis, DynamoDB | MongoDB | HBase, Cassandra | Neo4j |
| 강점 | 초고속 캐시 | 유연한 구조 | 대용량 쓰기 | 관계 분석 |

### 9.4 Hadoop MapReduce vs Spark 비교

| 구분 | MapReduce | Spark |
|------|-----------|-------|
| 처리 방식 | 디스크 기반 | 메모리 기반 |
| 처리 속도 | 느림 | 최대 100배 빠름 |
| 데이터 공유 | HDFS를 통한 디스크 I/O | 메모리 내 RDD 공유 |
| 언어 지원 | Java (주), 기타 | Scala, Python, R, Java |
| 스트리밍 | 불가 (배치 전용) | Structured Streaming 지원 |
| ML 지원 | Mahout (제한적) | MLlib (풍부) |
| 내결함성 | HDFS 복제 | RDD Lineage |

---

## 참고: ADP 시험 출제 포인트

ADP 시험에서 데이터 처리 기술 관련 주요 출제 포인트는 다음과 같다.

1. **ETL vs ELT 차이점** - 변환 시점, 적합 환경
2. **CAP 정리** - 세 가지 보장 중 두 가지만 선택 가능
3. **MapReduce 처리 흐름** - Map → Shuffle → Reduce 순서
4. **HDFS 블록 크기와 복제** - 기본 128MB, 복제 계수 3
5. **CDC 방식 종류** - 로그 기반 vs 트리거 기반 특징
6. **데이터 품질 6대 요소** - ISO/IEC 25012 기준
7. **스키마 유형** - 스타 vs 스노우플레이크 구조와 차이
8. **정규화 기법** - Min-Max vs Z-Score 수식
9. **Lambda vs Kappa 아키텍처** - 구성과 장단점
10. **Kafka 구성 요소** - Topic, Partition, Offset, Consumer Group

---

*작성 기준: ADP 시험 출제 범위 | 한국데이터산업진흥원 ADP 자격검정 공식 범위 참고*
