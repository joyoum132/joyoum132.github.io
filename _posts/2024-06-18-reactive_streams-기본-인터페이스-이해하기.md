---
title: Reactive Streams Interface
description:
date: 2024-06-18 +0900
categories: [Workspace]
tags: [Webflux,SpringBoot,Reactor, Reactive Streams]
---

"Non Block", "Backpressure"
- Subscriber 가 핸들링 가능한 수준의 양만 요청해서 처리
- 어떤 이벤트나 상황이 발생했을 때, 그에 반응해서 Non-blocking 으로 작업을 처리하는 동작 매커니즘을 “Reactive”한 시스템이라고 함

## <b>Reactive Streams </b>
- 논블록 백프레셔를 이용 비동기 데이터 처리를 위한 표준
- 구현체로는 ReactX, Reactor
  ```java
  public interface Publisher<T> {
     public void subscribe(Subscriber<? super T> s);
  }
  
  public interface Subscriber<T> {
  public void onSubscribe(Subscription s);
  public void onNext(T t);
  public void onError(Throwable t);
  public void onComplete();
  }
  
  public interface Subscription {
  public void request(long n);
  public void cancel();
  }
  
  public interface Processor<T,R> extends Subscriber<T>, Publisher<R> {
  }
  ```
- Subsriber 가 subsribe()를 통해 Publisher에게 구독 요청
- Publisher 가 onSubsribe()를 통해 Subsriber 에게 Subsription 을 전달
  - Subscription : Subsriber ↔ Publisher 통신 매개체로 둘은 직접 통신하지 않음 
- Publisher 가 Subsription 을 통해 데이터의 전송, 성공 및 실패 처리 시그널 구현


## <b>코드로 이해하기</b>
```java
import org.reactivestreams.Publisher;
import org.reactivestreams.Subscriber;
import org.reactivestreams.Subscription;

class Scratch {
  public static void main(String[] args) {
    Publisher<Integer> publisher = new Publisher<Integer>() {
      //Subscriber 는 subscribe 함수로 구독을 신청할 수 있고,
      @Override
      public void subscribe(Subscriber<? super Integer> s) {
        //publisher 를 구독하면 Subscription 을 전달받을 수 있다.
        s.onSubscribe(new Subscription() {
          //1. 배압조적을 위한 request 함수
          @Override
          public void request(long n) {
            System.out.println("n 개만큼 데이터 요청");
          }

          //2. 구독 취소 함수
          @Override
          public void cancel() {
            System.out.println("구독 취소");
          }
        });
      }
    };
  }

  Subscriber<Integer> subscriber = new Subscriber<>() {

    // Publisher 로부터 Subscription 을 전달받음
    // Publisher 와 직접적으로 컨택하지 않고 Subscription 로부터 데이터를 받아서 처리
    @Override
    public void onSubscribe(Subscription s) {

      // 처리 가능한 수준의 데이터를 요청함
      s.request(10);

      //구독 취소 요청
      s.cancel();
    }

    @Override
    public void onNext(Integer integer) {
      // request() 로 데이터를 전달받아서 subscriber 에서 처리
    }

    @Override
    public void onError(Throwable t) {
      // 에러 발생 → 이후 Publisher 에게 데이터 요청 불가능
    }

    @Override
    public void onComplete() {
      // 에러 발생 → 이후 Publisher 에게 데이터 요청 불가능
    }
  };
}
```
