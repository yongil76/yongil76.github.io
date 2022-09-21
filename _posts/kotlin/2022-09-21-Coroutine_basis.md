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

---

## Block vs Suspend?

> 코루틴을 실행 중인 스레드를 차단(Block)하지 않는 정지(Suspend)를 지원하므로 단일 스레드에서 많은 코루틴을 실행할 수 있습니다. 
> 정지는 많은 동시 작업을 지원하면서도 차단보다 메모리를 절약합니다.

- 상황
  - A 프로세스가 시작되고, 외부 서비스와 통신하는 B 프로세스가 있다고 가정한다.
  - B 프로세스와 완료 여부와 관계없이, A 프로세스가 해야할 일이 남아있다.
  - 또한, 외부 서비스의 네트워크 환경이 좋지 않기 때문에, A 프로세스의 일부 작업만 하고 B 외부 서비스에 최대한 빠른 시점에 통신을 해야 한다.
  
- Block
  - 단일 스레드에서 B가 끝나야 A 작업이 완료될 수 있음
  
~~~kotlin
fun main() = runBlocking { // this: CoroutineScope
  println("ProcessA Started!")
  launch { doConn() }
  delay(500L)
  println("ProcessA Finished!")
}

fun doConn() {
  println("ProcessB Started!")
  println("Network connection...")
  // delay(1000L)
  // Suspend function 'delay' should be called only from a coroutine or another suspend function
  println("ProcessB Finished!")
}
~~~

~~~text
// Result
ProcessA Started!
ProcessB Started!
Network connection...
ProcessB Finished!
ProcessA Finished!
~~~
  
- Suspend
  - B가 끝나지 않아도, A의 작업을 완료할 수 있음
  - doConn(B)가 실행(A의 suspend)되고, A의 작업이 재개(resume)

~~~kotlin
fun main() = runBlocking { // this: CoroutineScope
  println("ProcessA Started!")
  launch { doConn() }
  delay(500L)
  println("ProcessA Finished!")
}

suspend fun doConn() {
  println("ProcessB Started!")
  println("Network connection...")
  delay(1000L)
  println("ProcessB Finished!")
}
~~~

~~~text 
// Result
ProcessA Started!
ProcessB Started!
Network connection...
ProcessA Finished!
ProcessB Finished!
~~~

---

## Dispatcher?
- Coroutine을 스레드나 스레드 풀에 할당해주는 장치
- 특정 스레드나 스레드 풀을 지정할 수 있음

---
