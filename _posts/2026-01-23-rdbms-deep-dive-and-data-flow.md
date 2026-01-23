---
title: "MySQL 아키텍처 및 상세 설명 1편 - 아키텍처 및 데이터 플로우"
date: 2026-01-23
categories: [Database, MySQL]
tags: [mysql, innodb , architecture, storage-engine]
---

## 들어가며

```
SELECT * FROM wallet WHERE user_id = 1;
```

이 한 줄의 쿼리가 실행되고 결과가 돌아오기까지, 내부에서는 어떤 일이 벌어질까?

백엔드 개발을 하면서 수많은 쿼리를 작성했지만, 정작 그 쿼리가 어떤 경로를 거쳐 데이터를 가져오는지 깊이 고민해본 적은 많지 않았다. "인덱스 걸면 빨라진다", "트랜잭션으로 묶어야 안전하다"는 건 알지만, **왜** 그런지를 설명하려면 결국 MySQL의 내부 구조를 이해해야 한다.

이 글에서는 MySQL의 전체 아키텍처를 확인하고, 쿼리 한 줄이 어떻게 데이터가 되어 돌아오는지 그 과정을 확인해본다. 이 흐름을 이해하면 다음과 같은 질문에 답할 수 있게 된다.

- 왜 같은 쿼리인데 어떤 건 빠르고 어떤 건 느릴까?
- Buffer Pool 크기를 왜 신경 써야 할까?
- 트랜잭션 중간에 서버가 죽으면 데이터는 어떻게 될까?

---

## 목차

1. [Mysql Architecture: Application부터 Disk까지의 데이터흐름](#1-mysql-architecture와-application부터-disk까지의-데이터흐름)
2. [MySQL 엔진](#2-mysql-엔진)
  - 2.1 Connection Handler
  - 2.2 Parser
  - 2.3 Optimizer
  - 2.4 Executor
3. [InnoDB 스토리지 엔진](#3-innodb-스토리지-엔진)
  - 3.1 메모리 영역
  - 3.2 디스크 영역
4. [데이터 흐름 시나리오](#4-데이터-흐름-시나리오)
  - 4.1 SELECT: 읽기의 여정
  - 4.2 INSERT/UPDATE: 쓰기의 여정
5. [마무리: 인덱스는 왜 필요할까?](#5-마무리-인덱스는-왜-필요할까)

---

## 1. Mysql Architecture와 Application부터 Disk까지의 데이터흐름

### 1.1 MySQL 아키텍처 다이어그램
![MySQL Architecture Diagram](/assets/img/mysql-architecture-diagram.png)


위 그림은 많이 봤던 Mysql이라는 서버가 어떻게 구성 되어있는지를 간략하게 보여주는 전체적인 아키텍쳐이다.
위 그림을 기반으로 각 컴포넌트에 대허 **2장**과 **3장**에대해서 세부 컴포넌트를 설명하고
**4장**에서 실제 데이터가 Select, Insert/Update 에서 데이터가 어떤 방식으로 움직이는지 확인해보자

**핵심 포인트**: MySQL 엔진은 "어떻게 실행할지" 계획을 세우고, 스토리지 엔진은 "실제 데이터 읽기/쓰기"를 담당한다. 이 둘은 Handler API라는 인터페이스로 소통한다.

---

## 2. MySQL 엔진

![MySQL Engine Flow Diagram](/assets/img/mysql-engine-flow.png)
MySQL 엔진은 클라이언트의 요청을 받아서 **"어떻게 실행할 것인가"**를 결정한다.
주로 사람의 뇌에 비유하며 어떻게 최적화하여 업무를 진행할지 전달 하는 역할을 한다

### 2.1 Connection Handler

한줄 요약 : 연결 관리, 인증, 스레드 할당

클라이언트가 MySQL에 접속하면 제일 먼저 만나는 녀석이다. 하는 일은 단순하다.

1. **인증**: 너 누구야? 비밀번호 맞아?
2. **스레드 할당**: 오케이 통과, 너 담당할 스레드 하나 줄게

여기서 중요한 게 **Thread per Connection** 모델이다. MySQL은 기본적으로 커넥션 하나당 스레드 하나를 할당한다. 그래서 동시 접속이 1000개면 스레드도 1000개가 뜬다.

```sql
-- 현재 연결 수 확인
SHOW STATUS LIKE 'Threads_connected';

-- 최대 연결 수 설정
SHOW VARIABLES LIKE 'max_connections';  -- 기본값 151
```

근데 스레드를 매번 생성하고 삭제하는 건 비용이 크다. 그래서 **Thread Cache**라는 게 있다. 연결이 끊어져도 스레드를 바로 죽이지 않고 캐시에 보관했다가 재사용한다.

```sql
-- Thread Cache 설정
SHOW VARIABLES LIKE 'thread_cache_size';
```

#### 그렇다면 실제 앱에서는 어떻게 사용할까?

실제 앱에서는 Mysql, Nosql, HTTP 연결을 맺을때 추후 지속적으로 사용 할 것이라 생각이 들면  Connection Pool을 만들어 사용한다. 

예를들면, NestJS의 TypeORM이나 Spring의 HikariCP, 외부 HTTP 통신 Connection Pool과 같이 매번 커넥션을 맺지 않는다

왜냐하면, DB 커넥션, HTTP 커넥션을 맺는 것 자체가 무거운 작업이기 때문이다. 

TCP 3-way handshake, 인증, 스레드 할당... 매 요청마다 이걸 반복하면 오버헤드가 많아 지게 된다.


```typescript
// NestJS TypeORM 설정 예시
TypeOrmModule.forRoot({
  type: 'mysql',
  host: 'localhost',
  poolSize: 10,  // 커넥션 10개 유지
  // ...
})
```

### 2.2 Parser

한줄 요약 : SQL 문법 검사, 파싱 트리 생성

Parser는 두 단계로 동작한다.

**1단계: Lexical Analysis (어휘 분석)**

SQL 문자열을 의미 있는 단위(토큰)로 쪼갠다.

```sql
SELECT * FROM wallet WHERE user_id = 1;
```

이게 이렇게 쪼개진다:
```
[SELECT] [*] [FROM] [wallet] [WHERE] [user_id] [=] [1] [;]
```

**2단계: Syntax Analysis (구문 분석)**

토큰들이 SQL 문법에 맞는지 검사하고, 파싱 트리를 만든다.

```
         [SELECT Statement]
              /        \
      [SELECT *]    [FROM wallet]
                          \
                    [WHERE user_id = 1]
```

여기서 문법이 틀리면 바로 에러가 난다.

```sql
-- 이런 쿼리는 Parser에서 바로 튕김
SELEC * FROM wallet;  -- ERROR 1064: You have an error in your SQL syntax
```

사실 Parser는 단순한 녀석이다. "문법이 맞냐 틀리냐"만 본다. 테이블이 진짜 있는지, 컬럼명이 맞는지는 여기서 안 본다. 그건 다음 단계(Pre-processor)에서 체크한다.


### 2.3 Optimizer ⭐

한줄 요약 : **가장 중요한 컴포넌트**. 쿼리를 바로 실행하지 않고, 먼저 **실행 계획(Execution Plan)**을 수립한다.

많은 개발자들이 "쿼리 날리면 바로 실행되겠지"라고 생각하는데, 아니다. MySQL은 쿼리를 받으면 **"이걸 어떻게 실행하는 게 가장 빠를까?"**를 먼저 고민한다.

이때 Parser가 분석한 파싱 트리가 이용 된다 왜냐하면 mysql 자체는 사람이 작성한 query를 이해 할 수 없기 때문이다. 

이에 따라 Optimizer는 만들어진 파싱트리를 이용하여 최적화된 경로를 찾는다. 

예를 들어 이런 쿼리가 있다고 치자:

```sql
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.country = 'KR' AND o.status = 'COMPLETED';
```

Optimizer는 이런 걸 고민한다:
- users 먼저 읽을까, orders 먼저 읽을까?
- country 인덱스를 탈까, status 인덱스를 탈까?
- 아니면 그냥 Full Scan이 나을까?

#### Cost-Based Optimizer

MySQL의 Optimizer는 **비용 기반(Cost-Based)**이다. 각 실행 방법의 "비용"을 계산해서 가장 낮은 걸 선택한다.

비용 계산에 쓰이는 정보:
- 테이블의 전체 row 수
- 인덱스의 cardinality (고유값 개수)
- 데이터 분포 통계

```sql
-- 테이블 통계 정보 확인
SHOW TABLE STATUS LIKE 'orders';

-- 인덱스 통계 확인
SHOW INDEX FROM orders;
```

#### EXPLAIN: 실행 계획 까보기

Optimizer가 어떤 계획을 세웠는지 보려면 `EXPLAIN`을 쓰면 된다.

```sql
EXPLAIN SELECT * FROM wallet WHERE user_id = 1;
```

```
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------+
| id | select_type | table  | type | possible_keys | key     | key_len | ref   | rows | Extra |
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | wallet | ref  | idx_user_id   | idx_user_id | 4    | const |    1 |       |
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------+
```

핵심 컬럼들:
- **type**: 접근 방식. `ALL`(Full Scan)이면 위험, `ref`나 `const`면 좋음
- **key**: 실제로 사용한 인덱스
- **rows**: 예상 스캔 row 수 (적을수록 좋음)

```sql
-- 더 자세한 정보
EXPLAIN ANALYZE SELECT * FROM wallet WHERE user_id = 1;
```


### 2.4 Executor
한줄 요약 : 실행 계획대로 스토리지 엔진 호출

Executor는 심플하다. Optimizer가 짜준 실행 계획을 그대로 실행하는 컴포넌트이다.

```
Executor의 역할:
1. 실행 계획 받음
2. 스토리지 엔진(InnoDB)에게 "이 데이터 줘" 요청
3. 결과 받아서 조합
4. 클라이언트에게 반환
```

Executor와 스토리지 엔진은 **Handler API**라는 인터페이스로 통신한다. 이 덕분에 MySQL은 여러 스토리지 엔진(InnoDB, MyISAM 등)을 플러그인처럼 갈아끼울 수 있다.

---

## 3. InnoDB 스토리지 엔진

InnoDB는 MySQL의 기본 스토리지 엔진이다. **트랜잭션 지원, Row-level Lock, MVCC** 등 중요한 기능을 제공한다.

### 3.1 메모리 영역

### 3.2 디스크 영역


## 4. 데이터 흐름 시나리오

이제 실제로 쿼리가 실행될 때 어떤 일이 벌어지는지 따라가보자.

### 4.1 SELECT: 읽기의 여정

### 4.2 INSERT/UPDATE: 쓰기의 여정


## 5. 마무리: 인덱스는 왜 필요할까?

지금까지 MySQL의 전체 아키텍처와 데이터 흐름을 살펴봤다.

### 핵심 정리

| 구성요소 | 역할 | 기억할 포인트 |
|---------|------|--------------|
| MySQL 엔진 | 쿼리 분석 & 실행 계획 | Optimizer가 "어떻게" 실행할지 결정 |
| InnoDB 엔진 | 실제 데이터 처리 | Buffer Pool이 성능의 핵심 |
| Redo Log | Crash Recovery | 커밋 전에 먼저 기록 (WAL) |
| Undo Log | 롤백 & MVCC | 트랜잭션 격리 수준의 기반 |

### 남은 질문

이 글에서 의도적으로 깊게 다루지 않은 부분이 있다.

> **"Optimizer는 어떻게 '인덱스를 타겠다'고 결정하는 걸까?"**

4.1의 SELECT 흐름에서 ④번 단계를 다시 보자:

```
[Optimizer]
     │ 
     │ ④ 실행 계획 수립
     │    "user_id에 인덱스가 있네? 인덱스 스캔하자"
     │    (인덱스가 없다면? → Full Table Scan 😱)
```

- 인덱스가 있으면 특정 데이터를 빠르게 찾을 수 있다
- 인덱스가 없으면 테이블 전체를 뒤져야 한다

그렇다면:
- 인덱스는 내부적으로 어떤 구조인가? (B+Tree)
- 왜 하필 B+Tree인가?
- Clustered Index와 Secondary Index는 뭐가 다른가?
- Optimizer는 인덱스를 언제 타고, 언제 안 타는가?

**다음 편: MySQL 인덱스 - B+Tree**에서 이어서 다룬다.

---

## 참고 자료

- [MySQL 8.0 Reference Manual - InnoDB Architecture](https://dev.mysql.com/doc/refman/8.0/en/innodb-architecture.html)
- 백은빈, 이성욱 - 『Real MySQL 8.0』 (위키북스)
- Baron Schwartz 외 - 『High Performance MySQL』 (O'Reilly)
- [Percona Blog - InnoDB Internals](https://www.percona.com/blog/)

---
