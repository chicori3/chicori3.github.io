---
title: 'TransactionalEventListener 헤딩일기'
categories:
    - Spring
tags:
    - Spring 
    - Transaction
    - Event
toc: true
toc_sticky: true
---

### 들어가며

제가 구현하고자 했던 문제입니다.

```
숫자야구 정답 질의
-> 게임 결과 반환 및 저장
-> 이벤트 발행
-> 게임 기록 저장
```

저는 게임 결과를 반환하고 현재 게임의 상태를 영속화하는 것과 게임 기록을 영속화하는 것을 분리하고자 했고, 이를 위해 스프링이 제공하는 이벤트를 학습 겸 시도해봤습니다.     
스프링이 제공하는 이벤트 리스너는 크게 EventListener와 TransactionalEventListener가 존재하며 이 중 TransactionalEventListener를 사용했습니다.       

이에 앞서 TransactionalEventListener를 짤막하게 보겠습니다.

### TransactionalEventListener

<img width="999" alt="image" src="https://github.com/chicori3/number_baseball_server/assets/40778768/a13c5f35-a402-461a-82bb-ca4c5448cc5c"> {: width="750" .align-center}

Baeldung의 아티클을 살펴보면 트랜잭션 단계에 따라 이벤트를 처리할 수 있는 이벤트 리스너라고 소개하고 있습니다.      
이벤트 리스너는 트랜잭션 단계에 따라 이벤트를 처리할 수 있으며, 트랜잭션 단계는 BEFORE_COMMIT, AFTER_COMMIT, AFTER_ROLLBACK, AFTER_COMPLETION이 존재합니다.     

저의 경우에도 우선 게임 상태를 조회, 수정, 반환이 필요하고 이를 @Transactional를 이용해 트랜잭션을 보장하고자 했습니다.     
또 게임 상태가 예외없이 제대로 반영되고나서 게임 기록을 저장하는 것이 맞다고 생각하여 @TransactionalEventListener를 사용했습니다.

그럼 이제 제가 직접 작성한 코드를 보겠습니다.

### 예제 코드

```java
@Slf4j
@Component
public class GameCommandManager {

    private final BaseballGameRepository baseballGameRepository;
    private final ApplicationEventPublisher publisher;

    public GameCommandManager(BaseballGameRepository baseballGameRepository, final ApplicationEventPublisher publisher) {
        this.baseballGameRepository = baseballGameRepository;
        this.publisher = publisher;
    }

    @Transactional
    public GameResult guess(GameAnswerCommand command) {
        final BaseballGame baseballGame = baseballGameRepository.findById(command.roomId());
        final GameResult result = baseballGame.guess(command.answerToList());

        publisher.publishEvent(new BaseballGameEvent(baseballGame.getId(), command.answer(), result));
        log.info("EVENT PUBLISHED: {}", Thread.currentThread().getName());

        return result;
    }
}
```

GameCommandManager는 게임을 가져와 게임을 수행한 후 결과를 이벤트를 발행하고 게임 결과를 반환합니다.

```java
@Slf4j
@Component
public class HistoryCommandManager {

    private final HistoryRepository historyRepository;

    public HistoryCommandManager(final HistoryRepository historyRepository) {
        this.historyRepository = historyRepository;
    }
    
    @Transactional
    @TransactionalEventListener
    public void saveHistory(BaseballGameEvent event) {
        log.info("EVENT RECEIVED: {}", Thread.currentThread().getName());
        
        History history = History.builder()
                .roomId(event.getId())
                .answer(event.getAnswer())
                .result(new Result(event.getStrike(), event.getBall(), event.getOut()))
                .build();

        historyRepository.save(history);
    }
}
```
 
saveHistory 메서드가 이벤트 컨슈머이며 구독한 이벤트를 받아 게임 기록을 저장합니다. 
역시 DB에 저장하기 위해 @Transactional을 사용했습니다.

```java
class eventTest {
    @Test
    @DisplayName("TransactionalEventListener만 사용했을 경우")
    void test_v1() {
        final long roomId = 1L;
        final String answer = "123";
        GameAnswerCommand gameAnswerCommand = new GameAnswerCommand(roomId, answer);

        gameCommandManager.guess(gameAnswerCommand);
        List<History> histories = historyRepository.findAllByRoomId(1L);

        assertThat(histories).hasSize(1);
    }
}
```

게임을 수행하고 게임 기록이 저장되었는지 확인하는 간단한 테스트 코드를 실행해봤습니다.

<img width="1041" alt="image" src="https://github.com/chicori3/number_baseball_server/assets/40778768/55880a66-2d3d-40ee-88cb-f30238fa5167"> {: width="400" .align-center}

테스트는 실패합니다.

```java
1. 게임 조회 후 로직 수행
2. 이벤트 발행
3. 게임 상태 커밋
4. 게임 기록 조회 -> 실패
```

로그를 살펴보면 위처럼 진행되는데 게임 기록을 왜 저장하지 못했을까요?     
분명 @Transactional을 사용했고, @TransactionalEventListener도 사용했는데요.

### 다시보자 TransactionalEventListener

위에서 잠깐 본 TransactionalEventLitener는 트랜잭션 단계에 따라 이벤트를 처리할 수 있다고 했습니다.    

```java
public enum TransactionPhase {
	BEFORE_COMMIT,
	AFTER_COMMIT,
	AFTER_ROLLBACK,
	AFTER_COMPLETION
}
```

이 중 AFTER_COMMIT이 기본값이며 커밋이 성공적으로 완료된 후 이벤트를 처리하는 설정입니다.        
때문에 위의 로그에서도 이벤트를 발행한 후 게임 상태를 커밋하고 나서 이벤트 리스너가 동작하게 됩니다.       

그런데 게임 상태를 이미 커밋한 상태에서 @Transactional을 사용하면 어떻게 될까요?    
@Transactional은 기본 전파 전략이 REQUIRED이며 이는 이미 트랜잭션이 시작된 상태에서는 트랜잭션을 새로 시작하지 않고 기존 트랜잭션에 참여한다는 의미입니다.   

위에서 게임 상태를 이미 커밋을 한 후 이벤트 리스너가 동작을 했었죠?     
근데 이미 트랜잭션은 커밋되었고, 이미 커밋이 완료된 트랜잭션에서 또 게임 기록을 커밋하려고 하니 영속화하지 못하게 됩니다.       

그럼 saveHistory 메서드는 어떻게 guess에서 생성한 트랜잭션에 참여할 수 있었을까요?

### AbstractPlatformTransactionManager

스프링의 장점 중 하나는 추상화이며 Transaction을 관리하는 TransactionManager 또한 각각의 DB 접근 기술에 맞게 추상화되어 있습니다.

<img width="1219" alt="image" src="https://github.com/chicori3/number_baseball_server/assets/40778768/2f566b3c-7ed6-42a3-b20a-6821c017cf7f"> {: width="400" .align-center}

위의 그림은 TransactionManager의 추상화를 보여주는 다이어그램입니다.  
이 중 AbstractPlatformTransactionManager는 Spring 표준 트랜잭션 처리를 구현하는 추상 기본 클래스입니다.   
AbstractPlatformTransactionManager는 트랜잭션의 시작, 커밋, 롤백, 트랜잭션 상태 확인 등의 기능을 제공하며 여기서 제가 궁금한 것을 해결할 수 있었습니다.     

```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
    ...
    private void processCommit(DefaultTransactionStatus status) throws TransactionException {
        triggerBeforeCommit(status);

        doCommit(status);

        triggerAfterCommit(status);
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);

        cleanupAfterCompletion(status);
    }
    ...
}
```

TransactionalEventListener에서 적용 가능한 TransactionPhase와 관련된 코드만 추려봤습니다.   
제가 궁금했던 것은 트랜잭션에 참여했을 때 왜 커밋이 안되는 것이었나 했는데, 위 코드를 보면 간단하게 알 수 있죠.

이미 이벤트 발행자가 먼저 트랜잭션을 열고, 커밋을 했기 때문에 당연히 이벤트 리스너는 커밋을 할 수 없는 상태입니다.  
또 트랜잭션은 커밋을 완료한 후에도 바로 닫히는 것이 아니라, afterCommit, afterCompletion, cleanupAfterCompletion 등의 추가 작업을 수행합니다.    
이런 작업들이 끝난 후에야 트랜잭션이 비로소 종료되는데 AFTER_COMMIT 시점에는 커밋은 완료되었지만 트랜잭션은 종료되지 않았기 때문에 기존의 트랜잭션에 참여할 수 있었던 것이었습니다.

### 그럼 어떻게 해결할까?

1. @EventListener
2. @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
2. @Transactional(Propagation.REQUIRES_NEW)
3. @Async

위 방법들을 사용하면 기존의 문제를 해결할 수 있습니다.     
제 경우에는 @Transactional(Propagation.REQUIRES_NEW)를 사용해서 제가 원했던 흐름대로 해결할 수 있었습니다.
이 외에 방법들은 각각 상황에 맞게 사용하면 될 것 같습니다.  
특히 @Async는 비동기와 관련되어 있으며 이는 다음 포스팅에서 알아볼 예정입니다.

사실 이번 헤딩에서 중요한 것은 이벤트를 처리하는 방법이 아니라 스프링에서 트랜잭션이 어떤 흐름으로 진행되는지 파악하는 것이었네요.   
직접 코드를 짜보고 구현체를 뜯어보면서 이해할 수 있어서 좋은 경험이었습니다.

> 참고    
> <https://www.baeldung.com/spring-events>    
> <https://dzone.com/articles/transaction-synchronization-and-spring-application>   
> <https://findstar.pe.kr/2022/09/17/points-to-consider-when-using-the-Spring-Events-feature/>