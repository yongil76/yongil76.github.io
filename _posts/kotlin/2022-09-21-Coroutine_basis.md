---
title:  "Coroutine basis"

excerpt: "coroutine basis"

categories:
- kotlin

tags:
- coroutine

toc: true

toc_sticky: true

toc_label: "kafka"

last_modified_at: 2022-09-21T09:59:00-05:00

---

## Coroutine?

### Wiki
> 코루틴(coroutine)은 루틴의 일종으로서, 협동 루틴이라 할 수 있다(코루틴의 "Co"는 with 또는 together를 뜻한다). 
> 상호 연계 프로그램을 일컫는다고도 표현가능하다. 루틴과 서브 루틴은 서로 비대칭적인 관계이지만, 코루틴들은 완전히 대칭적인, 
> 즉 서로가 서로를 호출하는 관계이다. 코루틴들에서는 무엇이 무엇의 서브루틴인지를 구분하는 것이 불가능하다. 
> 코루틴 A와 B가 있다고 할 때, A를 프로그래밍 할 때는 B를 A의 서브루틴으로 생각한다. 그러나 B를 프로그래밍할 때는 A가 B의 서브루틴이라고 생각한다. 
> 어떠한 코루틴이 발동될 때마다 해당 코루틴은 이전에 자신의 실행이 마지막으로 중단되었던 지점 다음의 장소에서 실행을 재개(Resume)한다.

### Andorid
> A coroutine is a concurrency design pattern that you can use on Android to simplify code that executes asynchronously. 
> Coroutines were added to Kotlin in version 1.3 and are based on established concepts from other languages.

- 안드로이드라는 특정한 환경(Memory, CPU, 여러 Application)에서 동시성에 대한 제어가 쉽지 않은데, Coroutine은 그걸 쉽게 해주는 디자인 패턴 


## Why corutine?

![](/assets/images/corutine/dream_code_corutine1.png)
- 동기 코드

![](/assets/images/corutine/dream_code_corutine2.png)
- 콜백을 이용한 비동기 코드

![](/assets/images/corutine/dream_code_corutine3.png)
- 코루틴을 이용한 연속적인 비동기 코드

---


