---
title:  "Kotlin class for data"

excerpt: "Kotlin class for data"

categories:
- kotlin

tags:
- kotlin

toc: true

toc_sticky: true

toc_label: "kotlin"

last_modified_at: 2023-01-10T22:50:00-05:00

---

~~~kotlin
// Constructor
class T1(
    val name: String,
    val id: Int
)

fun main(args: Array<String>) {
  val t1 = T1(
    name = "Kwon", 
    id = 1
  )
  // t1.id = 30
}
~~~

- id는 `val`이므로, t1.id는 컴파일 에러

~~~kotlin
// Field
class T1_1 {
  val name: String = ""
  val id: Int = 0
  lateinit var lateInitial: String
}

fun main(args: Array<String>) {
  // Field excluded lateinit var is must be initialized
  val t1_1 = T1_1()
}
~~~

- `()` 필드 생성자가 없는 클래스

~~~kotlin
// Entity
@Entity
class T2(
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    @Column(name = "id", nullable = false)
    var id: Long? = null,
    val name: String = "",
    val option: Boolean = true
)

fun main(args: Array<String>) {
  // Entity
  val t2 = T2()
  t2.id = 1
  // t2.name = "Kim" // Entity always readable

  // Update or insert Entity
  val newEntity : T2 = T2(
    id = t2.id,
    name = t2.name,
    option = false
  )
}
~~~

- `Entity`로 사용할 수 있는 클래스
- `@GeneratedValue`는 `var`로 사용해야 `set`이 가능
- `update` or `insert`를 위해서는, 기존 `Entity`를 사용하는 것이 아니라, 신규 `Entity`를 생성
  - `Kotlin`은 `immutable`을 선호, 이미 생성된 객체에 존재하는 상태가 변경되는건 항상 사이드 이펙트가 존재 


~~~kotlin
// DTO
data class T3(
  var id: Long? = null,
  var name: String
)

fun main(args: Array<String>) {
  // DTO
  // Since the property has not yet been determined
  val t3 = T3(
    name = "Kim"
  )

  // Id property is determined!
  t3.id = 2

  // Data class deep copy
  val t3_2 = t3.copy()
  if (t3 == t3_2) {
    println("same")
  }
  
  // Not same
  t3.id = 4
  if (t3 != t3_2) {
    println("not same")
  }
}
~~~

- `DTO`로 사용할 수 있는 클래스
- `Entity`는 `Database`의 제약 조건을 항상 지키는 클래스
- `Persistence layer`외에 다른 layer에서 사용핧 수 있는 클래스가 필요
  - `Entity`의 `val`인 프로퍼티들을 결정할 수 없는 상황이거나 굳이 채울 필요가 없을 때