# JPA 프로그래밍 - 4장 엔티티 매핑

## 1. @Entity

- JPA를 사용해서 테이블과 매핑할 클래스는 반드시 `@Entity` 어노테이션을 붙여야 한다.
- `@Entity`가 붙은 클래스는 JPA가 관리하며, 이를 엔티티라 부른다.

### 주의사항
- 기본 생성자는 필수 (파라미터가 없는 public 또는 protected 생성자)
- final 클래스, enum, interface, inner 클래스에는 사용 불가
- 저장할 필드에 final 사용 불가

---

## 2. @Table

- `@Table`은 엔티티와 매핑할 테이블을 지정한다.
- 생략하면 엔티티 이름을 테이블 이름으로 매핑한다.

### 주요 속성
| 속성 | 기능 | 기본값 |
|---|---|---|
| name | 매핑할 테이블 이름 | 엔티티 이름 사용 |
| catalog | catalog 기능이 있는 DB에서 catalog 매핑 | |
| schema | schema 기능이 있는 DB에서 schema 매핑 | |
| uniqueConstraints | DDL 생성 시 유니크 제약조건 생성 | |

---

## 3. 데이터베이스 스키마 자동 생성

- JPA는 `hibernate.hbm2ddl.auto` 속성을 이용해 애플리케이션 실행 시점에 DDL을 자동으로 생성해주는 기능을 지원한다.
- 대표적인 옵션: `create`, `create-drop`, `update`, `validate`, `none`
- 이 기능으로 생성된 DDL은 데이터베이스 방언(Dialect)에 따라 적절한 DDL을 만들어준다.
- 단, 운영 환경에서는 직접 검증한 DDL을 사용하는 것이 안전하며, 자동 생성 기능은 개발 단계에서만 사용하는 것이 권장된다.

---

## 4. DDL 생성 기능 (@Column 속성 활용)

- `@Column`의 속성들(`nullable`, `length`, `unique` 등)을 사용하면 DDL을 편리하게 자동 생성할 수 있다.
- **중요:** 이 기능은 어디까지나 DDL을 자동 생성할 때만 사용될 뿐이며, **JPA의 실행 로직(런타임 동작)에는 전혀 영향을 주지 않는다.**
  - 예: `nullable = false`로 설정해도 JPA가 애플리케이션 로직에서 자동으로 null 체크를 해주는 것은 아니다. 단지 DDL 생성 시 `NOT NULL` 제약조건이 추가될 뿐이다.

---

## 5. 기본 키 매핑 (기본키 생성 전략)

JPA가 제공하는 기본 키 생성 전략은 크게 두 가지로 나뉜다.

### (1) 직접 할당
- `@Id`만 사용하여 애플리케이션에서 기본 키를 직접 할당하는 방식

### (2) 자동 생성 (`@GeneratedValue`)
데이터베이스에 위임하여 기본 키를 자동으로 생성하는 방식으로, 아래 4가지 전략이 있다.

- **IDENTITY**: 기본 키 생성을 데이터베이스에 위임 (예: MySQL의 AUTO_INCREMENT)

```java
@Entity
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
}
```
> IDENTITY 전략은 데이터베이스에 INSERT SQL을 실행한 이후에야 기본 키 값을 알 수 있다. 그래서 JPA는 엔티티를 `persist()` 하는 시점에 즉시 INSERT SQL을 실행하고 DB에서 식별자를 조회한다.

---

- **SEQUENCE**: 데이터베이스 시퀀스를 사용하여 기본 키를 할당 (오라클 등에서 사용, `@SequenceGenerator` 필요)

```java
@Entity
@SequenceGenerator(
    name = "MEMBER_SEQ_GENERATOR",
    sequenceName = "MEMBER_SEQ", // 매핑할 데이터베이스 시퀀스 이름
    initialValue = 1,
    allocationSize = 1
)
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                     generator = "MEMBER_SEQ_GENERATOR")
    private Long id;

    private String name;
}
```
> `persist()` 호출 시 먼저 데이터베이스 시퀀스를 사용해서 식별자를 조회한 후, 그 값을 엔티티에 할당하고 영속성 컨텍스트에 저장한다.

---

- **TABLE**: 키 생성 전용 테이블을 만들어 시퀀스처럼 사용하는 전략 (모든 데이터베이스에 적용 가능, `@TableGenerator` 필요)

```java
@Entity
@TableGenerator(
    name = "MEMBER_SEQ_GENERATOR",
    table = "MY_SEQUENCES",       // 매핑할 테이블 이름
    pkColumnValue = "MEMBER_SEQ", // 키로 사용할 값 이름
    allocationSize = 1
)
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.TABLE,
                     generator = "MEMBER_SEQ_GENERATOR")
    private Long id;

    private String name;
}
```
```sql
CREATE TABLE MY_SEQUENCES (
    sequence_name varchar(255) NOT NULL,
    next_val bigint,
    PRIMARY KEY (sequence_name)
);
```
> 키 생성 전용 테이블을 별도로 만들고, 그 테이블의 컬럼을 시퀀스처럼 사용해서 기본 키를 생성한다. 모든 데이터베이스에 적용 가능하지만 테이블을 사용하므로 성능상 조금 불리할 수 있다.

---

- **AUTO**: 데이터베이스 방언에 따라 위 세 전략 중 하나를 자동으로 선택 (기본값)

```java
@Entity
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String name;
}
```
> `strategy` 값을 생략해도 기본값이 `AUTO`이므로 동일하게 동작한다. 선택된 데이터베이스 방언에 따라 IDENTITY, SEQUENCE, TABLE 중 하나가 자동으로 선택된다.

---

## 6. 기타 매핑 관련 어노테이션 (레퍼런스)

| 어노테이션 | 설명 |
|---|---|
| `@Column` | 컬럼을 매핑한다 (이름, 길이, nullable 여부 등 지정) |
| `@Enumerated` | enum 타입을 매핑한다 (`EnumType.ORDINAL` / `EnumType.STRING`) |
| `@Temporal` | 날짜 타입(`Date`, `Calendar`)을 매핑한다 (`DATE`, `TIME`, `TIMESTAMP`) |
| `@Lob` | BLOB, CLOB 타입과 매핑한다 |
| `@Access` | JPA가 엔티티 데이터에 접근하는 방식을 지정한다 (`FIELD` / `PROPERTY`) |

---

## 정리
- 엔티티 매핑의 기본은 `@Entity`와 `@Table`이며, 기본 키 생성 전략을 이해하는 것이 중요하다.
- DDL 자동 생성 관련 속성은 편의 기능일 뿐 런타임 동작에는 관여하지 않는다는 점을 꼭 기억할 것.
