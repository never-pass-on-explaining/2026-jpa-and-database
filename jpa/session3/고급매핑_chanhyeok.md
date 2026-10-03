# Chapter 7. 고급 매핑

## 1. 상속 관계 매핑

관계형 데이터베이스에는 상속 개념이 없다. 대신 **슈퍼타입 서브타입 관계**라는 모델링 기법이 객체의 상속과 가장 비슷하다. ORM의 상속 관계 매핑은 객체의 상속 구조와 이 관계를 매핑하는 것이다.

논리 모델을 테이블로 구현하는 방법은 세 가지다.

| 변환 방법 | JPA 전략 |
| --- | --- |
| 각각의 테이블로 변환 | 조인 전략 (`JOINED`) |
| 통합 테이블로 변환 | 단일 테이블 전략 (`SINGLE_TABLE`) |
| 서브타입 테이블로 변환 | 구현 클래스마다 테이블 전략 (`TABLE_PER_CLASS`) |

### 조인 전략

엔티티 각각을 모두 테이블로 만들고, **자식 테이블이 부모 테이블의 기본 키를 받아 기본 키 + 외래 키로 사용**한다. 테이블에는 타입 개념이 없으므로 타입을 구분하는 컬럼(`DTYPE`)을 추가한다.

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)   // 상속 매핑은 부모 클래스에 선언
@DiscriminatorColumn(name = "DTYPE")              // 구분 컬럼. 기본값이 DTYPE
public abstract class Item {

    @Id @GeneratedValue
    @Column(name = "ITEM_ID")
    private Long id;

    private String name;
    private int price;
}

@Entity
@DiscriminatorValue("M")                          // 영화를 저장하면 DTYPE에 M이 들어간다
public class Movie extends Item {
    private String director;
    private String actor;
}

@Entity
@DiscriminatorValue("B")
@PrimaryKeyJoinColumn(name = "BOOK_ID")           // 자식 테이블의 PK 컬럼명 변경 (기본은 ITEM_ID)
public class Book extends Item {
    private String author;
    private String isbn;
}
```

| 장점 | 단점 |
| --- | --- |
| 테이블이 정규화된다 | 조회할 때 조인이 많아 성능이 떨어질 수 있다 |
| 외래 키 참조 무결성 제약조건을 활용할 수 있다 | 조회 쿼리가 복잡하다 |
| 저장 공간을 효율적으로 쓴다 | 등록할 때 INSERT가 두 번 실행된다 |

```sql
insert into Item (name, price, DTYPE, ITEM_ID) values (?, ?, 'M', ?)
insert into Movie (director, actor, ITEM_ID) values (?, ?, ?)
```

> JPA 표준 명세는 구분 컬럼을 쓰도록 하지만, 하이버네이트는 구분 컬럼 없이도 동작한다.
> 그래도 DB만 보고 자식 타입을 알 수 있도록 넣어 두는 편이 좋다.

### 단일 테이블 전략

테이블 하나만 쓰고 구분 컬럼(`DTYPE`)으로 어떤 자식 데이터인지 구분한다. 조인이 없어 일반적으로 가장 빠르다.

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "DTYPE")
public abstract class Item { ... }
```

Book을 저장하면 다른 엔티티가 매핑한 `ARTIST`, `DIRECTOR`, `ACTOR`는 null이 된다. 그래서 **자식 엔티티가 매핑한 컬럼은 모두 null을 허용해야 한다.**

| 장점 | 단점 |
| --- | --- |
| 조인이 없어 조회 성능이 빠르다 | 자식 엔티티가 매핑한 컬럼은 모두 null을 허용해야 한다 |
| 조회 쿼리가 단순하다 | 테이블이 커지면 오히려 조회가 느려질 수 있다 |

- 구분 컬럼이 반드시 필요하다. 책은 `@DiscriminatorColumn`을 꼭 설정하라고 한다.
- `@DiscriminatorValue`를 생략하면 **엔티티 이름**이 기본값이다.

> 실제로는 `@DiscriminatorColumn`을 생략해도 하이버네이트가 `DTYPE`을 자동으로 만든다.

### 구현 클래스마다 테이블 전략

자식 엔티티마다 테이블을 만들고 각 테이블에 부모 컬럼까지 모두 넣는다. 구분 컬럼은 쓰지 않는다.

- 장점: 서브 타입을 구분해서 처리할 때 효과적이고, `not null` 제약조건을 쓸 수 있다.
- 단점: 부모 타입으로 조회하면 자식 테이블을 전부 `UNION` 해야 해서 느리다.

**데이터베이스 설계자와 ORM 전문가 모두 추천하지 않는 전략이다.**

> 여러 테이블에 걸쳐 ID가 겹치면 안 되므로 `GenerationType.IDENTITY`를 쓸 수 없다.

### 어떤 전략을 쓸까

| | 조인 | 단일 테이블 | 구현 클래스마다 |
| --- | --- | --- | --- |
| 테이블 | 부모 1 + 자식 N | 1 | 자식 N |
| 구분 컬럼 | 선택 | 필수 | 사용 안 함 |
| 조회 | 조인 | 조인 없음 | 부모 타입 조회 시 `UNION` |
| 자식 컬럼 `not null` | 가능 | 불가 | 가능 |

기본은 **조인 전략**이다. 구조가 단순하고 확장될 일이 거의 없으면 **단일 테이블 전략**을 쓴다.

| 판단 기준 | 조인 | 단일 테이블 |
| --- | --- | --- |
| 자식 고유 컬럼 | 많고 서로 다름 | 1~2개로 단순 |
| 자식 컬럼에 `not null` 필요 | 가능 | 불가 |
| 특정 자식만 FK로 참조 (예: `ALBUM_REVIEW` → `ALBUM`) | 가능 | 불가 |
| 자식 타입 확장 가능성 | 높음 (테이블만 추가) | 낮음 |
| 부모 타입 목록 조회 | 드묾 (자식 수만큼 outer join) | 잦고 성능이 중요 |

예를 들어 결제수단(카드, 계좌이체)은 자식 컬럼이 필수값이고 종류가 늘어날 수 있어 조인 전략이 맞다. 알림(이메일, SMS)은 자식 고유 컬럼이 하나뿐이고 "내 알림 목록"처럼 부모 타입 조회가 대부분이라 단일 테이블 전략이 낫다.

## 2. @MappedSuperclass

부모 클래스는 테이블과 매핑하지 않고 자식에게 **매핑 정보만** 물려줄 때 쓴다. 추상 클래스와 비슷하다. `@Entity`는 실제 테이블과 매핑되지만 `@MappedSuperclass`는 매핑 정보 상속만을 위해 쓴다.

```java
@MappedSuperclass
public abstract class BaseEntity {

    @Id @GeneratedValue
    private Long id;

    private String name;
}

@Entity
@AttributeOverride(name = "id", column = @Column(name = "MEMBER_ID"))
public class Member extends BaseEntity {
    private String email;          // MEMBER 테이블: MEMBER_ID, name, email
}
```

물려받은 매핑을 재정의하려면 `@AttributeOverride(s)`, 연관관계를 재정의하려면 `@AssociationOverride(s)`를 쓴다.

### 특징

- 테이블과 매핑되지 않는다.
- **엔티티가 아니므로** `em.find()`나 JPQL로 조회할 수 없다.
- 직접 생성할 일이 없으므로 **추상 클래스**로 만드는 것을 권장한다.
- 주로 등록일, 수정일, 등록자, 수정자 같은 공통 속성을 모을 때 쓴다.

ORM에서 말하는 진짜 상속 매핑은 앞에서 본 슈퍼타입 서브타입 매핑이고, `@MappedSuperclass`는 공통 매핑 정보를 모아 줄 뿐이다.

> 엔티티는 엔티티이거나 `@MappedSuperclass`로 지정한 클래스만 상속받을 수 있다.

## 3. 복합 키와 식별 관계 매핑

### 식별 관계와 비식별 관계

| 관계 | 설명 |
| --- | --- |
| **식별 관계** | 부모의 기본 키를 받아 자식의 **기본 키 + 외래 키**로 사용 |
| **비식별 관계** | 부모의 기본 키를 받아 자식의 **외래 키로만** 사용 |

비식별 관계는 외래 키의 null 허용 여부로 다시 나뉜다.

- **필수적 비식별 관계**: 외래 키에 null을 허용하지 않는다. 연관관계를 반드시 맺어야 한다.
- **선택적 비식별 관계**: 외래 키에 null을 허용한다.

### 복합 키에는 식별자 클래스가 필요하다

JPA는 영속성 컨텍스트에 엔티티를 보관할 때 **식별자를 키로 쓰고**, `equals()`와 `hashCode()`로 식별자의 동등성을 비교한다. 식별자 필드가 둘 이상이면 `@Id`만 두 개 붙여서는 매핑 예외가 나고, 별도의 **식별자 클래스**를 만들어야 한다.

JPA는 이를 위해 `@IdClass`와 `@EmbeddedId`를 제공한다.

### @IdClass

관계형 데이터베이스에 가까운 방법이다.

```java
@Entity
@IdClass(ParentId.class)
public class Parent {

    @Id @Column(name = "PARENT_ID1")
    private String id1;        // ParentId.id1과 연결

    @Id @Column(name = "PARENT_ID2")
    private String id2;        // ParentId.id2와 연결
}

public class ParentId implements Serializable {
    private String id1;
    private String id2;
    // 기본 생성자, equals, hashCode
}
```

**식별자 클래스 조건**

- 식별자 클래스의 속성명과 엔티티의 식별자 속성명이 같아야 한다
- `Serializable`을 구현해야 한다
- `equals()`, `hashCode()`를 구현해야 한다
- 기본 생성자가 있어야 하고, `public` 클래스여야 한다

> 식별자 클래스에는 setter를 두지 않는다. 영속 상태 엔티티의 PK 값을 바꾸면 영속성 컨텍스트가 엔티티를 찾지 못한다.

```java
// 저장: 식별자 클래스가 보이지 않는다
parent.setId1("myId1");
parent.setId2("myId2");
em.persist(parent);

// 조회
Parent found = em.find(Parent.class, new ParentId("myId1", "myId2"));
```

저장할 때 `ParentId`를 만들지 않아도 되는 이유는 `em.persist()`가 영속성 컨텍스트에 등록하기 직전에 **`Parent.id1`, `Parent.id2` 값으로 `ParentId`를 내부에서 생성해 키로 쓰기** 때문이다.

자식이 부모의 복합 키를 참조하면 외래 키도 복합 키가 되므로 `@JoinColumns`로 컬럼마다 `@JoinColumn`을 매핑한다.

### @EmbeddedId

좀 더 객체지향적인 방법이다. 식별자 클래스에 기본 키를 직접 매핑한다.

```java
@Entity
public class Parent {
    @EmbeddedId
    private ParentId id;
}

@Embeddable
public class ParentId implements Serializable {
    @Column(name = "PARENT_ID1")
    private String id1;
    @Column(name = "PARENT_ID2")
    private String id2;
    // 기본 생성자, equals, hashCode
}
```

조건은 `@IdClass`와 같고 `@Embeddable`만 추가된다. 저장할 때는 식별자 클래스를 직접 만들어 `parent.setId(new ParentId(...))`로 넣는다.

### 왜 equals()와 hashCode()가 필수인가

`Object`의 기본 `equals()`는 참조를 비교하는 **동일성 비교**(`==`)다. 값이 같은 `ParentId` 두 개를 만들어도 `equals()`를 오버라이딩하지 않으면 `false`다.

영속성 컨텍스트는 식별자를 키로 엔티티를 관리하므로, 식별자의 동등성이 지켜지지 않으면 **예상과 다른 엔티티가 조회되거나 엔티티를 찾을 수 없는** 심각한 문제가 생긴다. 보통 모든 필드를 사용해 구현한다.

### @IdClass vs @EmbeddedId

| | `@IdClass` | `@EmbeddedId` |
| --- | --- | --- |
| 성격 | 데이터베이스 친화적 | 객체지향적 |
| JPQL | `select p.id1, p.id2 from Parent p` | `select p.id.id1, p.id.id2 from Parent p` |

취향에 맞는 것을 일관성 있게 쓰면 된다. `@EmbeddedId`는 중복이 없지만 JPQL이 조금 길어진다.

> 복합 키에는 `@GeneratedValue`를 쓸 수 없다. 복합 키를 구성하는 컬럼 중 하나에도 쓸 수 없다.

### 식별 관계 매핑

부모, 자식, 손자로 기본 키가 계속 전파된다. 자식은 부모의 기본 키를 포함한 복합 키를 써야 하므로 `@IdClass`나 `@EmbeddedId`가 필요하다.

**@IdClass**: `@Id`와 `@ManyToOne`을 같이 써서 기본 키와 외래 키를 한 번에 매핑한다.

```java
@Entity
@IdClass(ChildId.class)
public class Child {

    @Id
    @ManyToOne
    @JoinColumn(name = "PARENT_ID")
    public Parent parent;

    @Id @Column(name = "CHILD_ID")
    private String childId;
}

public class ChildId implements Serializable {
    private String parent;     // Child.parent 매핑
    private String childId;    // Child.childId 매핑
}
```

**@EmbeddedId**: `@Id` 대신 `@MapsId`를 쓴다.

```java
@Entity
public class Child {

    @EmbeddedId
    private ChildId id;

    @MapsId("parentId")        // ChildId.parentId 매핑
    @ManyToOne
    @JoinColumn(name = "PARENT_ID")
    public Parent parent;
}

@Embeddable
public class ChildId implements Serializable {
    private String parentId;

    @Column(name = "CHILD_ID")
    private String id;
}
```

`@MapsId`는 **외래 키와 매핑한 연관관계를 기본 키에도 매핑하겠다**는 뜻이다. 속성 값에는 식별자 클래스의 필드명을 지정한다.

손자는 `@JoinColumns`로 `PARENT_ID`, `CHILD_ID`를 함께 매핑하고, 식별자 클래스가 `ChildId`를 필드로 가진다. 패턴은 자식과 같다.

### 비식별 관계로 바꾸면

각 테이블이 `Long` 대리 키를 가지고 부모 키는 외래 키로만 받는다. `@ManyToOne` + `@JoinColumn`이면 끝나고, **복합 키 클래스가 전부 사라진다.**

### 일대일 식별 관계

자식의 기본 키로 **부모의 기본 키 값만** 쓴다. 부모 키가 복합 키가 아니면 자식도 복합 키가 필요 없다.

```java
@Entity
public class BoardDetail {

    @Id
    private Long boardId;

    @MapsId                    // 속성 값을 비우면 @Id 필드(boardId)와 매핑
    @OneToOne
    @JoinColumn(name = "BOARD_ID")
    private Board board;

    private String content;
}
```

저장할 때 `boardId`를 직접 넣지 않고 `boardDetail.setBoard(board)`만 하면 `board`의 PK가 그대로 들어간다.

### 식별 관계와 비식별 관계, 무엇을 고를까

**비식별 관계를 선호하는 이유**

- 식별 관계는 자식, 손자로 갈수록 기본 키 컬럼이 늘어난다. 조인 SQL이 복잡해지고 기본 키 인덱스가 불필요하게 커진다.
- 식별 관계는 비즈니스 의미가 있는 **자연 키**를 조합하는 경우가 많다. 요구사항이 바뀌면 자식, 손자까지 전파된 키를 바꾸기 어렵다.
- 테이블 구조가 유연하지 않다.
- JPA에서 복합 키는 별도 클래스가 필요하지만, 대리 키는 `@GeneratedValue`로 편하게 만든다.

**식별 관계의 장점**

기본 키 인덱스를 활용하기 좋다. `CHILD`의 기본 키가 `PARENT_ID + CHILD_ID`라면 `WHERE PARENT_ID = 'A'`나 `WHERE PARENT_ID = 'A' AND CHILD_ID = 'B'` 모두 별도 인덱스 없이 처리된다. 상위 키를 하위 테이블이 가지고 있어 조인 없이 검색할 수도 있다.

**정리**

될 수 있으면 **비식별 관계 + `Long` 대리 키**를 쓰고, 꼭 필요한 곳에만 식별 관계를 쓴다.

- `Integer`는 약 20억이라 부족할 수 있지만, `Long`은 약 920경이라 안전하다.
- **필수적 비식별 관계**가 좋다. 선택적은 null을 허용해 외부 조인이 필요하지만, 필수적은 **내부 조인만** 써도 된다.

## 4. 조인 테이블

| 방법 | 설명 |
| --- | --- |
| **조인 컬럼** | 외래 키 컬럼으로 관계를 관리한다. `@JoinColumn` |
| **조인 테이블** | 별도의 연결 테이블로 관계를 관리한다. `@JoinTable` |

- **조인 컬럼의 문제**: 회원이 사물함을 선택적으로 쓰면 외래 키에 null이 들어가 **외부 조인**을 써야 한다. 실수로 내부 조인을 쓰면 사물함 없는 회원이 빠진다.
- **조인 테이블의 문제**: null 문제는 없지만 **테이블이 하나 늘고 조인도 한 번 더** 해야 한다.

기본은 조인 컬럼을 쓰고, 필요하다고 판단될 때 조인 테이블을 쓴다. 조인 테이블은 주로 다대다를 일대다, 다대일로 풀 때 쓰지만 다른 관계에서도 쓸 수 있다. 연결 테이블, 링크 테이블이라고도 부른다.

```java
@OneToMany
@JoinTable(
    name = "PARENT_CHILD",                                  // 연결 테이블 이름
    joinColumns = @JoinColumn(name = "PARENT_ID"),          // 현재 엔티티를 참조하는 외래 키
    inverseJoinColumns = @JoinColumn(name = "CHILD_ID")     // 반대 방향 엔티티를 참조하는 외래 키
)
private List<Child> children = new ArrayList<>();
```

### 관계별 연결 테이블 구조

| 관계 | 연결 테이블의 제약조건 | 매핑 |
| --- | --- | --- |
| 일대일 | 양쪽 외래 키 컬럼에 각각 유니크 | `@OneToOne` + `@JoinTable` |
| 일대다 | 다(N) 쪽 외래 키(`CHILD_ID`)에 유니크 | `@OneToMany` + `@JoinTable` |
| 다대일 | 일대다와 방향만 반대 | `@ManyToOne` + `@JoinTable` |
| 다대다 | 두 외래 키를 합친 복합 유니크 | `@ManyToMany` + `@JoinTable` |

> 연결 테이블에 컬럼을 추가하면 `@JoinTable`을 쓸 수 없다. 연결 테이블을 **새로운 엔티티로 만들어** 매핑해야 한다.

## 5. 엔티티 하나에 여러 테이블 매핑

`@SecondaryTable`로 엔티티 하나에 여러 테이블을 매핑할 수 있다.

```java
@Entity
@Table(name = "BOARD")
@SecondaryTable(name = "BOARD_DETAIL",
    pkJoinColumns = @PrimaryKeyJoinColumn(name = "BOARD_DETAIL_ID"))
public class Board {

    @Id @GeneratedValue
    @Column(name = "BOARD_ID")
    private Long id;

    private String title;                  // 지정하지 않으면 기본 테이블(BOARD)

    @Column(table = "BOARD_DETAIL")
    private String content;
}
```

더 많은 테이블은 `@SecondaryTables`로 매핑한다.

**권장하지 않는다.** 항상 두 테이블을 함께 조회하므로 최적화하기 어렵다. 테이블당 엔티티를 만들어 **일대일로 매핑**하면 필요한 부분만 조회할 수 있다.

## 6. 정리

- 상속 매핑은 기본이 조인 전략, 단순하면 단일 테이블 전략. 구현 클래스마다 테이블 전략은 쓰지 않는다.
- `@MappedSuperclass`는 매핑 정보만 물려준다. 엔티티가 아니므로 조회할 수 없다.
- 복합 키는 `@IdClass`나 `@EmbeddedId`로 매핑하고, 식별자 클래스에 `equals()`와 `hashCode()`를 반드시 구현한다.
- 식별 관계보다 **필수적 비식별 관계 + `Long` 대리 키**를 권장한다.
- 연관관계는 조인 컬럼이 기본이고, 조인 테이블은 필요할 때만 쓴다.
- 엔티티 하나에 여러 테이블을 매핑하기보다 테이블당 엔티티를 만들어 일대일로 매핑한다.
