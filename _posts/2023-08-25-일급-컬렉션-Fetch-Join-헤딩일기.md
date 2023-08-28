---
title: '일급 컬렉션 Fetch Join 헤딩 일기'
categories:
    - Spring
tags:
    - JPA 
toc: true
toc_sticky: true
---

### 들어가며

오늘은 제가 Fetch Join을 사용하면서 겪었던 문제에 대해 정리해보려고 합니다.     
상황은 다음과 같습니다.   

- 장바구니 엔티티와 장바구니 아이템 엔티티가 존재합니다.
- 장바구니 엔티티는 장바구니 아이템 리스트를 `CartItems` 라는 일급 컬렉션으로 갖고 있습니다.
- `CartItems` 일급 컬렉션은 장바구니 아이템을 리스트로 가지며 `@OneToMany`가 설정되어 있습니다.
- 주문을 생성하기 위해 장바구니를 조회할 때 장바구니 아이템 리스트를 함께 조회하고자 합니다.

우선 제가 왜 일급 컬렉션을 사용했는지 알아볼까요? 

### 일급 컬렉션

소트웍스 앤솔로지라는 책에서 객체지향 생활체조를 소개하고 있습니다.       
객체지향 생활체조는 보다 객체지향적인 코드를 작성하기 위한 가이드라인으로 총 9가지의 원칙을 제시하는데 그 원칙은 아래와 같습니다.   

- 규칙 1: 한 메서드에 오직 한 단계의 들여쓰기(indent)만 한다.
- 규칙 2: else 예약어를 쓰지 않는다.
- 규칙 3: 모든 원시값과 문자열을 포장한다.
- 규칙 4: 한 줄에 점을 하나만 찍는다.
- 규칙 5: 줄여쓰지 않는다(축약 금지).
- 규칙 6: 모든 엔티티를 작게 유지한다.
- 규칙 7: 3개 이상의 인스턴스 변수를 가진 클래스를 쓰지 않는다.
- 규칙 8: 일급 콜렉션을 쓴다.
- 규칙 9: 게터/세터/프로퍼티를 쓰지 않는다.

다른 책이나 강의 또는 어디선가라도 몇 가지 원칙은 들어보셨을 것 같습니다.
이 중 규칙 8번인 일급 컬렉션을 쓴다라는 내용이 있는데, 왜 일급 컬렉션을 쓰라고 할까요?     

```java
public class BaseballGame {
    public List<Integer> ballNumbers = new ArrayList<>();

    public BaseballGame(final List<Integer> ballNumbers) {
        validateSize(ballNumbers);
        validateDuplicate(ballNumbers);
        this.ballNumbers = ballNumbers;
    }

    private void validateSize(final List<Integer> ballNumbers) {
        if (ballNumbers.size() != 3) {
            throw new IllegalArgumentException("숫자는 3개만 입력 가능합니다.");
        }
    }

    private void validateDuplicate(final List<Integer> ballNumbers) {
        if (ballNumbers.stream().distinct().count() != 3) {
            throw new IllegalArgumentException("중복된 숫자는 입력할 수 없습니다.");
        }
    }
    
    ...
}
```

숫자야구 게임을 예제로 작성해봤습니다.   
숫자야구 게임은 아래와 같은 규칙이 있습니다.

- 0 ~ 9까지의 숫자를 입력할 수 있습니다.
- 숫자는 3개를 입력해야 합니다.
- 중복된 숫자는 입력할 수 없습니다.

위와 같은 조건이 있기 때문에 검증 로직이 필요한데, 이를 `BaseballGame` 객체가 처리하고 있습니다.      

해당 검증 로직을 `BaseballGame`이라는 객체가 처리하는 것이 적절할까요?         
물론 지금 상황에서는 `BaseballGame`이라는 객체가 처리하는 것도 나쁘지 않다고 생각합니다.    
다만 저 `List<Integer>`가 다른 곳에서도 사용된다면 똑같은 검증 로직을 다른 객체에서도 사용해야 할 것이며 중복된 코드가 발생할 수 있고, `List<Integer>`가 무엇을 위한 값인지 파악하기도 쉽지 않을 것입니다.  

저는 객체지향의 핵심은 메시징과 캡슐화, 동적 바인딩이라는 앨런 케이의 말을 따라 가능한 실천하려고 합니다.          
그러기 위해서는 객체가 메시지를 받을 수 있어야 하고, 자신의 상태는 온전히 자신만이 관리할 수 있어야 합니다.
그럼 `List<Integer>`처럼 단순 컬렉션이 아닌 고유한 메서드를 가지고 메시지를 수신해 본인의 상태를 처리할 수 있는 자료구조가 필요한데 이 때 사용할 수 있는 것이 일급 컬렉션입니다.

```java
public class BaseballNumbers {
    public List<Integer> ballNumbers = new ArrayList<>();

    public BaseballNumbers(final List<Integer> ballNumbers) {
        validateSize(ballNumbers);
        validateDuplicate(ballNumbers);
        this.ballNumbers = ballNumbers;
    }

    private void validateSize(final List<Integer> ballNumbers) {
        if (ballNumbers.size() != 3) {
            throw new IllegalArgumentException("숫자는 3개만 입력 가능합니다.");
        }
    }

    private void validateDuplicate(final List<Integer> ballNumbers) {
        if (ballNumbers.stream().distinct().count() != 3) {
            throw new IllegalArgumentException("중복된 숫자는 입력할 수 없습니다.");
        }
    }
}
```

일급 컬렉션은 위와 같이 컬렉션을 래핑하는 객체입니다.      
거창한 이름과는 다르게 작성하기도 쉽고 다양한 장점이 있습니다.     

- 상태와 행위를 하나의 객체가 관리할 수 있습니다.
- 재사용이 가능합니다.
- 이름을 가질 수 있습니다.
- 불변성을 보장하도록 작성할 수 있습니다.

위와 같은 이유들로 저는 일급 컬렉션을 사용했습니다.       
그럼 이제부터 아래 프로젝트의 예제 코드를 보며 어떤 문제가 발생했고, 어떻게 해결했는지 알아보겠습니다.  

> 일급 컬렉션에 대해 더 알고 싶으시다면 [향로님의 블로그](https://jojoldu.tistory.com/412)를 참고해주세요.

### 예제 코드

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Cart {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long ownerId;
    @Embedded
    private CartItems cartItems = new CartItems();
    private BigDecimal totalPrice;

    ...
}

@Embeddable
public class CartItems {

    @OneToMany(mappedBy = "cart", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<CartItem> cartItems = new ArrayList<>();
    
    ...
}

@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class CartItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long lectureId;
    private BigDecimal price;
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cart_id")
    private Cart cart;

    ...
}

@Component
public class CreateOrderProcessor {

    ...
    
    private void verifyCart(final Long ownerId, final List<Long> lectureIds) {
        final Cart cart = cartRepository.findCartByOwnerId(ownerId);

        cart.hasCartItem(lectureIds);
    }
}
```

위 객체들은 앞서 말씀드린 장바구니, 장바구니 아이템 엔티티와 일급 컬렉션입니다.   
그리고 장바구니를 조회한 후 주문한 강의가 장바구니에 있는지 확인하는 `verifyCart` 메서드가 존재합니다.

우선 이 코드들을 수행한 후에 발생한 쿼리를 살펴볼까요?

<p align="center">
    <img width="600" src="https://github.com/f-lab-edu/infrun/assets/40778768/988ae156-18dd-4597-9f6b-bcf422557d29"/>
</p>

JPA를 공부해보셨다면 다들 아시다시피 `Cart` 엔티티를 조회하는 데 한 번, `CartItems`를 조회하는 데 한 번씩 총 두 번의 쿼리가 발생했습니다.   
일반적인 상황에서 말하는 N+1 문제는 아니지만 불필요한 쿼리가 발생하는 것은 마찬가지라 저는 여기서 `Cart` 엔티티를 조회할 때 `CartItems`를 한 번에 조인해서 가져오고 싶었습니다.     

JPA에서는 연관관계에 있는 엔티티를 함께 조회하는 방법을 제공하는데 이를 시도해봤습니다.

### EntityGraph

우선 `EntityGraph`는 JPA가 제공하는 기능으로 엔티티를 조회하는 시점에 연관된 엔티티들을 함께 조회할 수 있습니다.  
`@NamedEntityGraph` 어노테이션을 사용하면 엔티티 객체에 설정을 작성해야 하는데, 저는 가져오는 방식을 리포지토리에서 관리하기 위해  `@EntityGraph` 어노테이션을 사용하는 동적인 방식을 사용했습니다.

```java
public interface CartJpaRepository extends JpaRepository<Cart, Long> {

    @EntityGraph(attributePaths = {"cartItems"})
    Optional<Cart> findByOwnerId(final Long ownerId);
}
```

`CartJpaRepository` 내부에 존재하는 조회용 쿼리에 `@EntityGraph` 어노테이션을, `attributePaths`에는 가져오고 싶은 엔티티의 필드명을 작성하면 됩니다.    

<p align="center">
    <img width="600" src="https://github.com/f-lab-edu/infrun/assets/40778768/5b46eaa9-d957-41b9-83d7-2f3053051007"/>
</p>

아까와 같은 이미지가 아닙니다...😭    
`EntityGraph`를 적용했지만 여전히 `CartItems`를 조회하는 쿼리가 발생했습니다.

### Fetch Join

Fetch Join은 JPQL이 제공하는 특수한 조인 기능입니다.    
`@EntityGraph`와 마찬가지로 엔티티를 조회할 때 연관된 엔티티를 함께 조회할 수 있습니다.

```java
public interface CartJpaRepository extends JpaRepository<Cart, Long> {

    @Query("select distinct c from Cart c join fetch c.cartItems where c.ownerId = :ownerId")
    Optional<Cart> findByOwnerId(@Param("ownerId") final Long ownerId);
}
```

`em.createQuery`로 직접 작성할 수도 있지만, JPA에서 제공하는 `@Query` 어노테이션으로도 JPQL을 작성할 수 있습니다.     
그럼 역시 쿼리를 한 번 살펴볼까요?

<p align="center">
    <img width="600" src="https://github.com/f-lab-edu/infrun/assets/40778768/72335e09-6705-419e-942a-adea39e7ca5d"/>
</p>

이번에도 같은 이미지가 아니라 같은 결과가 나왔습니다.  
여기서 고민을 많이 했는데 원인을 잘 모르겠다보니 좀 무식한 방법을 사용해봤습니다.

### 해결해보자

처음부터 다시 작성해보는 것이었는데요.   
일단 일급 컬렉션을 제거하고 엔티티에서 `List` 타입으로 일대다 관계를 갖도록 바꿨습니다.

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Cart {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long ownerId;
    @OneToMany(mappedBy = "cart", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<CartItem> cartItems = new ArrayList<>();
    private BigDecimal totalPrice;

    ...
}
```

그리고 `@EntityGraph`와 Fetch Join을 사용하여 쿼리를 다시 확인해봤습니다.    

<p align="center">
    <img width="200" src="https://github.com/f-lab-edu/infrun/assets/40778768/076f4ea8-d72f-4cfb-9f36-11a48d77f913"/>
    <img width="200" src="https://github.com/f-lab-edu/infrun/assets/40778768/c62fa720-e157-4696-8816-4cfff6fcae00"/>
</p>

각각 `@EntityGraph`와 Fetch Join을 사용한 쿼리입니다.   
일급 컬렉션을 사용하지 않고 엔티티에서 `List` 타입으로 일대다 관계를 갖도록 바꾸니 원하는 대로 쿼리가 발생했습니다.    
그래서 다시 일급 컬렉션을 적용하고 Fetch Join 쿼리를 새로 작성해봤습니다.  

```java
public interface CartJpaRepository extends JpaRepository<Cart, Long> {

    @Query("select distinct c from Cart c join fetch c.cartItems.cartItems where c.ownerId = :ownerId")
    Optional<Cart> findByOwnerId(@Param("ownerId") final Long ownerId);
}
```

아까와 다른 점이 있다면 `c.cartItems.cartItems`로 작성했다는 점입니다.  

<p align="center">
    <img width="200" src="https://github.com/f-lab-edu/infrun/assets/40778768/3be18bf7-06cf-4dbf-bf30-853022b370d6"/>
</p>

네 적용이 됐습니다! 
Fetch Join을 `@Embddable`이 적용된 일급 컬렉션에 적용하려면 현재 조회하려는 엔티티를 기준으로 조회하려는 연관관계를 작성해야 합니다.    
조회하려 했던 것은 `CartItems` 내부에 `@OneToMany`가 적용된 `List<CartItem>`이기 때문에 `c.cartItems.cartItems`로 작성해야 조인이 수행되는 것입니다.        

다만 `@EntityGraph`의 경우에는 `@EntityGraph(attributePaths = {"cartItems.cartItems"})`로 작성해도 이전과 같이 두 번 쿼리가 발생합니다.    
`@EntityGraph`는 현재 엔티티에서 이름처럼 엔티티 그래프를 이용해 탐색을 하는데, `CartItems`는 엔티티가 아닌 객체이기 때문에 엔티티 그래프를 이용해 탐색할 수 없습니다.

### 마무리

이번에는 연관된 엔티티를 한 번에 조회하기 위해 겪은 경험을 작성해봤습니다.  
저의 경우에는 `Embeddable`이 적용된 일급 컬렉션을 사용했기 때문에 `@EntityGraph`를 사용할 수 없었고, Fetch Join을 사용하기 위해서는 엔티티가 가진 일급 컬렉션 내부의 연관관계를 가진 컬렉션 기준으로 JPQL 조인문을 작성해야 했습니다.   

저와 같이 일급 컬렉션을 사용하면서 지금같은 문제에 부딪히시는 분들이 계시다면 이번 글이 도움이 되었으면 좋겠습니다.   

잘못된 정보나 피드백이 있다면 편하게 댓글로 남겨주세요. 감사합니다! 😇

> 참고    
> <https://jojoldu.tistory.com/412>     
> <https://www.baeldung.com/jpa-entity-graph>   
> 자바 ORM 표준 JPA 프로그래밍