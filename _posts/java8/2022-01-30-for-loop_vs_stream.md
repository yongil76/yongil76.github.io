---
title:  "for-loop vs stream"
excerpt: "for문 vs 스트림"

categories:
- Java

tags:
- Java8

toc: true

toc_sticky: true

toc_label: "stream"

last_modified_at: 2022-01-30T12:24:00-05:00

---

## Definition?

### for-loop

---

- 일반적으로 사용하는 for 구문
- 자바 8 이전에 Collection를 다루기 위해서 사용

~~~ java
for (Person person : list) {     
    func(person);
}

// 
for (int i = 0; i < list.size(); i++){
    func(list.get(i));
}
~~~

### stream

---
- 자바 8부터 Collection를 다루기 위해서 사용 가능
- Lazy evaluation, 모든 연산을 수행하지 않을 수 있음
- filter
  - 특정 element를 걸러내기 위한 작업이 가능
- collect
  - stream으로 처리된 element들이 원하는 Collection으로 손쉽게 변환이 가능
- map
  - element를 원하는 형태로 변환이 가능

~~~ java
final List<Byte> collect = list.stream()
                              .filter(i -> i > 1)
                              .map(Integer::byteValue)
                              .collect(Collectors.toList());
~~~

## convert for-loop to stream

### Example

---

~~~java
    @Test
    public void for문_stream_변환(){
        final List<Person> copy = new ArrayList<>(list);

        int sum = 0;
        for(int i = 0; i < list.size(); i++){
            if(list.get(i).getIndex() % 2 == 0) {
                if(list.get(i).getIndex() % 3 == 0) {
                    sum += list.get(i).getIndex();
                }
            }
        }

        final int streamSum = copy.stream()
                                  .map(Person::getIndex)
                                  .filter(index -> index % 2 == 0)
                                  .filter(index -> index % 3 == 0)
                                  .reduce(Integer::sum)
                                  .get();

        assertEquals(sum, streamSum);
    }
~~~
- 유지 보수를 위해서 stream에 동작을 추가할 때, 기존 코드에 영향을 최소화하며 추가가 가능
- 여러가지 연산을 한 줄의 코드로 간단히 표현할 수 있음(가독성 증가)
---

## Difference?

### for-loop

---
- 일반적으로 stream의 forEach()보다 performance가 좋음
  - forEach()도 내부적으로 for문을 호출하지만, 함수형 인터페이스에 대한 검증으로 인한 오버헤드가 증가
- for문안에 여러 로직이 있다면, stream보다 가독성이 떨어짐
- 원시적 타입(int)의 Array 탐색에 대해서는 for-loop가 훨씬 성능이 좋음
  - 원시적 타입은 Stack 영역에서 접근되므로, 데이터에 대한 접근 시간이 오래 걸리지 않고 원시적 타입에 대한 컴파일러 최적화가 잘 이루어져 있음
  - 반면에, 원시적 타입이 아닌 선언한 타입(Class)에 대해서는 <mark style='background-color: #fff5b1'>Heap 영역에서 접근이 이루어지기 때문에</mark>, 접근 시간 자체가 오래 걸리며 컴파일러 최적화가 Stack 영역만큼 좋지 않기 때문에 for-loop가 stream의 차이가 원시적 타입보단 작음

---

### stream

---
- parellel 처리가 가능하므로, 데이터의 수가 많다면, for-loop보다 빠를 수 있음
  - <mark style='background-color: #fff5b1'>parllelStream()으로 간단하게 병렬 처리가 가능</mark>
- 가독성이 좋고 유지 보수가 용이함

---


## Conclusion

~~~java
  list.forEach(item -> func(item));
~~~
<mark style='background-color: #fff5b1'>위 코드처럼 stream을 단순히 forEach()만을 위해서 쓰는 것은 올바른 사용이 아니다.</mark> 또한
forEach()를 호출하는 stream이 parllelStream()이 아니라면 성능적으로 손해가 발생한다.

아래에 코드에서 볼 수 있듯이, forEach도 for-loop를 호출하기 때문에 결과적으론 같은 행위를 하는 것이다. 
아래 코드를 보면 데이터가 작을수록 for-loop가 stream의 성능 차이가 나는 이유를 알 수 있다.

~~~java
    // ArrayList forEach
    @Override
    public void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        for (E e : a) {
            action.accept(e);
        }
    }
~~~

for-loop에 대한 호출과 함수형 인터페이스를 검증하기 위한 코드가 삽입되었기 때문에 오버헤드가 발생한 것인데, 
데이터의 수가 작을수록 for문으로 순회하는 시간이 작기 때문에 추가된 코드들이 경과 시간에 영향을 줄 수 있게 된다.

<mark style='background-color: #fff5b1'>stream()을 올바르게 사용하기 위해서는, Collection의 데이터들을 가공하기 위한
filter, collect, map 등의 함수들이 필요한 경우에 stream()을 써야 의도대로 쓰고 있는 것이다.</mark>