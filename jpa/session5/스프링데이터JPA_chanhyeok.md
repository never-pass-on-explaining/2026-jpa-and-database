# Chapter 12. 스프링 데이터 JPA

## 1. 스프링 데이터 JPA란

데이터 접근 계층은 리포지토리마다 저장, 조회, 전체 조회 같은 CRUD 코드를 비슷하게 반복한다. 제네릭과 상속으로 공통 부모 클래스(GenericDAO)를 만들 수도 있지만, 부모 클래스에 너무 종속되고 구현 클래스 상속의 단점이 그대로 드러난다.

스프링 데이터 JPA는 **리포지토리를 인터페이스로만 작성하면 실행 시점에 구현 객체를 동적으로 만들어 주입**한다. 구현 클래스 없이 인터페이스만으로 데이터 접근 계층을 개발할 수 있다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    Member findByUsername(String username);   // 메서드 이름을 분석해 JPQL 생성
}
```

## 2. 공통 인터페이스

```
Repository
 └─ CrudRepository                 ← save, findOne, count, delete
     └─ PagingAndSortingRepository ← findAll(Sort), findAll(Pageable)
         └─ JpaRepository          ← flush, saveAndFlush, deleteInBatch 등 JPA 특화 기능
```

`PagingAndSortingRepository`까지는 스프링 데이터 공통이고, `JpaRepository`가 JPA에 특화된 기능을 더한다.

| 메서드 | 동작 |
| --- | --- |
| `save(S)` | 새 엔티티면 저장(`persist`), 이미 있으면 병합(`merge`) |
| `delete(T)` | `em.remove()` |
| `findOne(ID)` | `em.find()` |
| `getOne(ID)` | `em.getReference()`, 프록시 반환 |
| `findAll(...)` | 전체 조회. `Sort`, `Pageable`을 넘길 수 있다 |

> 지금 버전에서 `findOne`은 `findById`(반환 `Optional`), `getOne`은 `getReferenceById`다. `getReferenceById`가 프록시를 돌려주는 것은 8장의 `em.getReference()`와 같다.

## 3. 쿼리 메서드

### 메서드 이름으로 쿼리 생성

```java
List<Member> findByEmailAndName(String email, String name);
// select m from Member m where m.email = ?1 and m.name = ?2
```

`And`, `Or`, `Between`, `LessThan`, `Like`, `StartingWith`, `In`, `OrderBy`, `IgnoreCase` 같은 키워드를 조합한다. 엔티티 필드명을 바꾸면 메서드 이름도 함께 바꿔야 하며, 안 바꾸면 **애플리케이션 시작 시점에 오류**가 난다.

### `@Query`로 직접 정의

```java
@Query("select m from Member m where m.username = :name")
Member findByUsername(@Param("name") String username);

@Query(value = "SELECT * FROM MEMBER WHERE USERNAME = ?1", nativeQuery = true)
Member findByUsernameNative(String username);
```

- 메서드 이름으로 표현하기 어려운 쿼리는 JPQL을 직접 쓴다. **애플리케이션 시작 시점에 JPQL 문법 오류를 발견**할 수 있다.
- 네이티브 SQL이 필요하면 `nativeQuery = true`를 붙인다.
- 파라미터는 순서(`?1`)나 이름(`:name` + `@Param`)으로 바인딩한다. 순서가 바뀌어도 안전한 이름 기반을 쓴다.

### 반환 타입

| 상황 | 컬렉션 반환 | 단건 반환 |
| --- | --- | --- |
| 결과 없음 | 빈 컬렉션 | `null` |
| 결과 2건 이상 | 정상 | 예외 |

단건 조회는 내부에서 `getSingleResult()`를 호출한다. 결과가 없어 `NoResultException`이 나면 스프링 데이터 JPA가 무시하고 `null`을 반환한다. 지금은 `Optional<Member>`로 받는 것이 일반적이다.

## 4. 벌크성 수정 쿼리

```java
@Modifying(clearAutomatically = true)
@Query("update Product p set p.price = p.price * 1.1 where p.stockAmount < :stockAmount")
int bulkPriceUp(@Param("stockAmount") String stockAmount);
```

- 벌크 수정, 삭제 쿼리에는 `@Modifying`을 붙인다.
- 벌크 연산은 **영속성 컨텍스트를 무시하고 DB에 바로 실행**된다. 이미 조회해 둔 엔티티는 옛날 값을 그대로 들고 있게 된다.
- `clearAutomatically = true`면 실행 뒤 영속성 컨텍스트를 비운다. 기본값은 `false`다.

## 5. 페이징과 정렬

파라미터로 `Sort`(정렬)와 `Pageable`(페이징, 내부에 `Sort` 포함)을 받는다.

```java
Page<Member> findByName(String name, Pageable pageable);   // count 쿼리 추가 실행
List<Member> findByName(String name, Pageable pageable);   // count 쿼리 없음
List<Member> findByName(String name, Sort sort);
```

```java
PageRequest pageRequest = PageRequest.of(0, 10, Sort.by(Sort.Direction.DESC, "name"));
Page<Member> result = memberRepository.findByNameStartingWith("김", pageRequest);

List<Member> members = result.getContent();   // 조회된 데이터
int totalPages = result.getTotalPages();      // 전체 페이지 수
boolean hasNext = result.hasNext();           // 다음 페이지 여부
```

- 반환 타입이 `Page`면 전체 건수를 구하는 **count 쿼리가 한 번 더 나간다.**
- 페이지는 0부터 시작한다.
- 책은 `new PageRequest(...)` 생성자를 쓰지만 지금은 `PageRequest.of(...)`를 쓴다.

> 페이징 메서드에 컬렉션 fetch join이나 `@EntityGraph`로 컬렉션을 함께 가져오면, 하이버네이트는 SQL에 LIMIT을 걸지 못하고 전체를 메모리로 가져와 자른다. 페이징은 루트 엔티티만 하고 컬렉션은 배치 조회(`default_batch_fetch_size`)로 가져온다. (4회차 딥다이브)

컨트롤러에서 `Pageable`을 파라미터로 받으면 `?page=0&size=20&sort=name,desc`가 자동으로 바인딩된다. 기본값은 page=0, size=20이고 `@PageableDefault`로 바꾼다.

## 6. 스프링 데이터 JPA가 사용하는 구현체

공통 인터페이스는 `SimpleJpaRepository`가 구현한다.

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepository<T, ID> {

    @Transactional
    public <S extends T> S save(S entity) {
        if (entityInformation.isNew(entity)) {
            em.persist(entity);
            return entity;
        } else {
            return em.merge(entity);
        }
    }
}
```

- `@Repository`: JPA 예외를 스프링이 추상화한 예외로 바꾼다.
- `@Transactional`: 등록, 수정, 삭제 메서드에 붙어 있다. 서비스 계층에 트랜잭션이 없으면 리포지토리에서 시작하고, 있으면 그대로 전파받는다.
- `@Transactional(readOnly = true)`: 조회 메서드의 기본값이다. 플러시를 생략해서 약간의 성능 이점이 있다.
- `save()`: 식별자가 객체면 `null`, 기본 타입이면 `0`일 때 새 엔티티로 보고 `persist`, 아니면 `merge`를 호출한다. `Persistable` 인터페이스를 구현하면 판단 기준을 바꿀 수 있다.

```java
public interface Persistable<ID> extends Serializable {
    ID getId();
    boolean isNew();
}
```

## 7. 정리

- 스프링 데이터 JPA는 **인터페이스만 작성하면 구현체를 만들어 준다.** 공통 CRUD는 `JpaRepository`가 제공한다.
- 쿼리는 메서드 이름으로 만들거나 `@Query`로 직접 정의한다. 둘 다 시작 시점에 오류를 잡아 준다.
- 벌크 연산은 영속성 컨텍스트를 무시하므로 `@Modifying(clearAutomatically = true)`로 비운다.
- `Pageable`과 `Page`로 페이징을 처리하고, `Page`는 count 쿼리가 추가로 나간다.
- `save()`는 식별자로 새 엔티티 여부를 판단해 `persist`나 `merge`를 호출한다.
- 사용자 정의 리포지토리는 "리포지토리 이름 + `Impl`" 규칙으로 직접 구현할 메서드만 붙인다.
