---
title:  "Stream groupingBy"
excerpt: "Stream groupingBy"

categories:
- Java

tags:
- Java8

toc: true

toc_sticky: true

toc_label: "stream"

last_modified_at: 2022-05-08T09:59:00-05:00

---

## groupingBy?

---

> The static factory methods Collectors.groupingBy() and Collectors.groupingByConcurrent() provide us with functionality similar to the ‘GROUP BY' clause in the SQL language. We use them for grouping objects by some property and storing results in a Map instance.

## Function?

---
~~~java
static <T,K> Collector<T,?,Map<K,List<T>>> 
  groupingBy(Function<? super T,? extends K> classifier)
~~~

~~~java
@Test
public void groupingBy() {
    final List<Item> list = new ArrayList<>();
    list.add(new Item("Kwon", 3));
    list.add(new Item("Kwon", 4));
    list.add(new Item("Kim", 5));
    
    final Map<String, List<Item>> collect = 
        list.stream().collect(Collectors.groupingBy(Item::getName));       
    
    collect.forEach((key, value) -> {
        for (Item item : value) {
            System.out.println(item.getName()+' '+item.getVal());
        }
    });
}
~~~
- SQL의 group by와 동일한 결과
- Result
~~~
Kwon 3
Kwon 4
Kim 5
~~~
---
~~~java
static <T,K,A,D> Collector<T,?,Map<K,D>>
  groupingBy(Function<? super T,? extends K> classifier, 
    Collector<? super T,A,D> downstream)
~~~


~~~java
@Test
public void groupingBy2() {     
    final List<Item> list = new ArrayList<>();
    list.add(new Item("Kwon", 3));
    list.add(new Item("Kwon", 4));
    list.add(new Item("Kim", 5));

    final Map<String, Item> collect = list.stream()
                            .collect(Collectors.groupingBy(Item::getName, Collectors
                            .reducing(null, (item1, item2) -> {
                                if (item1 == null) {
                                    return item2;
                                }

                                item1.setVal(item1.getVal() + item2.getVal());
                                return item1;
                            })));

    collect.forEach((key, value) -> System.out.println(key + ' ' + value.getName() + ' ' + value.getVal()));
    System.out.println(collect.getClass());
}
~~~

- Value에 대해서 Collectors 함수를 통해서 조작이 가능
- Reducing은 identity라는 default value를 가지고 있어야 하며, default value에 대한 처리가 필요할 수 있음
- Result
~~~
Kwon Kwon 7
Kim Kim 5
class java.util.HashMap
~~~
---

~~~java
static <T,K,D,A,M extends Map<K,D>> Collector<T,?,M>
  groupingBy(Function<? super T,? extends K> classifier, 
    Supplier<M> mapFactory, Collector<? super T,A,D> downstream)
~~~

~~~java
@Test
public void groupingBy3() {
    final List<Item> list = new ArrayList<>();
    list.add(new Item("Kwon", 3));
    list.add(new Item("Kwon", 4));
    list.add(new Item("Kim", 5));

    final Map<String, Item> collect = list.stream()
                                        .collect(Collectors.groupingBy(Item::getName, TreeMap::new, Collectors
                                        .reducing(null, (item1, item2) -> {
                                            if (item1 == null) {
                                                return item2;
                                            }

                                            item1.setVal(item1.getVal() + item2.getVal());
                                            return item1;
                                        })));

    collect.forEach((key, value) -> System.out.println(key + ' ' + value.getName() + ' ' + value.getVal()));
    System.out.println(collect.getClass());
}
~~~
- mapFactory를 통해 생성된 Map의 구체 클래스를 설정 가능
- Result
~~~
Kim Kim 5
Kwon Kwon 7
class java.util.TreeMap
~~~

---
## Null Exception?
- stream()을 사용할 때는, NPE에 대한 고려가 항상 필요

### key가 null일 경우?
~~~java
public String getName() {
    if ("Kim".equals(name)) {
        return null;
    }

    return name;
}
~~~

- Result
~~~ java
java.lang.NullPointerException: element cannot be mapped to a null key
~~~

- Null safe code
~~~ java
.filter(item -> item.getName() != null)
~~~

### value가 null인 경우?

- NPE 발생하지 않음

---

## Conclusion?

- groupingBy 함수를 통해서, 복잡한 Map 리턴 구문을 간단하게 stream으로 작성할 수 있음
- key에 대해서 NPE가 발생하지 않도록 filter를 적절히 사용
- 다차원 Map(Map<String, Map<String, Object>>에 대해서도 쉽게 사용이 가능
---

## Reference?

- https://www.baeldung.com/java-groupingby-collector 


