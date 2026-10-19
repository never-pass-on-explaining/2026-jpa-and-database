# Chapter 8. 프록시와 연관관계 관리

## 1. 프록시

회원을 조회할 때 연관된 팀이 항상 필요한 것은 아니다. 회원 이름만 출력하는데 팀까지 함께 조회하면 낭비다. JPA는 연관된 엔티티를 **실제 사용하는 시점에 조회**하는 지연 로딩을 지원하고, 이를 위해 DB 조회를 미루는 가짜 객체인 **프록시**를 쓴다.

> JPA 표준 명세는 지연 로딩 구현 방법을 구현체에 맡긴다. 하이버네이트는 프록시와 바이트코드 수정 두 방법을 지원하며, 이 장은 별도 설정이 필요 없는 프록시를 다룬다.

```java
Member member = em.find(Member.class, "member1");          // 즉시 DB 조회
Member member = em.getReference(Member.class, "member1");  // DB 조회 X, 프록시 반환
```

### 프록시의 동작

프록시 클래스는 **실제 클래스를 상속**해서 만들어지므로 겉모양이 같다. 프록시는 실제 객체의 참조(`target`)를 보관하고, 메서드가 호출되면 실제 객체에 위임한다.

`member.getName()`처럼 실제 사용될 때 실제 엔티티를 만드는 것을 **프록시 초기화**라 한다.

1. 프록시의 `getName()` 호출
2. `target`이 비어 있으면 영속성 컨텍스트에 초기화 요청
3. 영속성 컨텍스트가 DB를 조회해 실제 엔티티 생성
4. 프록시가 실제 엔티티 참조를 `target`에 보관
5. `target.getName()`을 호출해 결과 반환

### 프록시의 특징

- 처음 사용할 때 **한 번만** 초기화된다.
- 초기화되어도 프록시가 실제 엔티티로 바뀌지 않는다. 프록시를 통해 실제 엔티티에 접근할 뿐이다.
- 원본 엔티티를 상속한 객체이므로 **타입 체크에 주의**한다.
- 영속성 컨텍스트에 엔티티가 이미 있으면 `getReference()`도 실제 엔티티를 반환한다.
- 초기화는 영속성 컨텍스트의 도움이 필요하다. **준영속 상태에서 초기화하면** `LazyInitializationException`이 발생한다.

```java
Member member = em.getReference(Member.class, "id1");
transaction.commit();
em.close();          // 영속성 컨텍스트 종료, member는 준영속

member.getName();    // LazyInitializationException
```

### 프록시와 식별자

프록시는 식별자 값을 보관하므로 `getId()`를 호출해도 초기화하지 않는다. 단 책은 엔티티 접근 방식이 필드(`@Access(AccessType.FIELD)`)면 JPA가 `getId()`가 무슨 일을 하는지 몰라 초기화한다고 설명한다.

그래서 프록시는 **연관관계를 설정할 때** 유용하다. 식별자만 필요하므로 SQL 없이 외래 키를 설정할 수 있다.

```java
Member member = em.find(Member.class, "member1");
Team team = em.getReference(Team.class, "team1");   // SQL 실행 X
member.setTeam(team);
```

초기화 여부는 `PersistenceUnitUtil.isLoaded(entity)`로 확인하고, 강제 초기화는 하이버네이트의 `Hibernate.initialize(entity)`를 쓴다. JPA 표준에는 강제 초기화 메서드가 없다.

## 2. 즉시 로딩과 지연 로딩

| | 즉시 로딩 | 지연 로딩 |
| --- | --- | --- |
| 설정 | `@ManyToOne(fetch = FetchType.EAGER)` | `@ManyToOne(fetch = FetchType.LAZY)` |
| 동작 | 엔티티를 조회할 때 연관 엔티티도 함께 조회 | 연관 엔티티를 실제 사용할 때 조회 |
| 연관 엔티티 | 실제 엔티티 | 프록시 |

### 즉시 로딩

대부분의 JPA 구현체는 즉시 로딩을 최적화하려고 가능하면 **조인 쿼리 한 번**으로 조회한다.

```sql
SELECT M.*, T.*
FROM MEMBER M LEFT OUTER JOIN TEAM T ON M.TEAM_ID = T.TEAM_ID
WHERE M.MEMBER_ID = 'member1'
```

외래 키가 NULL을 허용하면 팀이 없는 회원도 조회되어야 하므로 **외부 조인**을 쓴다. `@JoinColumn(nullable = false)`나 `@ManyToOne(optional = false)`로 필수 관계임을 알려주면 **내부 조인**을 쓴다.

### 지연 로딩

```java
Member member = em.find(Member.class, "member1");   // SELECT * FROM MEMBER ...
Team team = member.getTeam();                       // 프록시
team.getName();                                     // 이때 SELECT * FROM TEAM ...
```

컬렉션은 프록시 대신 하이버네이트의 **컬렉션 래퍼**(`PersistentBag`)가 지연 로딩을 처리한다. `member.getOrders()`만으로는 초기화되지 않고, `getOrders().get(0)`처럼 실제 데이터를 조회할 때 초기화된다.

## 3. 왜 지연 로딩이 기본인가

### JPA 기본 페치 전략

| 연관관계 | 기본값 |
| --- | --- |
| `@ManyToOne`, `@OneToOne` | 즉시 로딩 |
| `@OneToMany`, `@ManyToMany` | 지연 로딩 |

연관 엔티티가 하나면 즉시 로딩, 컬렉션이면 지연 로딩이다. 컬렉션에는 데이터가 수만 개 들어 있을 수도 있다. 회원 한 명을 조회했는데 주문 수만 건까지 함께 조회되면 위험하다.

### 모든 연관관계에 지연 로딩

책의 결론은 **모든 연관관계에 지연 로딩을 사용하는 것**이다. 개발이 어느 정도 끝난 뒤 실제 사용 상황을 보고 꼭 필요한 곳만 즉시 로딩으로 최적화한다. SQL을 직접 쓰면 이런 유연한 최적화가 어렵다.

실전 예제도 기본값이 즉시 로딩인 `@ManyToOne`, `@OneToOne`에 `fetch = FetchType.LAZY`를 붙여 전부 지연 로딩으로 바꾼다.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "MEMBER_ID")
private Member member;

@OneToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "DELIVERY_ID")
private Delivery delivery;
```

### 컬렉션 즉시 로딩의 위험

- **컬렉션을 2개 이상 즉시 로딩하지 않는다.** 일대다 조인은 결과가 다 쪽 수만큼 늘어난다. A를 N, M과 조인하면 N×M 행이 나와 성능이 떨어진다.
- **컬렉션 즉시 로딩은 항상 외부 조인이다.** 회원이 없는 팀이 결과에서 사라지면 안 되고, DB 제약조건으로 이를 막을 수 없다.

| 연관관계 | `optional = false` | `optional = true` |
| --- | --- | --- |
| `@ManyToOne`, `@OneToOne` | 내부 조인 | 외부 조인 |
| `@OneToMany`, `@ManyToMany` | 외부 조인 | 외부 조인 |

> 하이버네이트에서는 `List` 컬렉션 2개를 함께 즉시 로딩하면 성능 저하 전에 `MultipleBagFetchException`이 발생한다.

## 4. 영속성 전이: CASCADE

JPA에서 엔티티를 저장할 때 연관된 엔티티는 모두 영속 상태여야 한다. 영속성 전이를 쓰면 **부모만 영속화해도 자식까지 함께** 영속화된다.

```java
@OneToMany(mappedBy = "parent", cascade = CascadeType.PERSIST)
private List<Child> children = new ArrayList<>();
```

```java
Parent parent = new Parent();
child1.setParent(parent);              // 연관관계는 직접 설정
parent.getChildren().add(child1);

em.persist(parent);                    // 자식도 함께 저장
```

- 영속성 전이는 **연관관계 매핑과 아무 관련이 없다.** 함께 영속화하는 편의 기능일 뿐이라 양방향 연관관계는 직접 설정해야 한다.
- `CascadeType.REMOVE`면 부모만 삭제해도 자식까지 삭제된다. 외래 키 제약 때문에 자식을 먼저 삭제한다. 설정하지 않고 부모만 삭제하면 외래 키 무결성 예외가 발생한다.
- 종류는 `ALL`, `PERSIST`, `MERGE`, `REMOVE`, `REFRESH`, `DETACH`이고 여러 개를 함께 쓸 수 있다.
- 책은 `PERSIST`, `REMOVE`가 호출 즉시가 아니라 **플러시 시점에 전이된다**고 설명한다.

## 5. 고아 객체

부모와 **연관관계가 끊어진 자식을 자동으로 삭제**하는 기능이다.

```java
@OneToMany(mappedBy = "parent", orphanRemoval = true)
private List<Child> children = new ArrayList<>();
```

```java
Parent parent1 = em.find(Parent.class, id);
parent1.getChildren().remove(0);   // 플러시 시점에 DELETE FROM CHILD WHERE ID=?
```

- 참조가 끊긴 자식을 다른 곳에서 참조하지 않는 고아로 보고 삭제한다. 그래서 **특정 엔티티가 개인 소유하는 엔티티에만** 써야 한다.
- 이런 이유로 `@OneToOne`, `@OneToMany`에만 쓸 수 있다.
- 부모를 제거하면 자식도 고아가 되어 함께 제거된다. `CascadeType.REMOVE`와 같다.

### 영속성 전이 + 고아 객체

`CascadeType.ALL + orphanRemoval = true`를 함께 쓰면 **부모를 통해 자식의 생명주기를 관리**할 수 있다.

```java
parent.addChild(child1);                     // 저장: 부모에 등록만 하면 된다
parent.getChildren().remove(removeObject);   // 삭제: 부모에서 제거하면 된다
```

DDD의 **Aggregate Root** 개념을 구현할 때 유용하다.

## 6. 정리

- 지연 로딩은 **프록시**로 구현한다. 프록시는 실제 클래스를 상속하고, 처음 사용할 때 한 번 초기화된다.
- 준영속 상태에서 프록시를 초기화하면 `LazyInitializationException`이 발생한다.
- `@XToOne`의 기본값은 즉시 로딩, `@XToMany`는 지연 로딩이다. **모든 연관관계를 지연 로딩으로 두고** 필요한 곳만 최적화한다.
- 영속성 전이는 연관 엔티티를 함께 저장하거나 삭제하는 편의 기능일 뿐, 연관관계 매핑과 무관하다.
- 고아 객체 제거는 개인 소유 엔티티에만 쓴다. `CASCADE.ALL + orphanRemoval`이면 부모가 자식의 생명주기를 관리한다.
