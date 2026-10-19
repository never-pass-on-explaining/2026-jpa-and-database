# 8장. 프록시와 연관관계 관리

엔티티를 조회할 때 연관된 엔티티까지 항상 함께 조회할 필요는 없다. 이 장에서는 연관 엔티티의 조회 시점을 조절하는 **프록시**와 **즉시/지연 로딩**, 그리고 연관 엔티티의 저장·삭제를 함께 다루는 **영속성 전이**와 **고아 객체**를 정리한다.

---

## 1. 프록시

회원을 조회할 때 팀 정보까지 매번 DB에서 가져오는 것은 낭비다. JPA는 실제 사용 시점까지 DB 조회를 미루기 위해 **프록시**라는 가짜 객체를 사용한다. 이 절에서는 프록시를 얻는 방법, 프록시가 동작하는 방식, 그리고 프록시 사용 시 주의해야 할 예외를 순서대로 살펴본다.

### em.find() vs em.getReference()

프록시를 이해하려면 먼저 엔티티를 조회하는 두 메서드의 차이를 알아야 한다. 여기서는 `em.find()`와 `em.getReference()`가 각각 언제 DB에 접근하고 무엇을 반환하는지 비교한다.

| 메서드 | 동작 |
|---|---|
| `em.find()` | 호출 즉시 DB를 조회하고 실제 엔티티를 반환한다 |
| `em.getReference()` | DB 조회를 미루고 **프록시 객체**를 반환한다 |

```java
Member member = em.getReference(Member.class, 1L); // SELECT 안 나감
System.out.println(member.getClass());             // Member$HibernateProxy$...
member.getName();                                  // 이 시점에 SELECT
```

`getReference()`로 받은 객체는 실제 데이터를 사용하는 순간에야 SELECT가 실행된다.

### 프록시 초기화와 특징

프록시가 DB 조회를 미룬다면, 실제 데이터는 언제 어떻게 채워지는지가 궁금해진다. 여기서는 프록시가 실제 엔티티를 가져오는 과정인 **초기화**의 흐름을 보고, 이어서 프록시를 다룰 때 알아둬야 할 특징을 정리한다.

```
member.getName()
  → 프록시의 target이 null
  → 영속성 컨텍스트에 초기화 요청
  → DB 조회 후 실제 엔티티 생성
  → target이 실제 엔티티를 참조, 위임 호출
```

프록시는 다음과 같은 특징을 가진다.

- 실제 클래스를 **상속**해서 만들어지므로, 사용하는 입장에서는 실제 엔티티와 구분 없이 사용할 수 있다.
- 처음 사용할 때 **한 번만** 초기화된다.
- 초기화되어도 프록시가 실제 엔티티로 바뀌지는 않는다. target을 통해 실제 엔티티에 접근할 뿐이다.

  > 💡 프록시는 **빈 상자**, 실제 엔티티는 **상자 안에 들어갈 물건**이라고 생각하면 쉽다.
  > - 처음엔 빈 상자(프록시)만 받는다.
  > - 데이터가 필요해지면 DB에서 물건(실제 엔티티)을 꺼내 상자 **안에 넣는다**. 이게 초기화다.
  > - 상자가 물건으로 변하는 게 아니다. 내 손에는 여전히 **상자**가 들려 있고, 상자가 안에 든 물건을 대신 꺼내줄 뿐이다.
  >
  > 그래서 초기화 후에도 `member.getClass()`를 찍어보면 `Member`가 아니라 `Member$HibernateProxy$...`(상자)가 나온다.

- 따라서 타입 비교는 `==` 대신 **`instanceof`**를 사용해야 한다.
- 영속성 컨텍스트에 이미 엔티티가 있으면 `getReference()`도 **실제 엔티티**를 반환한다. 반대로 프록시가 먼저 조회됐다면 `find()`도 프록시를 반환한다. 영속성 컨텍스트의 동일성 보장 때문이다.

  > 💡 JPA에는 **"같은 id면 항상 같은 객체를 준다"**는 규칙이 있다. 즉 `m1 == m2`가 항상 `true`여야 한다.
  >
  > **경우 1: 실제 엔티티를 먼저 받은 경우**
  > ```java
  > Member m1 = em.find(Member.class, 1L);         // 실제 엔티티
  > Member m2 = em.getReference(Member.class, 1L); // 여기서 프록시를 새로 주면 m1 != m2
  > ```
  > 규칙이 깨지니까, `getReference()`도 이미 있는 **실제 엔티티**를 그대로 준다.
  >
  > **경우 2: 프록시를 먼저 받은 경우**
  > ```java
  > Member m1 = em.getReference(Member.class, 1L); // 프록시
  > Member m2 = em.find(Member.class, 1L);         // 여기서 실제 엔티티를 주면 m1 != m2
  > ```
  > 마찬가지로, `find()`도 이미 있는 **프록시**를 그대로 준다.
  >
  > 한 줄로 정리하면, **먼저 받은 쪽을 계속 돌려준다.**

- 식별자(`getId()`) 조회는 초기화를 일으키지 않는다. (`@Access(AccessType.PROPERTY)`일 때)

초기화 여부는 아래처럼 확인하거나 강제할 수 있다.

```java
// 초기화 여부 확인
emf.getPersistenceUnitUtil().isLoaded(member);
// 강제 초기화 (Hibernate)
Hibernate.initialize(member);
```

### 준영속 상태와 LazyInitializationException

앞에서 본 것처럼 프록시 초기화는 영속성 컨텍스트에 요청해서 이뤄진다. 그렇다면 영속성 컨텍스트가 없는 상황에서는 어떻게 될까? 여기서는 준영속 상태에서 프록시를 초기화할 때 발생하는 예외를 살펴본다.

```java
Member member = em.getReference(Member.class, 1L);
em.close(); // 또는 em.detach(member)

member.getName(); // LazyInitializationException
```

영속성 컨텍스트의 도움을 받을 수 없으므로 초기화에 실패하고 `LazyInitializationException`이 발생한다. 실무에서는 트랜잭션 밖(컨트롤러, 뷰)에서 지연 로딩 필드에 접근할 때 자주 만나는 예외다.

---

## 2. 즉시 로딩과 지연 로딩

프록시를 직접 다룰 일은 많지 않다. 실제로 프록시가 쓰이는 곳은 연관 엔티티를 조회하는 시점을 결정하는 **페치 전략**이다. 이 절에서는 즉시 로딩과 지연 로딩의 차이, JPA의 기본 페치 전략, 그리고 실무에서 권장하는 방식을 다룬다.

### FetchType.EAGER vs LAZY

연관 엔티티를 언제 조회할지는 `fetch` 속성으로 정한다. 여기서는 두 방식이 조회 시점, 반환 객체, 실행되는 SQL에서 어떻게 다른지 비교한다.

```java
@Entity
public class Member {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}
```

| | EAGER (즉시 로딩) | LAZY (지연 로딩) |
|---|---|---|
| 조회 시점 | 엔티티를 조회할 때 연관 엔티티도 함께 조회한다 | 연관 엔티티를 실제 사용할 때 조회한다 |
| 연관 엔티티 | 실제 엔티티 | 프록시 |
| SQL | 조인 한 번 | 필요할 때 별도 SELECT |

- 즉시 로딩은 외래키가 nullable이면 **외부 조인**을 사용한다. `nullable = false` 또는 `optional = false`로 설정하면 **내부 조인**을 사용한다.
- 컬렉션은 **컬렉션 래퍼**(`PersistentBag`)로 감싸진다. `member.getOrders()` 호출만으로는 초기화되지 않고, `getOrders().get(0)`처럼 실제 데이터에 접근할 때 초기화된다.

  > 💡 컬렉션 래퍼도 앞의 **빈 상자**와 같은 원리다. 다만 엔티티 하나가 아니라 **리스트를 담는 상자**다.
  > - 하이버네이트는 `orders` 리스트를 빈 상자(`PersistentBag`)로 바꿔 넣어둔다.
  > - `getOrders()`는 **상자를 건네주기만** 한다. 상자 안을 열어보지 않았으니 DB에 갈 필요가 없다.
  > - `get(0)`, `size()`, for문처럼 **상자 안을 들여다보는 순간** DB에서 주문 목록을 가져와 채운다.

### 기본 페치 전략

`fetch`를 따로 지정하지 않으면 JPA가 정한 기본값이 적용된다. 여기서는 연관관계 종류별 기본값과, 기본값을 그대로 쓸 때 주의할 점을 정리한다.

| 어노테이션 | 기본값 |
|---|---|
| `@ManyToOne`, `@OneToOne` | EAGER |
| `@OneToMany`, `@ManyToMany` | LAZY |

연관 엔티티가 하나면 즉시 로딩, 컬렉션이면 지연 로딩이 기본이다. 컬렉션에 EAGER를 적용하면 다음 문제가 생긴다.

- 컬렉션을 둘 이상 즉시 로딩하면 조인 결과가 곱해져 데이터가 폭증한다.
- 컬렉션 즉시 로딩은 항상 외부 조인을 사용한다.

### 실무 권장: 모두 LAZY + fetch join

기본 페치 전략을 그대로 쓰면 `@XToOne`은 즉시 로딩이 된다. 여기서는 즉시 로딩이 실무에서 왜 문제가 되는지 보고, 그 대안으로 지연 로딩과 fetch join을 함께 쓰는 방법을 정리한다.

즉시 로딩은 예상하지 못한 SQL을 만들고, JPQL에서는 **N+1 문제**를 일으킨다.

```java
// JPQL은 SQL로 그대로 번역된다
// → Member를 조회한 뒤, EAGER라서 Team을 회원 수만큼 추가 조회
List<Member> members = em.createQuery("select m from Member m", Member.class)
                         .getResultList();
```

따라서 `@XToOne`은 **명시적으로 LAZY**로 설정하고, 연관 엔티티가 함께 필요할 때만 **fetch join**으로 한 번에 조회한다.

```java
em.createQuery("select m from Member m join fetch m.team", Member.class)
  .getResultList();
```

---

## 3. 영속성 전이: CASCADE

지금까지는 연관 엔티티를 **조회**하는 시점을 다뤘다. 이번에는 연관 엔티티를 **저장·삭제**하는 방법을 다룬다. 부모를 저장할 때 자식도 하나하나 `persist()`하는 것은 번거롭다. 이 절에서는 이를 해결하는 영속성 전이의 동작 방식과 종류를 살펴본다.

### CASCADE 동작과 주의점

영속성 전이를 설정하면 부모 엔티티에 대한 작업이 자식 엔티티에도 전달된다. 여기서는 저장을 예로 CASCADE가 어떻게 동작하는지 보고, 사용할 때 오해하기 쉬운 부분을 정리한다.

```java
@Entity
public class Parent {
    @OneToMany(mappedBy = "parent", cascade = CascadeType.PERSIST)
    private List<Child> children = new ArrayList<>();

    public void addChild(Child child) {
        children.add(child);
        child.setParent(this);
    }
}
```

```java
Parent parent = new Parent();
parent.addChild(new Child());
parent.addChild(new Child());

em.persist(parent); // child까지 함께 persist
```

- 영속성 전이는 연관관계 매핑과 **무관**하다. 영속화를 편하게 해주는 기능일 뿐이므로 연관관계 편의 메서드는 여전히 필요하다.
- 전이는 바로 일어나지 않고 **flush 시점**에 발생한다.
- 자식의 소유자가 **하나**일 때만 사용한다. 다른 엔티티도 자식을 참조한다면 사용하지 않는다.

### CascadeType 종류

CASCADE는 어떤 작업을 전이할지 골라서 지정할 수 있다. 여기서는 지정 가능한 옵션을 정리한다.

| 타입 | 설명 |
|---|---|
| `ALL` | 모두 적용 |
| `PERSIST` | 영속 |
| `REMOVE` | 삭제 |
| `MERGE` | 병합 |
| `REFRESH` | 새로고침 |
| `DETACH` | 준영속 |

실무에서는 주로 `ALL`과 `PERSIST`를 사용한다.

---

## 4. 고아 객체

CASCADE의 `REMOVE`는 부모를 삭제할 때 자식을 함께 삭제한다. 그런데 부모는 그대로 두고 자식과의 연관관계만 끊는 경우도 있다. 이 절에서는 연관관계가 끊어진 자식을 자동으로 삭제하는 고아 객체 제거 기능을 다루고, CASCADE와 함께 썼을 때의 의미를 정리한다.

### orphanRemoval

부모와 연관관계가 끊어진 자식 엔티티를 **고아 객체**라고 한다. 여기서는 `orphanRemoval` 옵션으로 고아 객체를 자동 삭제하는 방법과 사용 조건을 살펴본다.

```java
@OneToMany(mappedBy = "parent", orphanRemoval = true)
private List<Child> children = new ArrayList<>();
```

```java
Parent parent = em.find(Parent.class, id);
parent.getChildren().remove(0); // flush 시 DELETE FROM CHILD ...
```

- 부모를 제거하면 자식도 함께 제거된다. `CascadeType.REMOVE`처럼 동작한다.
- 참조하는 곳이 **하나**일 때만 사용한다. 특정 엔티티가 자식을 개인 소유하는 경우다.
- `@OneToOne`, `@OneToMany`에만 사용할 수 있다.

### CASCADE + 고아 객체 = 생명주기 관리

CASCADE는 저장을, 고아 객체는 삭제를 부모에게 맡긴다. 여기서는 두 옵션을 함께 사용했을 때 부모가 자식의 생명주기 전체를 관리하게 되는 원리를 정리한다.

```java
@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Child> children = new ArrayList<>();
```

- 원래 엔티티는 `em.persist()`로 영속화하고 `em.remove()`로 제거하며, 스스로 생명주기를 관리한다.
- 두 옵션을 함께 쓰면 **부모 엔티티를 통해 자식의 생명주기를 관리**할 수 있다.
  - 자식 저장 → 부모에 등록만 하면 된다. (CASCADE)
  - 자식 삭제 → 부모에서 제거만 하면 된다. (orphanRemoval)
- DDD의 **Aggregate Root** 개념을 구현할 때 유용하다. 자식용 Repository를 따로 만들 필요가 없다.

---

## 5. 정리

이 장에서 다룬 내용을 주제별로 한 줄씩 정리한다.

| 주제 | 핵심 |
|---|---|
| 프록시 | 실제 사용 시점까지 DB 조회를 미루는 가짜 객체다. 준영속 상태에서 초기화하면 예외가 발생한다. |
| 즉시/지연 로딩 | 실무에서는 모두 LAZY로 설정하고, 필요하면 fetch join으로 조회한다. |
| CASCADE | 연관관계와 무관한 영속화 편의 기능이다. 자식의 소유자가 하나일 때만 사용한다. |
| 고아 객체 | 연관관계가 끊긴 자식을 자동 삭제한다. CASCADE와 함께 쓰면 부모가 자식의 생명주기를 관리한다. |
