# jpa-deepdive-0914

JPA(Hibernate)로 대량의 엔티티를 저장할 때, ID 생성 전략과 저장 방식에 따라 성능(실행 시간 / 쿼리 수 / 메모리)이 어떻게 달라지는지 직접 측정하고 원인을 분석하는 실습 프로젝트입니다.

## 목표

`Member` 엔티티 1만(10,000) 건을 대량 저장하는 상황을 가정하고, 아래 방식들을 비교합니다.

- ID 생성 전략: `IDENTITY` vs `SEQUENCE`
- 저장 방식: for문 + `save()` 1건씩 vs `saveAll()` 한 번에
- Hibernate JDBC batch insert 설정(`hibernate.jdbc.batch_size`, `order_inserts`) 적용 여부
- `SEQUENCE` 전략의 `allocationSize` 값(1 vs 100)에 따른 시퀀스 조회 횟수 차이

각 방식마다 실행 시간, 실제 나간 INSERT 쿼리 수, SEQUENCE 조회(`nextval`) 횟수, (가능하면) 힙 메모리 사용량을 측정하고, 왜 그런 결과가 나오는지 코드/쿼리 로그 기반으로 설명합니다.

> 참고: 여기서 말하는 "batch"는 **Spring Batch 프레임워크가 아닙니다.** Spring Batch(Job/Step/Chunk)는 이 프로젝트에 포함되어 있지 않고, `hibernate.jdbc.batch_size`는 Hibernate가 JDBC의 `addBatch()`/`executeBatch()`를 사용하도록 켜는 설정값일 뿐입니다.

## 기술 스택

- Spring Boot 4.1.1, Java 21, Gradle
- Spring Data JPA (Hibernate 7)
- MySQL 8.0 (Docker Compose로 실행)
- Lombok

## 비교 시나리오 (1~7)

| # | 전략 | 저장 방식 | 추가 설정 |
|---|------|-----------|-----------|
| 1 | IDENTITY | for + `save()` | - |
| 2 | SEQUENCE (allocationSize=1) | for + `save()` | - |
| 3 | IDENTITY | `saveAll()` | - |
| 4 | SEQUENCE (allocationSize=1) | `saveAll()` | - |
| 5 | SEQUENCE (allocationSize=1) | `saveAll()` | `hibernate.jdbc.batch_size=100`, `order_inserts=true` |
| 6 | SEQUENCE (allocationSize=100) | `saveAll()` | `hibernate.jdbc.batch_size=100`, `order_inserts=true` |
| 7 | SEQUENCE (allocationSize=100) | 주기적 `flush()`+`clear()` (100건마다) | 위와 동일 |

- **1 vs 3, 2 vs 4**: 같은 ID 전략에서 "건건이 save()" vs "saveAll() 한 번에"의 차이 — saveAll()은 하나의 트랜잭션으로 묶여서 커밋 횟수 자체가 줄어든다.
- **1·3 vs 2·4**: 같은 저장 방식에서 IDENTITY vs SEQUENCE(allocationSize=1)의 차이 — IDENTITY는 PK를 DB가 즉시 채번해줘야 해서 Hibernate가 insert를 배치로 못 묶는다.
- **4 vs 5**: batch_size=100을 켰을 때 SEQUENCE(allocationSize=1)가 실제로 빨라지는지 — allocationSize=1이면 row마다 시퀀스 테이블 조회가 필요해서 batch_size를 켜도 그 조회 자체는 못 묶인다.
- **5 vs 6**: 같은 batch_size 설정에서 allocationSize만 1→100으로 바꿨을 때 시퀀스 테이블 조회 횟수와 시간이 얼마나 줄어드는지.
- **6 vs 7**: 같은 설정에서 saveAll()(끝까지 영속성 컨텍스트에 다 들고 있음) vs 주기적 flush()+clear()(100건마다 비움)의 힙 메모리 사용량 차이.

## 프로젝트 구조

```
src/main/java/com/study/jpadeepdive/
├── domain/
│   ├── MemberIdentity.java          # @GeneratedValue(strategy = IDENTITY)
│   ├── MemberSequence.java          # @GeneratedValue(strategy = SEQUENCE), allocationSize = 1
│   └── MemberSequenceAllocated.java # SEQUENCE, allocationSize = 100
├── repository/
│   ├── MemberIdentityRepository.java
│   ├── MemberSequenceRepository.java
│   └── MemberSequenceAllocatedRepository.java
└── support/
    ├── SqlStatementCounter.java              # JDBC 레벨 카운터 (prepareStatement/addBatch/executeBatch/executeUpdate/executeQuery)
    ├── CountingDataSource.java               # 실제 DataSource를 감싸는 카운팅 프록시(JDK 동적 프록시)
    └── CountingDataSourceBeanPostProcessor.java # 위 프록시를 Spring이 만든 DataSource 빈에 꽂아준다

src/test/java/com/study/jpadeepdive/
├── support/
│   ├── BenchmarkResult.java   # 시나리오 1건의 측정 결과(record)
│   ├── BenchmarkReport.java   # 결과를 System.out.println으로 콘솔에 출력
│   ├── MemoryMeasurer.java    # Runtime 기반 힙 사용량 측정
│   └── SqlStatementCounterSanityTest.java # 카운터가 실제로 잘 세는지 확인하는 점검용 테스트
└── scenario/
    ├── IdentityInsertTest.java              # 시나리오 1, 3 (IDENTITY: for-loop vs saveAll)
    ├── SequenceInsertTest.java              # 시나리오 2, 4 (SEQUENCE alloc=1: for-loop vs saveAll)
    ├── SequenceBatchInsertTest.java         # 시나리오 5 (SEQUENCE alloc=1 + batch_size=100)
    ├── SequenceBatchAllocatedInsertTest.java# 시나리오 6 (SEQUENCE alloc=100 + batch_size=100)
    ├── SequenceFlushClearInsertTest.java    # 시나리오 7 (alloc=100 + batch_size=100 + 주기적 flush/clear)
    └── IdentityBatchInsertTest.java         # 보충 시나리오 (IDENTITY + batch_size=100 → 무시됨을 증명)
```

`allocationSize`는 어노테이션 값이라 런타임에 바꿀 수 없기 때문에, allocationSize=1과 100을 비교하기 위해 엔티티(및 시퀀스)를 별도로 분리했습니다.

### 쿼리 카운팅 방식

처음엔 Hibernate `StatementInspector`로 세려 했지만, MySQL에서 SEQUENCE 전략이 실제 시퀀스가 아니라 **테이블 하나(`*_seq`)로 에뮬레이션**되는데(MySQL은 `CREATE SEQUENCE`를 지원하지 않음), 그 테이블에 대한 내부 SELECT/UPDATE 쿼리는 `StatementInspector`를 거치지 않고 나간다는 걸 발견했습니다. 그래서 더 낮은 레벨인 **JDBC `DataSource`/`Connection`/`PreparedStatement`를 동적 프록시로 감싸서** `prepareStatement` / `addBatch` / `executeBatch` / `executeUpdate` / `executeQuery` 호출 자체를 세는 방식(`CountingDataSource`)으로 바꿨습니다. Hibernate 내부 구현에 기대지 않아 더 정확하고, 카운터도 테이블별 long 값만 유지해서 10만 건을 세도 메모리 비용이 늘지 않습니다.

## 실행 환경 준비

### 1. MySQL 컨테이너 실행

```bash
docker compose up -d
```

- 컨테이너: `jpa-deepdive-mysql` (MySQL 8.0)
- 접속 정보: DB `jpa_deepdive`, 계정 `study` / `study1234`
- **호스트 포트는 3307**입니다 (컨테이너 내부는 3306). 이 PC에 이미 로컬 MySQL/MariaDB 서비스가 3306을 점유하고 있어서 충돌을 피하려고 옮겼습니다. 다른 환경에서 그대로 쓰려면 `docker-compose.yml`의 포트 매핑과 `application.properties`의 `spring.datasource.url` 포트를 함께 맞춰야 합니다.
- healthcheck가 `healthy`가 될 때까지 기다린 후 애플리케이션/테스트를 실행하세요.

```bash
docker compose ps
```

### 2. 빌드 / 테스트

```bash
./gradlew.bat test
```

### 3. 벤치마크 시나리오만 실행

```bash
./gradlew.bat test --tests "com.study.jpadeepdive.scenario.*" -Dbenchmark.rowCount=10000
```

- `-Dbenchmark.rowCount`로 시나리오당 저장할 row 수를 조절할 수 있습니다 (아무 값도 안 주면 기본값 10000).
- for-loop + `save()` 시나리오(1, 2번)는 매 건마다 트랜잭션이 커밋돼 row 수에 거의 비례해서 느려집니다. 10만 건 그대로 돌리면 이 두 시나리오만 합쳐서 1시간 가까이 걸릴 수 있어, 실제 측정은 **10,000건** 기준으로 진행했습니다.
- 결과는 `[BENCHMARK] ...` 형태로 콘솔에 한 줄씩 출력됩니다. 파일에 따로 남기지 않으므로, IDE의 테스트 실행 아이콘(gutter run)으로 개별 시나리오를 돌려도 바로 출력 패널에서 확인할 수 있습니다.

## 실험 결과 (1만 건 기준)

| # | 시나리오 | 시간(ms) | prepareStmt | addBatch | executeBatch | executeUpdate | executeQuery | INSERT문 종류 | 시퀀스테이블 조회 | 힙증가(MB) |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | IDENTITY + for-loop | 103,761 | 10,000 | 0 | 0 | 10,000 | 0 | 10,000 | 0 | 0.54 |
| 2 | SEQUENCE(1) + for-loop | 214,225 | 30,000 | 0 | 0 | 20,000 | 10,000 | 10,000 | 20,000 | -0.84 |
| 3 | IDENTITY + saveAll() | 5,731 | 10,000 | 0 | 0 | 10,000 | 0 | 10,000 | 0 | 0.23 |
| 4 | SEQUENCE(1) + saveAll() | 107,197 | 30,000 | 0 | 0 | 20,000 | 10,000 | 10,000 | 20,000 | 0.24 |
| 5 | SEQUENCE(1) + batch100 + saveAll() | 100,436 | 20,001 | 10,000 | 100 | 10,000 | 10,000 | 1 | 20,000 | 0.28 |
| 6 | SEQUENCE(100) + batch100 + saveAll() | **1,910** | 203 | 10,000 | 100 | 101 | 101 | 1 | 202 | 0.36 |
| 7 | SEQUENCE(100) + batch100 + flush/clear | **1,597** | 300 | 10,000 | 100 | 100 | 100 | 100 | 200 | 0.07 |
| X* | IDENTITY + batch100(무시됨) + saveAll() | 7,706 | 10,000 | 0 | 0 | 10,000 | 0 | 10,000 | 0 | 0.70 |

\* X번은 표에 없는 보충 시나리오로, "IDENTITY는 batch_size를 켜도 무시된다"는 걸 직접 증명하기 위해 추가한 테스트다([IdentityBatchInsertTest.java](src/test/java/com/study/jpadeepdive/scenario/IdentityBatchInsertTest.java)).

위 표는 테스트 실행 시 콘솔에 출력된 `[BENCHMARK] ...` 로그를 수동으로 정리한 것이다 (결과를 파일로 자동 저장하지는 않는다).

## 원인 분석

### 1. `saveAll()`이 `for + save()`보다 압도적으로 빠른 이유 (1→3: 18배, 2→4: 2배)

두 시나리오의 쿼리 개수(`prepareStatement`/`executeUpdate`)는 **완전히 동일**하다. 차이는 쿼리 수가 아니라 **트랜잭션(커밋) 횟수**에 있다. `JpaRepository.save()`는 그 자체가 `@Transactional`이라 트랜잭션 없이 for문에서 호출하면 호출마다 트랜잭션을 열고 커밋한다. 반면 `saveAll()`도 자기 자신이 `@Transactional`이라 내부에서 N번 `persist()`를 호출해도 커밋은 메서드 끝에서 딱 1번이다.

MySQL InnoDB는 기본 설정(`innodb_flush_log_at_trx_commit=1`)에서 커밋마다 redo log를 디스크에 fsync한다. 이 fsync 자체가 고정 비용이라, 커밋을 1만 번 하면 그 비용도 1만 번 곱해진다. `for + save()`가 느린 건 "쿼리가 안 묶여서"가 아니라 **"커밋이 안 묶여서"**다.

### 2. IDENTITY는 왜 batch insert가 안 되는가

`MemberIdentity` + `batch_size=100` + `saveAll()`(위 표 X번)로 직접 확인한 결과, `addBatch()`가 **단 한 번도 호출되지 않았다.** Hibernate가 이 설정을 조용히 무시한다는 뜻이다.

이유는 구조적이다. `IDENTITY`는 PK 채번을 DB의 `AUTO_INCREMENT`에 위임하는데, JDBC 배치(`addBatch()`+`executeBatch()`)는 "여러 SQL을 큐에 담아뒀다가 한꺼번에 실행"하는 방식이라 **실행 전까지는 각 row의 PK 값을 알 수 없다.** 그런데 Hibernate는 `persist()` 직후 해당 엔티티의 식별자를 영속성 컨텍스트에 즉시 등록해야 한다(1차 캐시 키, 연관관계 매핑 등에 필요). 그래서 IDENTITY 전략에서는 Hibernate가 매 insert를 즉시 실행하고 `getGeneratedKeys()`로 그 자리에서 PK를 받아온다 — 큐잉 자체가 불가능한 구조라 `batch_size` 설정을 아예 무시하도록 되어 있다(Hibernate 공식 문서에 명시된 제약).

### 3. SEQUENCE(allocationSize=1)는 왜 batch_size를 켜도 거의 안 빨라지는가 (4번 107s vs 5번 100s)

5번 결과를 보면 `addBatch=10000`, `executeBatch=100`으로 **INSERT 자체는 정상적으로 100개씩 배치로 묶였다.** 그런데도 4번과 시간이 비슷한 이유는 `executeQuery=10000`, `시퀀스테이블 조회=20000`에 있다.

MySQL은 `CREATE SEQUENCE`를 지원하지 않아서 Hibernate가 `SEQUENCE` 전략을 **`member_sequence_seq`라는 테이블 하나로 에뮬레이션**한다:

```sql
select next_val as id_val from member_sequence_seq for update;  -- executeQuery
update member_sequence_seq set next_val = ? where next_val = ?; -- executeUpdate
```

`allocationSize=1`이면 row 1개당 이 SELECT+UPDATE 쌍이 매번 실행된다. PK 값을 미리 확보하기 위한 동기적 요청이라 insert처럼 "일단 큐에 쌓아놓고 나중에 한꺼번에" 처리할 수 없다. 결국 `batch_size=100`은 INSERT 문은 잘 묶어줬지만, 그 앞단의 시퀀스 조회 2만 번은 그대로 남아 전체 시간을 지배한다. IDENTITY의 "PK를 즉시 알아야 하는 문제"가 형태만 바뀌어 SEQUENCE(allocationSize=1)에도 그대로 나타나는 셈이다.

### 4. allocationSize=100이 왜 이렇게 극적인가 (5번 100s → 6번 1.9s, 53배)

`allocationSize=100`은 Hibernate에게 "시퀀스 값을 100개씩 미리 당겨놓고 메모리에서 하나씩 소진하라"고 지시하는 것이다(hi-lo 최적화). 그 결과 시퀀스 테이블 왕복이 20,000 → 202로 줄었다(10,000/100 ≈ 100번의 할당). 나머지는 메모리에서 `id = base + offset`으로 계산되어 DB 왕복이 없다. 여기에 `batch_size=100`으로 INSERT까지 묶이므로(`executeBatch=100`), 6번은 네트워크 왕복이 총 ~200번뿐이다 — 5번의 10,100여 번 대비 압도적으로 적다.

> MySQL이라 이 차이가 두드러진다. Oracle/PostgreSQL처럼 진짜 SEQUENCE 객체를 지원하는 DB는 `NEXTVAL`이 락 없는 원자적 연산이라 allocationSize=1이어도 이 정도로 느리지 않다. MySQL은 시퀀스를 테이블+락(`FOR UPDATE`)으로 흉내 내기 때문에 allocationSize가 성능에 미치는 영향이 훨씬 크다.

### 5. flush()+clear()가 메모리에 미치는 영향 (6번 0.36MB vs 7번 0.07MB)

`saveAll()`(6번)은 트랜잭션이 끝날 때까지 1만 개 엔티티를 전부 영속성 컨텍스트에 들고 있다. Hibernate는 관리 중인 엔티티마다 더티 체킹용 스냅샷(로드 시점 상태 복사본)을 별도로 유지하므로, 엔티티 수만큼 이 오버헤드가 쌓인다.

7번은 100건마다 `entityManager.flush()`로 DB에 반영한 뒤 `clear()`로 영속성 컨텍스트를 비우므로, 동시에 관리되는 엔티티가 항상 100개 이하로 유지된다. 그 결과 힙 증가량이 6번의 1/5 수준(0.07MB)으로 나타났다. 다만 1만 건 규모에서는 절대값 자체가 작아 신호가 약한 편이라(6번도 0.36MB), 차이가 확실히 벌어지는 걸 보려면 row 수를 훨씬 늘리거나(수십만 건) 엔티티 필드 수를 늘려야 더 뚜렷하게 드러날 것이다. 2번 시나리오의 힙 증가량이 음수(-0.84)로 나온 것도 `Runtime` 기반 측정(`System.gc()` 이후 측정)의 한계로, 정밀한 프로파일러 없이는 노이즈를 완전히 제거하기 어렵다.

## 면접용 요약

### 한 줄 요약

JPA 대량 INSERT 성능을 좌우하는 건 "쿼리 문법"이 아니라 **① 트랜잭션(커밋) 횟수**와 **② PK 값을 언제 알아야 하는가**이며, ID 생성 전략(IDENTITY vs SEQUENCE)과 그 세부 설정(batch_size, allocationSize)이 이 두 가지에 어떤 영향을 주는지 MySQL 환경에서 실측했다.

### 핵심 결론 5가지 (결론 → 근거)

1. **`saveAll()`이 `for + save()`보다 빠른 건 "배치" 때문이 아니라 "커밋 횟수" 때문이다.**
   근거: IDENTITY 기준 두 방식의 쿼리 수(prepare=10,000, executeUpdate=10,000)는 완전히 동일한데 시간은 18배 차이(약 100s vs 5.7s). `save()`는 호출마다 자체 트랜잭션이라 커밋 1만 번, `saveAll()`은 메서드 전체가 트랜잭션 1개라 커밋 1번.

2. **IDENTITY 전략은 JDBC batch insert가 구조적으로 불가능하다.**
   근거: `batch_size=100`을 켜도 `addBatch=0`, `executeBatch=0`으로 무시됨(직접 실측). IDENTITY는 insert 즉시 DB가 채번한 PK를 알아야 하는데, JDBC batch는 "다 모았다가 한꺼번에 실행"하는 방식이라 구조적으로 충돌한다.

3. **SEQUENCE라도 `allocationSize=1`이면 `batch_size`가 사실상 무용지물이다.**
   근거: SEQUENCE(alloc=1)+batch_size=100(약 100s)이 batch_size 없는 버전(약 107s)과 거의 차이 없음. MySQL은 진짜 SEQUENCE 객체가 없어 테이블(`SELECT ... FOR UPDATE` + `UPDATE`)로 흉내 내는데, allocationSize=1이면 이 조회가 row마다 발생해 배치가 못 붙는다.

4. **`allocationSize`를 100으로 올리면 53배 빨라진다 — 진짜 병목은 `batch_size`가 아니라 `allocationSize`였다.**
   근거: 약 100s → 1.9s. 시퀀스 테이블 왕복이 20,000번 → 202번으로 감소.

5. **`flush()`+`clear()`는 영속성 컨텍스트 메모리 관리에 도움이 된다.**
   근거: `saveAll()`(0.36MB) vs 100건마다 flush+clear(0.07MB). 다만 1만 건 규모에서는 신호가 약해서, 대용량일수록 차이가 더 뚜렷할 것으로 예상.

### 예상 꼬리질문

- **"그럼 실무에서는 대량 저장을 어떻게 하나요?"**
  → SEQUENCE 전략 + `allocationSize`를 적당히 크게(100 등) + `hibernate.jdbc.batch_size`/`order_inserts` + `saveAll()`. 건수가 아주 많다면 주기적 `flush()`+`clear()`까지 추가해서 영속성 컨텍스트가 무한정 커지지 않게 관리한다.

- **"그럼 IDENTITY는 언제 쓰나요?"**
  → 대량 삽입이 드물고 일반적인 CRUD 위주인 서비스라면 IDENTITY도 무방하다(구현이 단순하고 DB가 알아서 채번). 대량 배치 저장이 잦다면 SEQUENCE + allocationSize 조합이 유리하다.

- **"MySQL이 아니라 Oracle/PostgreSQL이면 결과가 다를까요?"**
  → SEQUENCE가 진짜 DB 객체인 DB에서는 `NEXTVAL`이 락 없는 원자적 연산이라 allocationSize=1이어도 지금처럼 느리지 않을 가능성이 높다. 지금 결과가 극적으로 갈린 건 MySQL이 SEQUENCE를 테이블+락으로 흉내 내기 때문이라, "MySQL에서는 SEQUENCE도 결국 row-lock 기반 카운터 테이블"이라는 전제가 깔려 있다는 점을 짚어줘야 한다.

### 실험의 한계

- 힙 메모리 측정은 `Runtime.getRuntime()` + `System.gc()` 기반 근사치다. 정밀한 프로파일러(JFR, async-profiler 등)로 측정한 게 아니라서 시나리오 2번처럼 음수 델타가 나오는 등 노이즈가 있다.
- 원래 계획은 10만 건이었지만 for-loop 시나리오의 실행 시간 문제로 1만 건으로 축소해서 측정했다.
- 로컬 Docker MySQL 단일 인스턴스, localhost 통신 기준이라 실제 운영 환경(네트워크 지연, 동시 부하)과는 수치가 다를 수 있다. 다만 "무엇이 병목의 원인인가"에 대한 정성적 결론(커밋 횟수, PK 채번 시점, allocationSize)은 환경이 달라져도 유효하다.

