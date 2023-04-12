---
title: 'Comparator를 람다로 사용했을 때'
categories:
    - Java
tags:
    - Java
    - Functional
toc: true
toc_sticky: true
---

자바의 Comparator는 두 객체를 비교하는 데 사용되는 인터페이스입니다.      
자바 8부터 지원되는 함수형 프로그래밍 방식을 이용하면 이를 람다식과 메서드 참조로 편하게 사용할 수 있는데 여기서 발생했던 문제에 대해 정리해 보겠습니다.

### 예제

```java
class Member {
    private String name;
    private int age;

    public Member(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }
}

public class MemberSortingTest {

    List<Member> members = new ArrayList<>();

    @Before
    public void setUp() {
        members.add(new Member("A", 20));
        members.add(new Member("B", 10));
        members.add(new Member("C", 30));
    }
}
```

예제는 간단하게 이름과 나이가 있는 Member 클래스를 정의하고, 이를 List에 담아서 정렬하는 테스트입니다. 

### Comparator를 람다로 사용했을 때

```java
@Test
public void comparingWithLambda() {
    List<Member> sortedList = members.stream()
            .sorted(Comparator.comparing(member -> member.getAge()))
            .collect(Collectors.toList());

    assertThat(sortedList.get(0).getName()).isEqualTo("B"); // true
    assertThat(sortedList.get(2).getName()).isEqualTo("C"); // true
}
```

Member는 Comparable 인터페이스를 구현하지 않았기 때문에, List의 sort 메서드를 사용하면 컴파일 에러가 발생합니다.              
하지만 자바 8부터 지원되는 default 메서드와 함수형 인터페이스의 힘으로 Comparator를 이용하여 쉽게 정렬이 가능합니다.      
위의 테스트는 나이를 기준으로 오름차순 정렬이 되며 테스트가 통과된 것을 확인할 수 있습니다.        

### Comparator를 메서드 참조로 사용했을 때

```java
@Test
public void comparingWithMethodReference() {
    List<Member> sortedList = members.stream()
            .sorted(Comparator.comparing(Member::getAge))
            .collect(Collectors.toList());

    assertThat(sortedList.get(0).getName()).isEqualTo("B");
    assertThat(sortedList.get(2).getName()).isEqualTo("C");
}
```

자바 8에서는 메서드 참조를 제공하는데 람다 식에서 단 하나의 메서드를 호출하는 경우 불필요한 매개변수를 제거하고 간결하게 표현할 수 있도록 해줍니다.    
결국 람다 식과 같은 기능을 하기 때문에 위의 테스트도 통과합니다.       

### Comparator의 comparing()과 reverse()를 호출했을 때 문제

```java
@Test
public void comparingAndReverse() {
    List<Member> lambda = members.stream()
            .sorted(Comparator.comparing(member -> member.getAge()).reversed()) // 컴파일 에러
            .collect(Collectors.toList());

    List<Member> methodReference = members.stream()
            .sorted(Comparator.comparing(Member::getAge).reversed())
            .collect(Collectors.toList());
}
```

위와 같은 코드를 작성했을 때 메서드 참조를 이용한 정렬 후 역정렬은 아무런 이상이 없지만 람다 식을 이용한 경우에는 컴파일 에러가 발생합니다.          
Object 클래스의 getAge 메서드를 호출할 수 없다는 `Cannot resolve method 'getAge' in 'Object'` 메시지를 보여줍니다.   

reversed()의 반환 타입을 확인해 보면

<img width="1025" alt="Object" src="https://user-images.githubusercontent.com/40778768/231411327-563e0c53-2f44-44a8-a7c3-fe900fb31a9d.png">
<img width="937" alt="Member" src="https://user-images.githubusercontent.com/40778768/231411528-fc31ef94-840a-4f5f-878e-6dce1a7eb81c.png">

위처럼 제네릭 타입이 Member가 아닌 Object로 추론되어 있습니다.   
스택 오버플로에서 검색해 본 결과로는 자바 컴파일러 타입 추론의 약점이라고 하는데, 원인은 저도 잘 모르겠고 댓글 다신 분도 잘 모르겠다고 하시네요... (아신다면 키워드라도 알려주세요..)              

이를 통해 알 수 있는 것은 메서드 참조는 타입 추론에 도움이 되는 무언가를 전달하지만 매개변수 타입을 지정하지 않은 람다 식의 경우에는 그렇지 않다는 것입니다.

### 해결 방법

```java
@Test
public void comparingAndReverse() {
    List<Member> sortedList = members.stream()
            .sorted(Comparator.comparing((Member member) -> member.getAge()).reversed())
            .collect(Collectors.toList());

    assertThat(sortedList.get(2).getName()).isEqualTo("B"); // true
    assertThat(sortedList.get(0).getName()).isEqualTo("C"); // true
}
```

해결은 아주 쉬운데요, 컴파일러가 타입을 쉽게 파악할 수 있도록 람다 식의 매개변수에 타입을 지정해 주면 됩니다.     

### 추가

해당 스레드의 댓글에는 이를 위한 추가 설명이 있었습니다.    

<img width="657" alt="answer" src="https://user-images.githubusercontent.com/40778768/231420505-dfd8e24c-f6e8-4ad4-8aec-91e1390cae06.png">

람다 식은 매개변수 타입이 지정된 명시적인 경우와 지정되지 않은 암시적인 경우로 나뉘고, 메서드 참조는 오버로딩이 없는 정확한 경우와 오버로딩이 있는 부정확한 경우로 나뉩니다.      
제네릭 메서드를 호출할 때 람다 인수가 있는 경우에 타입 매개변수가 다른 인수에서 추론되지 않는 경우가 발생할 수 있습니다. `comparing() -> reversed()`      
이런 경우에 명시적으로 타입을 제공해 주면 컴파일러가 타입을 추론할 수 있습니다.