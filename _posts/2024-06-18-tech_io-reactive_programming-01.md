---
title: 1. Introduction to Reactive Programming
description: 컨셉과 특징
date: 2024-06-19 +0900
categories: [Study, Reactive Programming]
tags: [Reactor, Reactive Streams]
---

> [tech.io](https://tech.io/playgrounds/929/reactive-programming-with-reactor-3/Intro) 와 [백기선님 유튜브](https://www.youtube.com/watch?v=VeSHa_Xsd2U&list=PLfI752FpVCS9hh_FE8uDuRVgPPnAivZTY) 로 공부한 Reactive Programming
{: .prompt-info}

## <b>Why</b>
**Reactive Programming is**
- declarative code (similar to Functional Programming)
- event-based model : 데이터가 컨슈머에게 Publish 후 사용가능해짐
- non-blocking
  - 다른 작업이 진행중이어도 작업이 끝날때까지 기다리지 않고 남은 작업 수행(다른 작업 진행동안 현재 작업이 비활성되지 않는다)
  - 다른 작업을 할 수 있느냐 없느냐
- asynchronous
  - 요청에 대한 응답을 기다리지 않음. 콜백으로 응답 완료 후에 처리(결과를 바로 대기하지 않음)
  - 응답을 기다리느냐 기다리지 않느냐
- JDK에서 비동기프로그래밍을 위해 콜백 기반 API 나 Future 의 대안
- 논블록과 비동기를 핵심으로써, JDK에서 비동기 프로그래밍을 위해 사용하는 콜백 기반 API나 Future에 대한 대안

<br>

---

## <b>Reactive Streams</b>
- 산업(기업체)가 주도해서 정의한 JVM 표준 라이브러리
- 호환 가능 : Reactor3, RxJava, Akka Streams, Vert.x, Ratpack
- JVM 9 부터 도입
- 4개의 단순한 인터페이스로 구성

<br>

---

## <b>Interactions</b>
![주체간의 상호작용](/assets/docs/reactive/ch1_01.png){: width="500"}
*출처 : https://tech.io/playgrounds/929/reactive-programming-with-reactor-3/Intro*

- Publisher
  - 데이터 제공
  - Subscriber 가 등록되기 전까지는 데이터를 전송하지 않음 (그 전에는 아무 동작 안함)
- Subscriber
  - 데이터 수신
- feedback == `BACKPRESSURE`
  - subscriber 의 로드밸런싱
  - 주체는 Subscriber
- Operator
  - 각 단계별로 데이터 처리에 대한 체이닝
  - 한 단계의 체이닝 후 새로운 중간 Publisher 반환
    - Stream 체이닝에서 map 또는 filter 사용시 중간값을 생성한다는 의미로 이해

<br>

---

## <b>Quiz</b>
- Operator 에 대한 내용

```java
//expected: fooA
//actual : A
Flux<String> flux = Flux.just("A");
flux.map(s -> "foo" + s);
flux.subscribe(System.out::println);

//expected: fooA
//actual : fooA
Flux<String> flux = Flux.just("A");
//중간 계산값을 저장하는 flux2 필요
Flux<String> flux2 = flux.map(s -> "foo" + s);
flux2.subscribe(System.out::println);
```
