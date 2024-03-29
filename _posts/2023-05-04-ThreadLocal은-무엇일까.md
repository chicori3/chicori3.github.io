---
title: 'ThreadLocal은 무엇일까'
categories:
    - Java
tags:
    - Java
    - Concurrency
toc: true
toc_sticky: true
---

흔히 동시성 문제는 상태와 관련이 되어 있습니다.     
멀티 스레드 환경에서 상태를 가진 객체를 사용할때 주의하지 않으면 예상치 못한 결과가 발생하는 데 간단한 예제를 통해 확인해보겠습니다.

### 예제

```java
class UserStore {
    private String storedName;

    public void store(String name) {
        System.out.println("현재 이름: " + storedName);

        this.storedName = name;
        System.out.println("저장 이름: " + name);

        try {
            sleep(100);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}

```

이름을 저장하는 UserStore 클래스를 정의하고, store 메서드가 호출될 때마다 현재 저장된 이름과 저장할 이름을 출력하도록 했습니다.

```java
public class ConcurrencyTest {

    private final UserStore userStore = new UserStore();

    @Test
    void concurrencyIssueTest() throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        Thread threadA = new Thread(() -> {
            userStore.store("name1");
            countDownLatch.countDown();
        });
        Thread threadB = new Thread(() -> {
            userStore.store("name2");
            countDownLatch.countDown();
        });

        threadA.setName("threadA");
        threadB.setName("threadB");

        threadA.start();
        sleep(100);
        threadB.start();

        countDownLatch.await();
    }
}
```

스레드 A와 스레드 B가 각각 이름을 저장하는 store 메서드를 호출하도록 했습니다.    
제가 원하는 결과는 스레드 A는 name1을 저장하고 그것이 반환되어야 하며, 스레드 B는 name2를 저장하고 그것이 반환되어야 합니다.

<img width="251" src="https://user-images.githubusercontent.com/40778768/236136738-9d76fa1f-0192-4eb0-97f4-3e63d8fe837e.png">

결과를 보면 스레드 A가 이름을 저장하고 조회하는 그 사이에 스레드 B가 이름을 새로 저장했기 때문에 발생한 동시성 문제입니다.     
이 문제를 해결하는 여러 방법이 있겠지만 오늘은 ThreadLocal에 대해서 알아보려고 합니다.


### ThreadLocal로 해결해보기

일단 위에서 발생한 문제를 ThreadLocal을 사용하여 해결해보겠습니다.

```java
class ThreadLocalUserStore {
    private final ThreadLocal<String> storedName = new ThreadLocal<>();

    public String store(String name) {
        System.out.println("Thread: " + Thread.currentThread().getName() + " 현재 이름: " + storedName.get());

        storedName.set(name);
        System.out.println("Thread: " + Thread.currentThread().getName() + " 저장 이름: " + name);

        try {
            sleep(1000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        System.out.println("Thread: " + Thread.currentThread().getName() + " 조회 이름: " + storedName.get());
        return storedName.get();
    }
}
```

기존의 storedName을 ThreadLocal로 래핑하고 set과 get을 사용한 것을 제외하면 이전 예제와 같은 코드입니다.    

<img width="251" src="https://user-images.githubusercontent.com/40778768/236138117-cbfd996c-5013-4d2d-9f17-b757773fd3c2.png">

제가 원하던 결과가 반환된 모습입니다.   
ThreadLocal은 어떻게 이런 결과를 만들어낼 수 있었을까요?

### ThreadLocal 알아보기

ThreadLocal은 이름에서 유추할 수 있다시피 스레드의 지역 변수를 생성하고 관리하는 클래스입니다.      
ThreadLocal로 관리되는 변수는 스레드 간 격리되어 해당 스레드에서만 접근이 가능합니다.

ThreadLocal에서 중요한 메서드는 get과 set이며 하나씩 살펴보겠습니다.

```java
public class ThreadLocal<T> {
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
    }
}
```

set 메서드는 현재 스레드로 ThreadLocalMap을 가져오고, 맵이 존재하는 경우 해당 맵에 ThreadLocal 자신과 값을 저장합니다.     
맵이 존재하지 않는 경우에는 맵을 생성하고 this와 값을 저장하도록 되어 있습니다.     
this는 현재의 Thread를 가리키며 더 들어가보면 Thread의 ThreadLocal을 넘기게 됩니다.
여기서 갑자기 ThreadLocalMap이 나오는데 이 클래스는 대체 뭘까요?

```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    private Entry[] table;
}
```

ThreadLocalMap은 ThreadLocal 내부에 선언된 내부 정적 클래스입니다.   
Entry는 데이터를 저장하기 위한 클래스로 ThreadLocal을 키로 사용하는 맵과 같은 형태입니다.  
> WeakReference를 상속한 것을 볼 수 있는데 이는 약한 참조만으로 참조되고 있는 경우 GC의 대상이 될 수 있다는 것입니다.

중요한 것은 실제 값이 저장되는 것은 Entry라는 것이며 ThreadLocalMap은 Entry 타입의 배열을 가지고 있습니다.

```java
public class ThreadLocal<T> {
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
}
```

ThreadLocal의 get 메서드는 set과 마찬가지로 현재 스레드의 map을 가져옵니다.    
map에서는 Entry를 가져온 후 Entry에 저장된 값을 반환하게 됩니다.     
> map이나 Entry가 존재하지 않는 경우에는 setInitialValue 메서드를 호출하는데 맵이 없다면 생성하고, 맵에 null 값을 저장하고 이를 반환합니다.

### 정리를 해보자

구조를 간단하게 정리하면 다음과 같습니다.

- ThreadLocal<T>
- ThreadLocalMap<ThreadLocal, Entry>
- Entry<ThreadLocal, Object>

ThreadLocal을 사용하면 ThreadLocal 클래스 하위 클래스의 새로운 인스턴스가 생성되고 이 인스턴스는 ThreadLocalMap에 저장됩니다.     
ThreadLocalMap은 ThreadLocal 인스턴스를 키로 하고 Entry를 값으로 가지는 맵이며 Entry가 실질적인 데이터를 보관하는 객체입니다.      
또 Thread가 종료되면 ThreadLocal의 인스턴스도 모두 GC의 대상이 됩니다.

### ThreadLocal을 제대로 사용하지 않는다면

```java
class User {
    private String name;

    public User(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

ThreadLocal에 공유 객체를 넘기면 어떻게 될까요?    
테스트를 해보기 위해 작성한 간단한 User 클래스입니다.

```java
class WrongThreadLocalTest {
    @Test
    void threadLocalTest() throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        User originUser = new User("User");

        Thread a = new Thread(() -> {
            ThreadLocal<User> threadLocal = new ThreadLocal<>();
            threadLocal.set(originUser);

            User localUser = threadLocal.get();
            localUser.setName("Thread A User");
            threadLocal.set(localUser);

            try {
                sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }

            System.out.println("Thread A ThreadLocalUser: " + threadLocal.get().getName());
            System.out.println("Thread A OriginUser: " + originUser.getName());
            System.out.println(originUser.equals(localUser));

            countDownLatch.countDown();
        });

        Thread b = new Thread(() -> {
            ThreadLocal<User> threadLocal = new ThreadLocal<>();
            threadLocal.set(originUser);

            User localUser = threadLocal.get();
            localUser.setName("Thread B User");
            threadLocal.set(localUser);

            System.out.println("Thread B ThreadLocalUser: " + threadLocal.get().getName());
            System.out.println("Thread B OriginUser: " + originUser.getName());
            System.out.println(originUser.equals(localUser));

            countDownLatch.countDown();
        });

        a.start();
        sleep(100);
        b.start();

        countDownLatch.await();
    }
}
```

코드가 길어 파악이 어려울 수 있지만 하나씩 설명해보자면,

1. 공유되는 객체인 User 인스턴스인 OriginUser를 생성합니다.
2. Thread A에서 OriginUser를 ThreadLocal에 저장합니다.
3. Thread A에서 ThreadLocal에 저장된 User 인스턴스를 가져와서 이름을 변경, 저장 후 sleep을 합니다.
4. Thread B에서 OriginUser를 ThreadLocal에 저장합니다.
5. Thread B에서 ThreadLocal에 저장된 User 인스턴스를 가져와서 이름을 변경한 후 다시 저장합니다.
6. Thread A에서 ThreadLocal에 저장된 User 인스턴스와 OriginUser를 비교해보고 각각 저장된 상태를 호출합니다.
7. Thread B에서 ThreadLocal에 저장된 User 인스턴스와 OriginUser를 비교해보고 각각 저장된 상태를 호출합니다.

![image](https://user-images.githubusercontent.com/40778768/236253170-817b1f66-8d72-44b8-bb4f-63dad0e4cb08.png)

결과는 둘 다 Thread B User로 출력되었는데, Thread B에서 변경한 내용이 Thread A에도 전파된 것입니다.      
ThreadLocal에 저장되는 객체도 같은 참조값을 가지기 때문에 위와 같은 현상이 발생합니다.  
때문에 ThreadLocal은 공유되는 객체를 각각 관리하기 위해서도 아니고, 할 수도 없습니다.      
스레드 별로 독립적인 상태를 다루기 위해서 사용하는 것이기 때문에 ThreadLocal에 공유 객체를 넘기면 안됩니다.  

### 만약 스레드 풀을 사용한다면

스레드 풀은 미리 생성된 스레드를 재사용합니다.    
스레드를 매번 삭제하고 새로 생성하는 과정이 없기 때문에 스레드 풀을 사용하는 것이 성능상 유리합니다.   
하지만 스레드 풀과 ThreadLocal을 사용하면 문제가 발생할 수 있습니다.

해당 스레드가 종료되면 자동으로 ThreadLocal은 GC의 대상이 되지만, 스레드를 재사용하는 스레드 풀 환경에서는 스레드가 종료되지 않기 때문에
이전 스레드에서 사용하던 ThreadLocal 또한 재사용될 수 있기 때문입니다.

때문에 항상 스레드를 새로 시작하거나 종료할 때 ThreadLocal의 remove 메서드를 호출해서 사용한 스레드의 ThreadLocal을 초기화해주는 것이 좋습니다.

> 참고    
> <https://www.baeldung.com/java-threadlocal>       
> <https://blog.devgenius.io/a-deep-dive-into-java-threadlocal-9d0c2591a72f>    
> <https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8>        
> 자바 병렬 프로그래밍

언제든 편하게 피드백을 주세요! 감사합니다!