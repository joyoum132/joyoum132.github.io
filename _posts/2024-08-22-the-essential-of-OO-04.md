---
title: 04. 역할, 책임, 협력
description: 
date: 2024-08-22 +0900
categories: [Study, 객체지향의 사실과 오해]
tags: [Team Study]
---

## <b>협력</b>
- 다수의 요청과 응답으로 구성
- 요청을 받은 객체는, 요청을 수행하기 위한 행동과 책임이 있음을 의미 

---
## <b>책임</b>
- 객체에 의해 정의되는 응집도 있는 행위의 집합
  - 알아야 하는 정보
  - 수행해야 할 행위
- 책임을 실행하기 위해서는 다른 객체로부터의 요청을 수신해야 함

---
## <b>역할</b>
- 책임에 대한 정의(이름)
- 역할이 같다는 것은 동일한 행위(책임) 을 수행한다는 것을 의미
  - 동일한 행위를 할 수 있다면 객체는 대체 가능하다 → 대체가능성 (재사용성을 높임)
- 역할을 통해 협력에 필요한 책임을 추상화, 단순화하여 표현

---
## <b>객체에서 중요한 것은 객체가 갖는 행동이다</b>
- 협력의 관점에서 객체의 행동을 설계하고
- 협력 안에서 어떤 책임과 역할을 수행할 것인지를 정의

---
## <b>객체지향 설계 기법</b>
### [ 책임 주도 설계 ]

### [ 디자인 패턴 ]
- 반복적으로 발생하는 문제와, 그 문제에 대한 해결책을 의미
- 책임 주도 설계의 결과물
  - 패턴마다 역할, 책임, 협력이 이미 정의되어있음

### [ 테스트 주도 개발 ]
- 책임을 수행할 객체가 메시지를 수신할 때 어떤 결과를 반환하고, 어떤 객체와 협력할지에 대한 기대(예상) 코드로 작성하며 설계하는 기법
- 책임 주도 설계의 기본적인 원칙을 지키면서 테스트를 통해 검증해가는 설계 기법
  




