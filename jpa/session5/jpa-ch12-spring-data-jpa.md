# 스프링 데이터 JPA

---

## 1. 스프링 데이터 JPA 설정

스프링 데이터 JPA는 **리포지토리 인터페이스만 선언하면 애플리케이션 로딩 시점에 구현 클래스를 동적으로 생성**해서 빈으로 등록한다. 이를 위해 스캔할 패키지를 지정해야 한다.

```xml
<!-- XML 설정 -->
<jpa:repositories base-package="jpabook.jpashop.repository" />
```

```java
// JavaConfig 설정
@Configuration
@EnableJpaRepositories(basePackages = "jpabook.jpashop.repository")
public class AppConfig {}
```

> **현재 버전**: Spring Boot에서는 자동 설정되므로 `@EnableJpaRepositories`를 직접 선언할 필요가 없다. `@SpringBootApplication` 패키지 하위가 기본 스캔 대상이다.

---

## 2. 공통 인터페이스 기능

`JpaRepository<엔티티, 식별자 타입>`을 상속하면 기본 CRUD 메소드를 바로 사용할 수 있다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
}
```

### 인터페이스 계층

```
Repository                         ← 마커 인터페이스 (spring-data-commons)
 └─ CrudRepository                 ← save, findOne, exists, count, delete
     └─ PagingAndSortingRepository ← findAll(Sort), findAll(Pageable)
         └─ JpaRepository          ← flush, saveAndFlush, findAll() List 반환 등 (JPA 특화)
```

### 주요 메소드

| 메소드 | 동작 |
|---|---|
| `save(S)` | 새 엔티티면 `persist`, 아니면 `merge` |
| `delete(T)` | `em.remove()` 호출 |
| `findOne(ID)` | `em.find()` 호출 |
| `getOne(ID)` | `em.getReference()` 호출 → 프록시 반환 |
| `findAll(...)` | 전체 조회. `Sort`, `Pageable` 조건 전달 가능 |

> **현재 버전**: `findOne` → `findById` (반환 타입 `Optional<T>`), `getOne` → `getReferenceById`

---

## 3. 쿼리 메소드 기능

스프링 데이터 JPA에서 쿼리를 정의하는 주된 방법은 두 가지다.

1. 메소드 이름으로 쿼리 생성
2. `@Query`로 리포지토리 인터페이스에 쿼리 직접 정의

> **참고: JPA NamedQuery**
> 엔티티 클래스에 `@NamedQuery(name = "Member.findByUsername", query = "...")`로 JPQL에 이름을 붙여 두고, 리포지토리 메소드 이름이 `도메인 클래스.메소드 이름`과 일치하면 그 쿼리를 실행하는 방식이다. 로딩 시점에 문법 검증이 된다는 장점은 있지만, **쿼리가 엔티티 클래스에 섞여 엔티티가 지저분해지고, 쿼리와 그 쿼리를 쓰는 메소드가 서로 다른 파일에 흩어진다.** 같은 장점(로딩 시점 검증)을 `@Query`가 리포지토리 메소드 바로 위에서 제공하므로 실무에서는 거의 쓰지 않는다.

### 3.1 메소드 이름으로 쿼리 생성

정해진 규칙에 따라 메소드 이름을 지으면 JPQL을 생성해서 실행한다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    List<Member> findByEmailAndName(String email, String name);
}
```

```sql
-- 실행되는 JPQL
select m from Member m where m.email = ?1 and m.name = ?2
```

주요 키워드:

| 키워드 | 예 | JPQL |
|---|---|---|
| And | `findByLastnameAndFirstname` | `where x.lastname = ?1 and x.firstname = ?2` |
| Or | `findByLastnameOrFirstname` | `where x.lastname = ?1 or x.firstname = ?2` |
| Between | `findByStartDateBetween` | `where x.startDate between ?1 and ?2` |
| LessThan | `findByAgeLessThan` | `where x.age < ?1` |
| IsNull | `findByAgeIsNull` | `where x.age is null` |
| Like | `findByFirstnameLike` | `where x.firstname like ?1` |
| OrderBy | `findByAgeOrderByLastnameDesc` | `where x.age = ?1 order by x.lastname desc` |
| In | `findByAgeIn(Collection<Age> ages)` | `where x.age in ?1` |

**장점**: 엔티티 필드명이 바뀌면 애플리케이션 로딩 시점에 오류가 발생한다 → 컴파일 수준에 가까운 안정성.
**단점**: 조건이 2~3개를 넘어가면 메소드 이름이 지나치게 길어진다.

### 3.2 @Query, 리포지토리 메소드에 쿼리 정의

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    @Query("select m from Member m where m.username = ?1")
    Member findByUsername(String username);

    // 네이티브 SQL
    @Query(value = "SELECT * FROM MEMBER WHERE USERNAME = ?0", nativeQuery = true)
    Member findByUsernameNative(String username);
}
```

- 이름 없는 Named 쿼리라고 볼 수 있다.
- **애플리케이션 로딩 시점에 JPQL 문법 오류를 발견**할 수 있다는 것이 가장 큰 장점.
- 책 기준 주의: JPQL 위치 기반 파라미터는 1부터, 네이티브 SQL은 0부터 시작한다고 설명한다.

### 3.3 파라미터 바인딩

```java
select m from Member m where m.username = ?1      // 위치 기반
select m from Member m where m.username = :name   // 이름 기반
```

```java
@Query("select m from Member m where m.username = :name")
Member findByUsername(@Param("name") String username);
```

→ 코드 가독성과 유지보수성을 위해 **이름 기반 바인딩을 사용**한다.

### 3.4 벌크성 수정 쿼리

```java
@Modifying(clearAutomatically = true)
@Query("update Product p set p.price = p.price * 1.1 where p.stockAmount < :stockAmount")
int bulkPriceUp(@Param("stockAmount") String stockAmount);
```

- 벌크 수정·삭제 쿼리에는 `@Modifying`이 필요하다. 반환값은 영향받은 엔티티 수.
- **벌크 연산은 영속성 컨텍스트를 무시하고 DB에 직접 실행**된다 → 이미 로딩된 엔티티와 DB 상태가 어긋날 수 있다.
- `clearAutomatically = true`로 쿼리 실행 후 영속성 컨텍스트를 초기화한다. (기본값 `false`)

### 3.5 반환 타입

```java
List<Member> findByName(String name);   // 컬렉션
Member findByEmail(String email);       // 단건
```

| 상황 | 컬렉션 반환 | 단건 반환 |
|---|---|---|
| 결과 없음 | 빈 컬렉션 | `null` |
| 결과 2건 이상 | 정상 | `IncorrectResultSizeDataAccessException` |

- 단건 조회는 내부적으로 `getSingleResult()`를 호출하는데, 결과가 없어 `NoResultException`이 발생하면 스프링 데이터 JPA가 예외를 무시하고 `null`을 반환한다.

> **현재 버전**: 단건 반환 타입으로 `Optional<Member>`를 쓰는 것이 일반적이다.

### 3.6 페이징과 정렬

파라미터로 `Sort`, `Pageable`을 받을 수 있다.

```java
Page<Member> findByName(String name, Pageable pageable);   // count 쿼리 추가 실행
Slice<Member> findByName(String name, Pageable pageable);  // count 없이 다음 페이지 여부만
List<Member> findByName(String name, Pageable pageable);   // count 쿼리 없음
List<Member> findByName(String name, Sort sort);
```

```java
// 이름이 "김"으로 시작하는 회원을 이름 내림차순으로, 0페이지부터 10건씩
PageRequest pageRequest = PageRequest.of(0, 10, Sort.by(Sort.Direction.DESC, "name"));

Page<Member> result = memberRepository.findByNameStartingWith("김", pageRequest);

List<Member> members = result.getContent();     // 조회된 데이터
int totalPages = result.getTotalPages();        // 전체 페이지 수
boolean hasNext = result.hasNext();             // 다음 페이지 존재 여부
```

- 반환 타입이 `Page`면 전체 데이터 건수를 구하는 **count 쿼리가 추가로 실행**된다.
- 페이지 번호는 **0부터 시작**한다.

> 책 원문은 `new PageRequest(0, 10, new Sort(Direction.DESC, "name"))` 생성자 방식이다. 현재 생성자는 `protected`라서 정적 팩토리 `of()`를 써야 한다.

#### `Pageable`과 구현체 `PageRequest`

`Pageable`은 **"몇 번째 페이지를, 몇 개씩, 어떤 순서로"라는 요청 정보를 담는 인터페이스**이고, `PageRequest`가 그 기본 구현체다.

```
Pageable (interface)
 ├─ AbstractPageRequest (abstract)  ← page, size 보관 + 검증 + offset 계산
 │   └─ PageRequest                 ← Sort 보관, 정적 팩토리 제공 (일반적으로 쓰는 것)
 └─ Unpaged                         ← Pageable.unpaged(), 페이징 없음을 표현
```

아래는 실제 소스를 핵심만 남겨 단순화한 코드다.

```java
public interface Pageable {

    int getPageNumber();      // 요청 페이지 번호 (0부터)
    int getPageSize();        // 페이지당 건수
    long getOffset();         // 건너뛸 행 수 → SQL OFFSET
    Sort getSort();           // 정렬 조건

    Pageable next();          // 다음 페이지 요청
    Pageable previousOrFirst();
    Pageable first();
    Pageable withPage(int pageNumber);
    boolean hasPrevious();

    static Pageable unpaged() { return Unpaged.sorted(Sort.unsorted()); }
    static Pageable ofSize(int pageSize) { return PageRequest.of(0, pageSize); }

    default boolean isPaged() { return true; }
}
```

```java
public abstract class AbstractPageRequest implements Pageable, Serializable {

    private final int page;
    private final int size;

    public AbstractPageRequest(int page, int size) {
        if (page < 0) {
            throw new IllegalArgumentException("Page index must not be less than zero");
        }
        if (size < 1) {
            throw new IllegalArgumentException("Page size must not be less than one");
        }
        this.page = page;
        this.size = size;
    }

    public int getPageNumber() { return page; }
    public int getPageSize()   { return size; }

    // OFFSET = 페이지 번호 × 페이지 크기
    public long getOffset() { return (long) page * (long) size; }

    public boolean hasPrevious() { return page > 0; }

    public Pageable previousOrFirst() { return hasPrevious() ? previous() : first(); }

    public abstract Pageable next();
    public abstract Pageable previous();
    public abstract Pageable first();
}
```

```java
public class PageRequest extends AbstractPageRequest {

    private final Sort sort;

    protected PageRequest(int page, int size, Sort sort) {
        super(page, size);
        this.sort = sort;
    }

    // 생성은 정적 팩토리로
    public static PageRequest of(int page, int size) {
        return of(page, size, Sort.unsorted());
    }
    public static PageRequest of(int page, int size, Sort sort) {
        return new PageRequest(page, size, sort);
    }
    public static PageRequest of(int page, int size, Sort.Direction direction, String... properties) {
        return of(page, size, Sort.by(direction, properties));
    }
    public static PageRequest ofSize(int pageSize) {
        return of(0, pageSize);
    }

    public Sort getSort() { return sort; }

    // 불변 객체: 상태를 바꾸지 않고 새 객체를 반환
    public PageRequest next()     { return new PageRequest(getPageNumber() + 1, getPageSize(), sort); }
    public PageRequest previous() { return getPageNumber() == 0 ? this
                                         : new PageRequest(getPageNumber() - 1, getPageSize(), sort); }
    public PageRequest first()    { return new PageRequest(0, getPageSize(), sort); }

    public PageRequest withPage(int pageNumber)  { return new PageRequest(pageNumber, getPageSize(), sort); }
    public PageRequest withSort(Sort sort)       { return new PageRequest(getPageNumber(), getPageSize(), sort); }
}
```

정리하면:

- `PageRequest`는 **불변 객체**다. `next()`, `withSort()` 등은 기존 객체를 바꾸지 않고 새 객체를 만든다.
- 생성 시점에 `page < 0`, `size < 1`을 검증한다.
- 리포지토리는 `getOffset()`, `getPageSize()`, `getSort()`를 꺼내 쿼리에 적용한다.

```java
PageRequest pageable = PageRequest.of(2, 10, Sort.by("name").descending());
memberRepository.findAll(pageable);
```

```sql
-- getOffset() = 2 * 10 = 20, getPageSize() = 10
select ... from member m order by m.name desc limit 10 offset 20
-- 반환 타입이 Page이므로 count 쿼리 추가 실행
select count(m.id) from member m
```

#### 결과 쪽: `Page`와 `Slice`

요청이 `Pageable`이라면, 응답은 `Slice` / `Page`다.

```java
public interface Slice<T> {
    List<T> getContent();
    int getNumber();          // 현재 페이지 번호
    int getSize();
    boolean hasNext();
    boolean hasPrevious();
    Pageable nextPageable();
    <U> Slice<U> map(Function<? super T, ? extends U> converter);
}

public interface Page<T> extends Slice<T> {
    long getTotalElements();  // 전체 건수 → count 쿼리 필요
    int getTotalPages();
    <U> Page<U> map(Function<? super T, ? extends U> converter);
}
```

- `Slice`: 실제로는 `size + 1`건을 조회해서 다음 페이지 존재 여부만 판단. count 쿼리 없음 → "더보기"/무한 스크롤에 적합.
- `Page`: 전체 건수가 필요한 페이지 번호 UI에 사용. 구현체는 `PageImpl`.
- count 쿼리가 무거우면 분리할 수 있다.

```java
@Query(value = "select m from Member m left join m.team t",
       countQuery = "select count(m) from Member m")
Page<Member> findMemberAllCountBy(Pageable pageable);
```

### 3.7 힌트

```java
@QueryHints(value = { @QueryHint(name = "org.hibernate.readOnly", value = "true") },
            forCounting = true)
Page<Member> findByName(String name, Pageable pageable);
```

- SQL 힌트가 아니라 **JPA 구현체(하이버네이트)에 전달하는 힌트**다.
- `forCounting = true`면 페이징용 count 쿼리에도 힌트를 적용한다. (기본값 `true`)

### 3.8 Lock

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
List<Member> findByName(String name);
```

- 쿼리 시 락을 걸 때 사용한다. 락 자체는 16장(트랜잭션과 락)에서 다룬다.

---

## 4. 명세 (Specification)

### 4.1 어떤 문제를 풀기 위한 것인가

검색 화면에는 보통 조건이 여러 개 있고, 사용자가 **어떤 조건을 입력할지 미리 알 수 없다.** 예를 들어 주문 검색에서 회원 이름만 넣을 수도, 주문 상태만 고를 수도, 둘 다 넣을 수도 있다.

메소드 이름 쿼리나 `@Query`는 조건이 고정되어 있어서 이런 상황을 처리하지 못한다. 조합마다 메소드를 만들면 조건이 n개일 때 2ⁿ개가 필요하다. JPQL 문자열을 `if`로 이어 붙이는 방법은 공백, `and` 위치 실수가 잦고 런타임에야 오류가 드러난다.

명세는 이 문제를 **"검색 조건 하나하나를 독립된 객체로 만들고, 필요한 것만 골라 조합한다"**는 방식으로 푼다.

### 4.2 Specification이 하는 일

`Specification<T>`는 메소드가 하나뿐인 인터페이스다.

```java
public interface Specification<T> {
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}
```

이 메소드가 반환하는 `Predicate`는 **WHERE 절에 들어갈 조건식 하나**다. 즉 Specification 하나 = 조건 하나다.

세 파라미터는 JPA Criteria API의 객체들이다.

| 파라미터 | 역할 | 예 |
|---|---|---|
| `Root<T>` | 조회 대상 엔티티. 필드 경로를 꺼내거나 조인할 때 사용 | `root.get("status")`, `root.join("member")` |
| `CriteriaQuery<?>` | 쿼리 전체. distinct 등 쿼리 자체를 조정할 때 사용 | `query.distinct(true)` |
| `CriteriaBuilder` | 조건식을 만드는 도구 | `cb.equal(...)`, `cb.like(...)`, `cb.between(...)` |

그리고 중요한 규칙 하나: **`toPredicate`가 `null`을 반환하면 그 조건은 무시된다.** 이 규칙 덕분에 "값이 없으면 조건을 빼라"는 동적 쿼리 로직을 각 조건 안에 넣을 수 있다.

### 4.3 코드로 보기

**1) 리포지토리에 `JpaSpecificationExecutor` 추가**

```java
public interface OrderRepository
        extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order> {
}
```

`JpaSpecificationExecutor`는 Specification을 받는 조회 메소드를 제공한다.

```java
Optional<T> findOne(Specification<T> spec);
List<T> findAll(Specification<T> spec);
Page<T> findAll(Specification<T> spec, Pageable pageable);
List<T> findAll(Specification<T> spec, Sort sort);
long count(Specification<T> spec);
```

**2) 조건을 하나씩 정의**

```java
public class OrderSpec {

    // 조건 1: 회원 이름이 일치
    public static Specification<Order> memberName(String memberName) {
        return (root, query, cb) -> {
            if (!StringUtils.hasText(memberName)) {
                return null;                                 // 값이 없으면 조건 제외
            }
            Join<Order, Member> m = root.join("member", JoinType.INNER);  // order join member
            return cb.equal(m.get("name"), memberName);      // m.name = ?
        };
    }

    // 조건 2: 주문 상태가 일치
    public static Specification<Order> orderStatus(OrderStatus status) {
        return (root, query, cb) -> {
            if (status == null) {
                return null;
            }
            return cb.equal(root.get("status"), status);     // o.status = ?
        };
    }
}
```

**3) 필요한 조건을 조합해서 조회**

```java
import static org.springframework.data.jpa.domain.Specification.where;
import static jpabook.OrderSpec.*;

List<Order> result = orderRepository.findAll(
        where(memberName(name)).and(orderStatus(status)));
```

입력값에 따라 실제로 실행되는 쿼리:

```sql
-- name = "kim", status = ORDER
select o from Order o join o.member m where m.name = 'kim' and o.status = 'ORDER'

-- name = null, status = ORDER  → memberName 조건이 null을 반환해 빠짐
select o from Order o where o.status = 'ORDER'

-- 둘 다 null → 조건 없음
select o from Order o
```

조합 메소드는 `and`, `or`, `not`, `where`, `allOf`, `anyOf`가 있다. 결국 명세는 **"조건을 만드는 코드"와 "조건을 조합하는 코드"를 분리**해 주는 장치이고, 실제 쿼리 생성은 스프링 데이터 JPA가 Criteria API로 처리한다.

### 4.4 한계

- 내부가 Criteria API라서 조건이 복잡해지면 코드가 금방 읽기 어려워진다.
- `root.get("status")`처럼 **필드명을 문자열로** 쓰므로 오타가 런타임에 드러난다. (메타모델 `Order_.status`로 보완 가능하지만 번거롭다)
- fetch join, DTO 조회, 서브쿼리 등을 조합하기 불편하다.

이 때문에 실무에서는 동적 쿼리에 명세보다 **QueryDSL을 주로 사용**한다. (8장 참고)

> **현재 버전**: 책의 `Specifications` 유틸 클래스와 익명 클래스 방식은 `Specification.where(...)` + 람다로 대체되었다. 위 코드는 현재 문법 기준이다.

---

## 5. 사용자 정의 리포지토리 구현

### 5.1 언제 직접 구현해야 하나

스프링 데이터 JPA가 자동으로 만들어 주는 메소드(공통 인터페이스, 메소드 이름 쿼리, `@Query`, 명세)로 해결되지 않을 때다.

| 상황 | 예 |
|---|---|
| 복잡한 동적 쿼리 | 검색 조건이 많고 조인·정렬까지 조건에 따라 바뀜 → QueryDSL로 구현 (가장 흔한 경우) |
| JPA 외 기술을 같이 써야 할 때 | 대량 insert를 `JdbcTemplate.batchUpdate`로, 통계 쿼리를 MyBatis로 |
| `EntityManager`를 직접 다뤄야 할 때 | 스트리밍 조회 후 주기적으로 `flush()`/`clear()`, 영속성 컨텍스트 세밀 제어 |
| 조회 결과를 복잡하게 가공할 때 | 여러 쿼리 결과를 조립해서 DTO 하나로 반환 |

### 5.2 구현 방법

리포지토리 전체를 구현하지 않고 **필요한 메소드만 구현**해서 기존 리포지토리에 붙인다.

```java
// 1. 사용자 정의 인터페이스
public interface MemberRepositoryCustom {
    List<Member> findMemberCustom();
}

// 2. 구현 클래스: 리포지토리 인터페이스 이름 + Impl
@RequiredArgsConstructor
public class MemberRepositoryImpl implements MemberRepositoryCustom {

    private final EntityManager em;

    @Override
    public List<Member> findMemberCustom() {
        return em.createQuery("select m from Member m", Member.class)
                 .getResultList();
    }
}

// 3. 리포지토리 인터페이스에서 사용자 정의 인터페이스 상속
public interface MemberRepository
        extends JpaRepository<Member, Long>, MemberRepositoryCustom {
}
```

- 클래스 이름 규칙: `리포지토리 인터페이스 이름 + Impl` → 스프링 데이터 JPA가 사용자 정의 구현으로 인식.
- 접미사는 `repositoryImplementationPostfix`로 변경 가능.

> **현재 버전**: `사용자 정의 인터페이스 이름 + Impl` (예: `MemberRepositoryCustomImpl`) 방식도 지원하며, 이 방식이 권장된다.

> **실무 팁**: 특정 화면이나 API 전용 조회 쿼리라면, 굳이 사용자 정의 리포지토리로 합치지 않고 `MemberQueryRepository` 같은 **별도 클래스(@Repository)로 분리**하는 경우도 많다. 엔티티의 핵심 저장/조회 로직과 화면 맞춤 조회 로직의 변경 주기가 다르기 때문이다.

---

## 6. Web 확장

스프링 MVC에서 쓰는 편의 기능 두 가지. 실무에서의 사용 빈도는 크게 다르다.

| 기능 | 실무 사용 |
|---|---|
| 페이징·정렬 파라미터 바인딩 | **매우 많이 씀** |
| 도메인 클래스 컨버터 | 거의 안 씀 |

### 6.1 설정

```java
@Configuration
@EnableWebMvc
@EnableSpringDataWebSupport
public class WebAppConfig { ... }
```

> **현재 버전**: Spring Boot는 자동 설정되므로 별도 선언이 필요 없다.

### 6.2 페이징과 정렬 기능

컨트롤러 파라미터에 `Pageable`을 선언하면 요청 파라미터를 읽어 `PageRequest`를 만들어 준다.

```java
@GetMapping("/members")
public Page<MemberDto> list(Pageable pageable) {
    return memberRepository.findAll(pageable)
                           .map(MemberDto::new);     // 엔티티 → DTO 변환
}
```

요청 예: `/members?page=0&size=20&sort=name,desc&sort=age`

| 파라미터 | 의미 |
|---|---|
| `page` | 현재 페이지 (0부터 시작) |
| `size` | 한 페이지에 노출할 데이터 건수 |
| `sort` | `정렬 속성,방향(asc/desc)`. 여러 번 지정 가능 |

#### 기본값 설정

```java
// 메소드 단위
@GetMapping("/members")
public Page<MemberDto> list(
        @PageableDefault(size = 12, sort = "name", direction = Sort.Direction.DESC)
        Pageable pageable) { ... }
```

```yaml
# 전역 (Spring Boot)
spring:
  data:
    web:
      pageable:
        default-page-size: 20     # 기본 20
        max-page-size: 2000       # size 요청 상한, 기본 2000
        one-indexed-parameters: false
```

#### 페이징 정보가 둘 이상일 때

```java
public String list(@Qualifier("member") Pageable memberPageable,
                   @Qualifier("order")  Pageable orderPageable) { ... }
// /members?member_page=0&order_page=1
```

#### 실무 주의점

- **엔티티 대신 DTO로 반환**한다. `Page.map()`으로 변환하면 페이징 메타정보는 그대로 유지된다.
- **1부터 시작하는 페이지**: `one-indexed-parameters: true`로 요청은 1부터 받을 수 있지만, 응답 `Page`의 `number` 값은 여전히 0 기준이라 혼란스럽다. 필요하면 직접 `PageRequest.of(page - 1, size)`로 만들거나 응답 DTO를 따로 정의한다.
- **sort 파라미터 노출**: 클라이언트가 아무 필드명이나 정렬 조건으로 보낼 수 있다. 허용 필드를 제한하거나, 정렬 기준을 서버에서 정하는 것이 안전하다.
- **`Page` 직접 직렬화**: Spring Data 3.3부터는 `PageImpl`을 JSON으로 바로 반환하면 경고가 뜬다(JSON 구조가 버전마다 바뀔 수 있음). `@EnableSpringDataWebSupport(pageSerializationMode = VIA_DTO)`를 쓰거나 응답용 페이지 DTO를 직접 만든다.

### 6.3 도메인 클래스 컨버터 기능 (거의 쓰지 않음)

HTTP 파라미터로 넘어온 엔티티 아이디로 엔티티를 찾아 바인딩해 준다.

```java
@GetMapping("/members/{id}")
public String findMember(@PathVariable("id") Member member) {   // id로 findById 자동 호출
    return member.getUsername();
}
```

잘 쓰지 않는 이유:

- **조회 용도로만 쓸 수 있다.** 트랜잭션 밖에서 조회된 엔티티라 수정해도 DB에 반영되지 않는다.
- **엔티티가 컨트롤러 파라미터로 노출된다.** API 계층은 DTO로 받고 반환하는 것이 일반적이다.
- **조회 로직이 숨는다.** 권한 체크, 없는 id 처리(404 등), fetch join 같은 조회 전략을 서비스 계층에서 명시적으로 다루는 편이 낫다.

---

## 7. 스프링 데이터 JPA가 사용하는 구현체

> 이 절은 새 기능이 아니라 **지금까지 쓴 기능이 내부에서 어떻게 동작하는지**를 보는 절이다. 트랜잭션 없이 리포지토리를 호출해도 동작하는 이유, `save()` 호출 시 SELECT가 한 번 더 나가는 이유 등이 모두 이 구현체에서 나온다.

공통 인터페이스는 `SimpleJpaRepository`가 구현한다.

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {

    @Transactional
    public <S extends T> S save(S entity) {
        if (entityInformation.isNew(entity)) {
            em.persist(entity);
            return entity;
        } else {
            return em.merge(entity);
        }
    }
    ...
}
```

| 요소 | 의미 |
|---|---|
| `@Repository` | JPA 예외를 스프링 추상화 예외(`DataAccessException`)로 변환 |
| `@Transactional` (쓰기 메소드) | 서비스 계층에 트랜잭션이 없어도 리포지토리에서 트랜잭션 시작. 있으면 전파받아 사용 |
| `@Transactional(readOnly = true)` (클래스 레벨) | 조회 메소드 기본값. 플러시를 생략해 약간의 성능 이점 |
| `save()` | 새 엔티티면 `persist`, 아니면 `merge` |

### 새 엔티티 판단 기준 (`isNew`)

- 식별자가 객체 타입이면 `null`, 기본 타입이면 `0`일 때 새 엔티티로 판단.
- 식별자를 직접 할당하는 경우(`@GeneratedValue` 미사용)에는 저장 시점에 이미 id가 있으므로 새 엔티티가 아니라고 판단 → `merge` 호출 → **DB에 SELECT를 한 번 날려 확인한 뒤 INSERT**한다.
- `Persistable` 인터페이스를 구현하면 판단 로직을 바꿀 수 있다.

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Item implements Persistable<String> {

    @Id
    private String id;               // 직접 할당

    @CreatedDate
    private LocalDateTime createdDate;

    @Override
    public String getId() { return id; }

    @Override
    public boolean isNew() {
        return createdDate == null;  // 생성일이 없으면 새 엔티티
    }
}
```

### `merge` 주의점

`merge`는 넘긴 객체를 영속 상태로 만드는 게 아니라 **새 영속 객체를 반환**한다. 따라서 `save()`의 반환값을 써야 한다.

```java
Item saved = itemRepository.save(item);   // 이후 로직은 saved로
```

---

## 8. 스프링 데이터 JPA와 QueryDSL 통합

QueryDSL 자체는 동적 쿼리 작성에 **매우 많이 쓰인다.** 다만 책에서 소개하는 두 가지 통합 방식은 현재 실무에서 잘 쓰지 않고, **사용자 정의 리포지토리 + `JPAQueryFactory`** 조합이 사실상 표준이다.

| 방식 | 실무 사용 |
|---|---|
| `QuerydslPredicateExecutor` | 거의 안 씀 |
| `QuerydslRepositorySupport` | 거의 안 씀 |
| 사용자 정의 리포지토리 + `JPAQueryFactory` | **주로 사용** |

### 8.1 QuerydslPredicateExecutor (책 소개, 잘 안 씀)

```java
public interface ItemRepository
        extends JpaRepository<Item, Long>, QuerydslPredicateExecutor<Item> {
}

Iterable<Item> result = itemRepository.findAll(
        item.name.contains("장난감").and(item.price.between(10000, 20000)));
```

잘 쓰지 않는 이유:
- **명시적 조인(join, fetch join)이 불가능**하다. 묵시적 조인만 되어 N+1이나 비효율 쿼리가 생기기 쉽다.
- 서비스 계층이 QueryDSL `Predicate`를 직접 만들어 넘겨야 해서, **쿼리 기술이 서비스 계층까지 새어 나간다.**

### 8.2 QuerydslRepositorySupport (책 소개, 잘 안 씀)

```java
public class OrderRepositoryImpl extends QuerydslRepositorySupport
        implements OrderRepositoryCustom {

    public OrderRepositoryImpl() { super(Order.class); }

    public List<Order> search(OrderSearch cond) {
        return from(order)          // from부터 시작하는 QueryDSL 3 스타일
                .where(...)
                .fetch();
    }
}
```

잘 쓰지 않는 이유:
- `from()`으로 시작하는 QueryDSL 3 스타일이라 `select`부터 시작하는 QueryDSL 4 문법과 맞지 않는다.
- 페이징 헬퍼 `getQuerydsl().applyPagination()`이 스프링 `Sort`를 QueryDSL 정렬로 변환하는데, 조인된 필드 등에서 제대로 동작하지 않는 경우가 있다.
- `JPAQueryFactory`를 직접 쓰는 것보다 얻는 이점이 크지 않다.

### 8.3 실무 방식: 사용자 정의 리포지토리 + JPAQueryFactory

**1) `JPAQueryFactory` 빈 등록**

```java
@Configuration
public class QuerydslConfig {
    @Bean
    public JPAQueryFactory jpaQueryFactory(EntityManager em) {
        return new JPAQueryFactory(em);
    }
}
```

**2) 사용자 정의 인터페이스와 리포지토리 연결**

```java
public interface OrderRepositoryCustom {
    Page<OrderDto> search(OrderSearchCond cond, Pageable pageable);
}

public interface OrderRepository
        extends JpaRepository<Order, Long>, OrderRepositoryCustom {
}
```

**3) 구현**

```java
@RequiredArgsConstructor
public class OrderRepositoryCustomImpl implements OrderRepositoryCustom {

    private final JPAQueryFactory queryFactory;

    @Override
    public Page<OrderDto> search(OrderSearchCond cond, Pageable pageable) {

        // 내용 조회
        List<OrderDto> content = queryFactory
                .select(new QOrderDto(order.id, member.name, order.status, order.orderDate))
                .from(order)
                .join(order.member, member)                 // 명시적 조인 가능
                .where(
                        memberNameEq(cond.getMemberName()),
                        statusEq(cond.getStatus()),
                        orderDateBetween(cond.getFrom(), cond.getTo())
                )
                .orderBy(order.orderDate.desc())
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .fetch();

        // count 쿼리 (조인이 필요 없으면 빼서 가볍게)
        JPAQuery<Long> countQuery = queryFactory
                .select(order.count())
                .from(order)
                .join(order.member, member)
                .where(
                        memberNameEq(cond.getMemberName()),
                        statusEq(cond.getStatus()),
                        orderDateBetween(cond.getFrom(), cond.getTo())
                );

        // 첫 페이지인데 content가 size보다 적거나, 마지막 페이지면 count 쿼리 생략
        return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
    }

    // 조건 메소드: null을 반환하면 where 절에서 무시됨
    private BooleanExpression memberNameEq(String name) {
        return StringUtils.hasText(name) ? member.name.eq(name) : null;
    }

    private BooleanExpression statusEq(OrderStatus status) {
        return status != null ? order.status.eq(status) : null;
    }

    private BooleanExpression orderDateBetween(LocalDateTime from, LocalDateTime to) {
        if (from == null || to == null) return null;
        return order.orderDate.between(from, to);
    }
}
```

**4) 사용**

```java
Page<OrderDto> page = orderRepository.search(cond, PageRequest.of(0, 20));
```

이 방식이 주로 쓰이는 이유:

- **동적 조건 처리 방식이 명세와 같으면서 훨씬 읽기 쉽다.** 조건 메소드가 `null`을 반환하면 무시되는 구조라 명세의 장점(조건 단위 분리, 재사용)을 그대로 가져가고, 코드는 JPQL에 가깝게 읽힌다.
- **타입 안전**: `order.status`처럼 Q타입 필드를 쓰므로 필드명 오타가 컴파일 시점에 잡힌다.
- **조인, fetch join, DTO 프로젝션, 서브쿼리**를 모두 쓸 수 있다.
- 서비스 계층은 `orderRepository.search(cond, pageable)`만 호출하므로 **QueryDSL이 리포지토리 안에 갇힌다.**
- `PageableExecutionUtils.getPage()`로 불필요한 count 쿼리를 줄일 수 있다.

> **참고**: QueryDSL 원본 프로젝트는 업데이트가 뜸해져, 최근에는 커뮤니티 포크(OpenFeign 측 유지보수 버전)를 사용하는 프로젝트도 있다. 도입 시 Spring Boot·Hibernate 버전과의 호환성을 확인한다.

---

## 책 이후 변경점

| 책 (Spring Data JPA 1.x) | 현재 (2.x / 3.x) |
|---|---|
| `findOne(ID)` → `T` | `findById(ID)` → `Optional<T>` |
| `getOne(ID)` | `getReferenceById(ID)` |
| `delete(ID)` | `deleteById(ID)` |
| `save(Iterable)` | `saveAll(Iterable)` |
| `new PageRequest(...)`, `new Sort(...)` | `PageRequest.of(...)`, `Sort.by(...)` |
| `Specifications.where(...)` | `Specification.where(...)` |
| `QueryDslPredicateExecutor` | `QuerydslPredicateExecutor` |
| `QueryDslRepositorySupport` | `QuerydslRepositorySupport` (실무에선 `JPAQueryFactory` 직접 사용) |
| QueryDSL 3 `list()` | QueryDSL 4 `fetch()` |
| `XxxRepositoryImpl`만 인식 | `XxxRepositoryCustomImpl`도 인식 (권장) |
| XML / `@EnableJpaRepositories` 설정 | Spring Boot 자동 설정 |
| `javax.persistence.*` | `jakarta.persistence.*` (Spring Boot 3 이상) |
