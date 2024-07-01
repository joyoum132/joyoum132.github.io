---
title: 2-3. Learn how to create Flux, Mono instances
description: Flux 와 Mono 의 이해
date: 2024-07-01 +0900
categories: [Study, Reactive Programming]
tags: [Reactor, Reactive Streams, Flux, Mono]
---

> [tech.io](https://tech.io/playgrounds/929/reactive-programming-with-reactor-3/Intro) 와 [백기선님 유튜브](https://www.youtube.com/watch?v=VeSHa_Xsd2U&list=PLfI752FpVCS9hh_FE8uDuRVgPPnAivZTY) 로 공부한 Reactive Programming
{: .prompt-info}

## <b> Flux </b>
- 0 ~ n 개의 요소를 내보낸 후 완료, 에러, 종료를 발생
- terminal event 가 없으면 Flux 는 무한히 흐른다 (구독해야 결과를 반환)

## <b> Mono </b>
- 0 ~ 1 개의 요소를 내보낸 후 완료, 에러, 종료를 발생
- terminal event 가 없으면 Mono 는 무한히 흐른다 (구독해야 결과를 반환)

> Flux 와 Mono 의 내부 동작을 확인하고 싶다면 반환 전에 .log() 를 사용해볼 것
{: .prompt-tip}

## <b>Quiz</b>

- Flux 사용 예제

```java
Flux<String> emptyFlux() {
  return Flux.empty();
}

//just 의 인자는 varargs
Flux<String> fooBarFluxFromValues() {
  return Flux.just("foo", "bar");
}

// 배열을 넣어주는 것 가능
Flux<String> fooBarFluxFromList() {
  return Flux.fromIterable(Arrays.asList("foo", "bar"));
}

//try-catch 대신 error 신호를 흘려보냄
//이후 stream 흐름 멈춤
Flux<String> errorFlux() {
  return Flux.error(new IllegalStateException());
}

//interval 은 지정 시간마다 데이터를 흘려보내지먄 제한이 없음. 원하는 개수만큼 pick
Flux<Long> counter() {
  return Flux.interval(Duration.ofMillis(1000)).take(10);
}

Mono<String> emptyMono() {
  return Mono.empty();
}
```

<br>

- Mono 사용 예제

```java
Mono<String> emptyMono() {
  return Mono.empty();
}

//onComplete 도 리턴하지 않음 -> 언제쓰려나?
//완료되지 않는 Publisher 임. 구독자에게 신호를 보내지도, 값을 발생시키지도 않음 -> 무한 대기
// 언제사용?
// - timeout 처리, 테스트(이 경우도 타임아웃 관련), 특정 조건에서 Mono 가 완료되지 않도록 하는 경우에 사용함/
Mono<String> monoWithNoSignal() {
  return Mono.never();
}

Mono<String> fooMono() {
  return Mono.just("foo");
}

Mono<String> errorMono() {
  return Mono.error(new IllegalStateException());
}
```

- Mono.never() 는 언제 사용하는가? 
  - [stackOverFlow](https://stackoverflow.com/questions/48273301/when-to-use-mono-never)
