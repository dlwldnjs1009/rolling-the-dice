# 주사위 굴리기 연습
요구사항
사람은 주사위를 굴릴 수 있다.
굴려진 주사위는 1~6 사이 무작위 정수를 반환한다.
주사위는 굴릴 때마다 다른 수가 나와야 한다.
주사위를 굴린 결과를 출력할 수 있다.

기능
~~주사위 던지기 기능~~
주사위 다시 던지기 기능
새로운 주사위 던지기 기능

추가 구현해볼 기능
스레드 풀을 이용해서 각 스레드마다 요청이 있을 때마다 스레드에서 주사위를 던지도록 구현해보자
비동기도 활용해보자 CompletableFuture
비동기로 주사위 굴리다 에러나는 경우 exceptionally(에러핸들링)도 고려해야 한다.

최종적으로는 
1. 주사위 굴리기
2. 비동기로 실행
3. 실행완료되면 콜백함수로 "주사위 몇번 실행완료" 출력
4. 주사위 굴린 결과 출력

주사위를 굴린다
사람이 주사위를 굴린다 Person class
사람은 주사위를 굴리는 메소드가 있어야 함
랜덤값 생성시키는 클래스, RandomUtil

Dice class, 주사위 클래스에는 숫자를 반환해주는, 사람에게 숫자를 알려주는 메소드가 있어야 한다.(객체 스스로 일을 한다)
Dice class 안에는 1~6까지의 숫자 중 하나를 랜덤하게 반환해주는 메소드가 있어야 함.

위처럼 비동기, 스레드풀, Completable을 선택한 이유:
공부를 하며 비동기 vs 동기, 논블락킹 vs 블락킹이 이해가 제대로 안됐다고 느껴서 이번 기회에 실습해보며 익혀보려고 선택했습니다.
스레드풀 같은 경우는 스프링부트를 공부하며 동작방식에서 queue를 이용해서 비동기로 동작한다는 것을 알게되었고
스레드 관리감독을 알아서 해 컴퓨터 리소스를 효율적으로 사용하게 해준다고 알게되어 사용해보려고 선택했습니다.
CompletableFuture는 면접 준비할 때 비동기 공부하면서 실습했던 내용인데 이번기회에 다시 복습하고 실습하려고 선택했습니다.

동기 방식이라면 한사람이 주사위를 던지고 그 결과가 나올때까지 기다리고, 결과를 보고 다시 던지고 하는 방식이라 할 수 있습니다.
비동기 방식이라면 한사람이 주사위를 스레드풀의 최대 스레드 개수만큼 박스에 가지고 있고 주사위를 손에서 던져서 굴리고 다시 박스에서 주사위를 꺼내서 주사위를 던질 수 있다고 상황을 설정했습니다.

# 1초 동안 Executor에 제출된 작업 수 측정 실험 문서

## 개요
이 실험은 1초 동안 Executor에 할당(submit)된 주사위 굴리기 작업의 총 횟수를 측정하는 것이다. 작업이 1초 이후에 완료되더라도, 제출 시점에 이미 할당된 작업으로 카운트된다.

## 실험 목적 및 주의 사항
실험 목적은 1초 동안 Executor에 제출된(할당된) 주사위 굴리기 작업의 총 횟수를 측정하는 것이다.  
주의 사항은 다음과 같다.
- 카운트 대상은 1초 동안 Executor에 할당(submit)된 작업 수이다.
- 작업이 1초 이후에 완료되더라도 카운트에 포함된다.

## 실험 환경
실행 환경은 M1 Mac Pro이다.  
실험에 사용된 Executor 종류는 다음과 같다.
- SingleThreadExecutor
- FixedThreadPool (10 threads)
- CachedThreadPool

## 실험 방법
실험은 다음과 같이 진행된다.
1. 시작 시각을 기록하고, 1초 후의 마감 시각(deadline)을 설정한다.
2. while 루프를 통해 1초 동안 `Person.rollTheDice()`를 호출하여 작업을 제출한다.  
   작업 제출 시마다 `AtomicInteger`를 증가시켜 제출된 작업 수를 기록한다.  
   제출된 작업은 `futures` 리스트에 저장하여, 후속에 모든 작업의 완료를 대기할 수 있도록 한다.
3. `CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join()`를 사용하여 1초 이후에도 완료되는 모든 작업이 종료될 때까지 대기한다.
4. 최종적으로 1초 동안 제출된 작업 수(AtomicInteger 값)를 출력한다.

## 예상 가설
예상 가설은 다음과 같다.
- SingleThreadExecutor는 작업 제출 속도가 상대적으로 낮아 총 작업 횟수가 적을 것으로 예상된다.
- FixedThreadPool (10 threads)는 다중 스레드를 사용하므로 SingleThreadExecutor보다 작업 제출 속도가 높아 더 많은 작업이 할당될 것으로 예상된다.
- CachedThreadPool은 스레드 생성 제한이 없으므로 가장 많은 작업이 할당될 것으로 예상된다.

## 실험 결과
실험 결과는 다음과 같다.
- SingleThreadExecutor: 4,220,576회
- FixedThreadPool (10 threads): 4,187,157회
- CachedThreadPool: 71,687회

## 결과 분석 및 결론
실험 결과는 각 Executor가 내부적으로 사용하는 작업 큐의 특성에 따라 제출 속도와 작업 할당 횟수가 크게 달라짐을 보여준다.

<img width="881" alt="3" src="https://github.com/user-attachments/assets/d7269e4a-5a11-4fed-a8b2-48b7cdd766e7" />
<img width="881" alt="1" src="https://github.com/user-attachments/assets/83d96585-ebc1-4941-9909-84230a4107aa" />
<img width="881" alt="2" src="https://github.com/user-attachments/assets/7fc35c4b-4d8f-4b9c-8f10-db2ea2557e2b" />

SingleThreadExecutor와 FixedThreadPool은 내부적으로 **LinkedBlockingQueue**를 사용한다.  
LinkedBlockingQueue는 기본적으로 용량이 매우 크거나 무제한에 가까워 작업 제출 시 블로킹 없이 빠르게 작업을 수용한다.  
따라서 메인 스레드가 1초 동안 약 4백만 건 이상의 작업을 빠르게 제출할 수 있었다.

<img width="881" alt="4" src="https://github.com/user-attachments/assets/9e016725-a1b2-4732-abe4-0a676cd97210" />
반면, CachedThreadPool은 내부적으로 **SynchronousQueue**를 사용한다.  
SynchronousQueue는 내부 버퍼가 없으므로, 작업 제출 시 즉시 다른 스레드가 작업을 받아야 한다.  
만약 워커 스레드가 즉시 작업을 수신하지 않으면 제출하는 스레드가 블로킹되어 작업 제출 속도가 크게 떨어진다.  
이로 인해 CachedThreadPool에서는 1초 동안 약 7만 건 정도의 작업만 제출되었다.

예상 가설에서는 CachedThreadPool이 스레드 생성 제한이 없으므로 가장 많은 작업이 할당될 것으로 보았으나, 실제로는 SynchronousQueue의 핸드오프 특성 때문에 제출 시 블로킹이 발생하여 작업 제출 속도가 크게 낮아진 것으로 나타났다.  
SingleThreadExecutor와 FixedThreadPool은 모두 LinkedBlockingQueue를 사용하므로, 매우 높은 제출 속도를 보이며 결과가 유사하게 나타났다.

결론적으로, 내부 큐의 특성에 따라 Executor의 작업 제출 속도와 할당 횟수가 크게 달라짐을 확인하였다.  
LinkedBlockingQueue를 사용하는 경우 1초 동안 약 4백만 건의 작업이 빠르게 할당되고,  
SynchronousQueue를 사용하는 경우 작업 제출 시 즉시 핸드오프가 요구되어 블로킹이 발생, 1초 동안 약 7만 건의 작업만 제출됨을 확인하였다.  
따라서 Executor 선택 시 내부 큐의 특성을 고려하는 것이 매우 중요하다.

참고:
https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/LinkedBlockingDeque.html
https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/SynchronousQueue.html
