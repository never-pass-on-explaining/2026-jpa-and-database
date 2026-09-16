# Chapter 4. 엔티티 매핑

## 1. @Entity / @Table

- `@Entity`: 테이블과 매핑할 클래스에 필수로 붙인다. 이 클래스는 JPA가 관리하며 **엔티티**라 부른다.
- `@Table`: 매핑할 테이블을 지정한다. 생략하면 엔티티 이름을 테이블 이름으로 사용한다.

> 스프링 부트는 카멜 표기법을 언더스코어로 자동 변환한다.
> `OrderItem` → `order_item`, `roleType` → `role_type`

### 주의사항

- **기본 생성자 필수** (파라미터 없는 `public` 또는 `protected`)
- `final` 클래스, enum, interface, inner 클래스 불가

**기본 생성자가 필요한 이유**

JPA는 DB에서 조회한 데이터로 객체를 만들 때 **빈 생성자로 객체를 먼저 만들고 값을 채운다.**

자바는 생성자를 하나도 선언하지 않으면 컴파일러가 기본 생성자를 자동으로 넣어준다. 하지만 **생성자를 하나라도 직접 선언하면 만들어주지 않는다.**

```java
protected Member() {}              // JPA가 객체를 만들 때 사용. 직접 선언해야 한다

public Member(String name) {       // 생성자를 하나라도 선언하면
    this.name = name;              // 컴파일러가 기본 생성자를 만들어주지 않는다
}
```

> 실무에서는 `@NoArgsConstructor(access = AccessLevel.PROTECTED)`로 처리한다.
> `protected`로 막아 외부에서 빈 객체를 만들지 못하게 하면서 JPA 요구사항은 충족시킨다.

## 2. 데이터베이스 스키마 자동 생성

JPA는 **매핑 정보와 데이터베이스 방언**을 사용해 스키마를 자동 생성한다.

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: create
    show-sql: true
```

| 옵션 | 설명 |
| --- | --- |
| `create` | 기존 테이블 삭제 후 새로 생성 (DROP + CREATE) |
| `create-drop` | create + 종료 시 DDL 제거 |
| `update` | 테이블과 매핑정보를 비교해 **변경 사항만** 수정 |
| `validate` | 차이가 있으면 **애플리케이션을 실행하지 않음**. DDL은 수정하지 않는다 |
| `none` | 사용 안 함 |

### 운영에서 절대 쓰면 안 되는 것

**`create`, `create-drop`, `update`** — 운영 중인 테이블이나 컬럼을 삭제할 수 있다.

| 환경 | 권장 |
| --- | --- |
| 개발 초기 | `create` / `update` |
| 테스트 서버 | `update` / `validate` |
| 스테이징, 운영 | `validate` / `none` |

운영에서 `validate`를 쓰는 이유는 **매핑과 실제 테이블이 다르면 아예 뜨지 않게** 만들어 사고를 막기 위해서다.

## 3. DDL 생성 기능

`@Column`의 일부 속성은 자동 생성되는 DDL에 제약조건을 추가한다.

```java
@Column(name = "NAME", nullable = false, length = 10)
private String username;
```
```sql
NAME varchar(10) not null
```

### DDL 생성에만 쓰인다

`nullable`, `length` 같은 속성은 **DDL 자동 생성에만 사용되고 JPA 실행 로직에는 영향을 주지 않는다.**

`nullable = false`라고 해서 JPA가 애플리케이션에서 null 체크를 해주지 않는다. 스키마를 직접 관리한다면 **엔티티만 보고 제약조건을 파악하는 문서 역할**로 남겨두는 정도다.

## 4. 기본 키 매핑

```java
@Id                    // 직접 할당
@GeneratedValue(...)   // 자동 생성
```

전략이 여러 개인 이유는 **DB마다 키 생성 방식이 다르기 때문**이다. 오라클과 PostgreSQL은 시퀀스를 제공하지만 MySQL은 없고 대신 `AUTO_INCREMENT`를 제공한다.

### IDENTITY

기본 키 생성을 DB에 위임한다. **MySQL, PostgreSQL, SQL Server**에서 사용한다.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

### 쓰기 지연이 동작하지 않는다

엔티티가 영속 상태가 되려면 식별자가 반드시 필요하다. 그런데 `IDENTITY`는 **DB에 저장해야 식별자를 알 수 있으므로**, `persist()` 호출 즉시 INSERT SQL이 나간다.

→ 대량 INSERT에서 `hibernate.jdbc.batch_size`를 설정해도 **배치가 묶이지 않는다.** MySQL + JPA 조합에서 배치 작업이 느린 원인이다.

### SEQUENCE

DB 시퀀스를 사용한다. **오라클, PostgreSQL, H2**에서 사용한다.

```java
@SequenceGenerator(name = "BOARD_SEQ_GENERATOR",
                   sequenceName = "BOARD_SEQ",
                   allocationSize = 1)

@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE,
                generator = "BOARD_SEQ_GENERATOR")
private Long id;
```

`persist()` 때 **시퀀스에서 식별자만 조회**하고 INSERT는 커밋까지 미룬다. 쓰기 지연이 정상 동작한다.

| | `persist()` 시점 | 쓰기 지연 |
| --- | --- | --- |
| `IDENTITY` | **INSERT 실행** → 식별자 획득 | 불가 |
| `SEQUENCE` | 시퀀스에서 **식별자만 조회** | 가능 |

### TABLE / AUTO

- **TABLE**: 키 생성 전용 테이블로 시퀀스를 흉내낸다. 모든 DB에서 가능하지만 **테이블을 조회하고 갱신해야 해서 성능이 떨어지고 락 경합이 생긴다.** 실무에서 거의 안 쓴다.
- **AUTO**: 방언에 따라 자동 선택 (기본값). 명시적이지 않아 직접 지정하는 편이 낫다.

## 5. 권장 식별자 전략

DB 기본 키의 3가지 조건 — `null` 아님, 유일함, **변하지 않음**

| 전략 | 설명 | 예 |
| --- | --- | --- |
| **자연 키**(natural key) | 비즈니스에 의미가 있는 키 | 주민등록번호, 이메일, 전화번호 |
| **대리 키**(surrogate key) | 비즈니스와 무관한 임의의 키 | 시퀀스, `auto_increment` |

### 자연 키보다 대리 키

전화번호는 없을 수도 있고 바뀔 수도 있다. 주민등록번호는 3조건을 만족하는 것처럼 보이지만, **정부 정책이 바뀌어 법적으로 저장할 수 없게 되면서** 이를 외래 키로 쓰던 테이블과 로직을 전부 수정해야 했던 사례가 있다.

> **비즈니스 요구사항은 계속 변하는데 테이블은 한 번 정의하면 변경하기 어렵다.**

이메일이나 주민등록번호처럼 자연 키 후보가 되는 컬럼은 **유니크 인덱스**를 걸어 쓰면 된다.

### 기본 키는 변경하면 안 된다

저장된 엔티티의 기본 키를 바꾸면 JPA는 예외를 던지거나 정상 동작하지 않는다. `setId()`를 외부에 공개하지 않는 것도 예방법이다.

## 6. 필드와 컬럼 매핑

| 어노테이션 | 설명 |
| --- | --- |
| `@Column` | 컬럼 매핑. `name`, `nullable`이 주로 쓰인다 |
| `@Enumerated` | enum 매핑 |
| `@Temporal` | `java.util.Date` 매핑용. `LocalDateTime`을 쓰면 안 붙여도 된다 |
| `@Lob` | `String` → CLOB, `byte[]` → BLOB |
| `@Transient` | 이 필드는 DB에 매핑하지 않는다 |
| `@Access` | 접근 방식 지정. `@Id` 위치로 결정되므로 보통 생략 |

### @Enumerated는 반드시 STRING

| 속성값 | 저장 방식 | |
| --- | --- | --- |
| `ORDINAL` | enum **순서** 저장 (ADMIN=0, USER=1) | **기본값** |
| `STRING` | enum **이름** 저장 ('ADMIN', 'USER') | |

`ADMIN`(0), `USER`(1) 사이에 enum이 추가되어 `ADMIN`(0), `NEW`(1), `USER`(2)가 되면

```
기존 데이터:  USER = 1
신규 데이터:  USER = 2     ← 같은 값인데 다른 숫자
```

**데이터가 조용히 어긋나고 에러도 나지 않는다.** 항상 `EnumType.STRING`을 쓴다.

### 자바 기본형에 @Column을 붙일 때

```java
int data1;        // @Column 생략  → data1 integer not null
@Column int data2;                // → data2 integer      ← not null이 사라진다
```

자바 기본형에는 null을 넣을 수 없으므로 JPA가 `not null`을 자동 추가하는데, `@Column`을 붙이면 `nullable` 기본값이 `true`라 빠진다. → `nullable = false`를 명시하는 게 안전하다.

## 정리

- DDL 관련 속성(`nullable`, `length`)은 **스키마 자동 생성에만 관여**하고 실행 로직에는 영향이 없다
- 운영에서 `ddl-auto`는 **`validate` 또는 `none`**
- `IDENTITY`는 `persist()` 시점에 INSERT가 나가 **쓰기 지연이 동작하지 않는다**
- 식별자는 **자연 키보다 대리 키**
- enum은 **반드시 `EnumType.STRING`**
