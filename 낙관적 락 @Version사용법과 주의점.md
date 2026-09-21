# 낙관적 락 @Version 사용법

동시에 여러 사용자가 같은 데이터에 접근하여 수정하려고하면 데이터의 정합성이 깨지는 문제가 발생합니다(동시성 문제).
<br>
JPA는 이러한 문제에 대해 비관적 락과 낙관적 락을 지원하는데 이번 글에서는 **낙관적 락**에대해 알아보고 적용하는 법에대해 정리하겠습니다.

## 1. 코드 예시 (Poi.java)
```java
@Entity
public class Poi extends BaseTimeEntity {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version // <== 낙관적 락
    @Column(name = "version")
    private Long version;
    
    // ... 다른 필드들 생략 ...
}
```

<br>

## 2. 낙관적 락이란?
대부분의 트랜잭션은 서로 충돌하지 않을 것이다라고 낙관적으로 가정하고 데이터베이스에 접근하는 방식입니다.
<br>
DB 자체의 레코드에 실제 락(비관적 락)을 걸면 대기 시간이 생겨 성능이 저하될 수 있지만, 낙관적 락은 JPA 레벨에서 단순히 **버전 번호를 비교**하여 충돌을 감지하므로 성능 저하가 덜합니다.

<br>

## 3. @Version은 어떻게 동작할까요?
엔티티 클래스 내부에 숫자(`Long`, `Integer` 등) 타입 필드를 선언하고 `@Version` 애노테이션을 붙여주면 JPA가 자동으로 버전 관리를 시작합니다.
>다른 필드가 아닌 version을 관리하는 필드여야함

1. 데이터를 DB에서 처음 조회할 때 `version` 값도 함께 읽어옵니다. (예: `version = 1`)
2. 데이터(예: 위시 횟수, 조회수 등)를 수정하고 트랜잭션이 커밋될 때, JPA는 UPDATE 쿼리의 조건(WHERE 절)으로 방금 읽어온 버전을 포함시킵니다.
   - 쿼리 예시: `UPDATE poi SET wish_count = wish_count + 1, version = 2 WHERE id = 1 AND version = 1;`
3. 만약 내가 수정하는 동안 **다른 사용자가 먼저 데이터를 수정해버렸다면**, DB에 저장된 실제 버전은 이미 `2`로 올라가 있을 것입니다.
4. 그러면 내가 날린 쿼리의 `WHERE version = 1` 조건에 맞는 데이터가 없으므로 업데이트되는 행(Row)은 0이 됩니다.
5. JPA는 아무 행도 업데이트되지 않은 것을 보고 **충돌이 발생했다**고 판단하여 즉시 `OptimisticLockException` 예외를 발생시킵니다.

이를 통해 `Poi` 엔티티의 조회수(`viewCount`), 위시 횟수(`wishCount`), 인기 점수(`popularityScore`) 등 동시에 업데이트가 빈번하게 일어날 수 있는 데이터가 덮어씌워져 유실(Lost Update)되는 현상을 안전하게 방어할 수 있습니다.

<br>

## 4. 동시성 문제 해결을 위한 추가 애노테이션 (@DynamicUpdate, @Modifying, @Query)

낙관적 락만으로는 모든 동시성 문제를 해결하기 어려울 수 있습니다. 그리고 이를 보완하기 위해 아래 기법들을 함께 활용하여 **조회수 증발 문제**와 **락 충돌 문제**를 동시에 해결합니다.

### 4.1. @Modifying과 @Query (Repository 계층)
Spring Data JPA에서 영속성 컨텍스트를 거치지 않고 DB에 직접 벌크 업데이트 쿼리를 날릴 때 활용합니다.

```java
@Modifying
@Query("UPDATE Poi p SET p.viewCount = p.viewCount + 1 WHERE p.id = :poiId")
void increaseViewCount(@Param("poiId") Long poiId);
```
조회수나 찜수 같은 단순 카운트는 `viewCount = viewCount + 1`처럼 DB 단의 쿼리를 통해 원자적으로 증가시킵니다. 
<br>
**이 방식은 엔티티를 거치지 않으므로 쿼리에 명시하지 않는 한 `@Version`이 증가하지 않습니다.** 이렇게 의도적으로 버전을 올리지 않음으로써, 단순 조회 행위 때문에 다른 사용자의 중요한 작업(리뷰 작성 등)이 `OptimisticLockException`으로 튕기는 현상을 방지합니다.

<br>

### 4.2. @DynamicUpdate (Entity 레벨)
JPA는 기본적으로 엔티티를 수정(Dirty Checking)하여 저장할 때 **모든 컬럼을 UPDATE** 하는 쿼리를 생성합니다. 
<br>
하지만 여기서 엔티티 클래스(예: `Poi`)에 `@DynamicUpdate`를 적용하면 영속성 컨텍스트(메모리) 상에서 **실제 값이 변경된 컬럼만 찾아 그 부분만 UPDATE 쿼리의 SET 구문에 포함**시킵니다. (변경되지 않은건 제외)

*(DB의 최신 값과 비교하는 것이 아님. 엔티티를 처음 조회했을 때의 스냅샷과 현재 엔티티 상태를 비교하여 수정된 필드만 골라내는 기능입니다. 또한 변경된 컬럼과 함께 항상 `version = version + 1` 조건이 포함됨. version을 우회하는것이 아님)*


<br>

## 시나리오: 동시성 제어 및 조회수 증발 해결 

**[상황 가정]**
1. **A의 리뷰 작성 시작:** A가 데이터를 읽어옵니다. (메모리 상태: `viewCount = 1`, `rating = 0`, `version = 1`)
2. **B의 단순 조회:** B가 해당 장소를 조회하여 DB에 벌크 쿼리(`@Modifying`)가 날아갑니다. 
   - 버전을 올리지 않았기 때문에 DB 상태는 `viewCount = 2`, `version = 1`이 됩니다.
   - DB 버전이 여전히 1이므로, A가 나중에 리뷰를 등록할 때 버전 충돌(OptimisticLockException)이 발생하지 않습니다.
3. **A의 리뷰 작성 완료:** A가 애플리케이션에서 평점을 5점으로 수정하고 저장을 시도합니다. (메모리 상태: `viewCount = 1`, `rating = 5`, `version = 1`)
---
**만약 `@DynamicUpdate`가 없다면?**
<br>

JPA는 A의 메모리 상태를 기반으로 모든 컬럼을 업데이트하는 쿼리를 생성합니다.
- **실행 쿼리:** `UPDATE poi SET rating = 5, view_count = 1, version = 2 WHERE id = ? AND version = 1;`
- **결과:** DB에 잘 올라가 있던 B의 조회수(`viewCount = 2`)가 A가 과거에 읽어둔 `1`로 덮어씌워져 B의 조회수가 증발(Lost Update)해 버립니다!

---

**`@DynamicUpdate`가 적용되어 있다면?**
<br>

JPA는 A가 엔티티를 처음 읽어왔을 때의 스냅샷과 현재를 비교(Dirty Checking)해 보고 쿼리를 만듭니다.
- **JPA의 입장:** A가 처음 읽었을 땐 `viewCount`가 1이었고 지금도 1이네? 변경 안 했구나, 근데 `rating`은 0에서 5로 바뀌었네?
- **실행 쿼리:** `UPDATE poi SET rating = 5, version = 2 WHERE id = ? AND version = 1;`
- **결과:** 쿼리에 `view_count = ?` 구문 자체가 아예 빠져버리고, DB에 있던 **`viewCount = 2` 데이터가 아무런 영향을 받지 않고 안전하게 유지**됩니다.

---
### 3줄요약

조회수 증가 => 강제로 DB에 대고 벌크 쿼리날려서 조회수 증가 버전은 그대로 유지(@Modifying과 @Query)
<br>
리뷰 작성 => 리뷰를 작성후 처음 상태와 비교해서 그대로인건 업데이트 쿼리에서 제외시킴 버전은 증가(@DynamicUpdate)
<br>
결과 => 리뷰 작성를 작성하는동안의 조회수가 증발하는 문제 해결
