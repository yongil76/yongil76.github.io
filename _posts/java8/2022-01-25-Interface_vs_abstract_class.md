---
title:  "Interface vs Abstract class"
excerpt: "인터페이스 vs 추상 클래스"

categories:
- Java

tags:
- Java8

toc: true

toc_sticky: true

toc_label: "Index"

last_modified_at: 2022-01-23T15:46:00-05:00

---

## Java 8 이전?

### Interface
- 인터페이스를 구현하는 클래스에서 해당 메소드들을 구현해야함

### Abstract class
- 인스턴스를 생성할 수 있기 때문에, 상태를 가질 수 있음
- 일반적인 클래스로의 역할이 가능
- 추상 클래스를 상속받는 클래스는 추상 메소드들에 대해서 구현이 필요

### Interface vs Abstract Class
- 인터페이스는 다중 구현이 가능하며, "HAS" 관계
- 추상클래스는 하나만 상속할 수 있으며, "IS" 관계

---

## Java 8 이후?

### Interface
~~~ java
public interface Java8Interface {
    int JAVA_VERSION = 8;
}
~~~

- 클래스 변수를 가질 수 있으나, <mark style='background-color: #fff5b1'>인스턴스 변수는 가질 수 없음</mark>
- JAVA 변수에 값을 할당해줘야함

~~~ java
public interface Java8Interface {
    int JAVA_VERSION = 8;

    default void java8Function(){
        System.out.println("THIS IS JAVA " + JAVA_VERSION);
    }
}

public class Java8Class implements Java8Interface{
    @Override
    public void java8Function() {
        Java8Interface.super.java8Function();
        System.out.println("Java8Class call java8Function()");
    }
}

~~~

- 추상 클래스처럼 default를 이용해 메소드를 구현 가능
- 구현한 클래스에서 해당 메소드를 오버라이드 가능

~~~ java
public interface Java8Interface {
    int JAVA_VERSION = 8;

    default void java8Function(){
        System.out.println("THIS IS JAVA " + JAVA_VERSION);
    }

    static void staticFunction(){
        System.out.println("THIS IS STATIC FUNCTION");
    }
}

public class Java8Class implements Java8Interface{
    @Override
    public void java8Function() {
        Java8Interface.super.java8Function();
        System.out.println("Java8Class call java8Function()");
    }
    
    public void callStaticMethod(){
        Java8Interface.staticFunction();
    }
}
~~~

- <mark style='background-color: #fff5b1'>인터페이스에 클래스 메소드를 선언할 수 있음</mark>
- 클래스 메소드는 오버라이딩이 불가능
- 모든 클래스에서 동일하게 적용될 수 있는 메소드를 정의

---

## Java8 Interface vs Abstract class

###  <mark style='background-color: #fff5b1'> 인터페이스는 상태를 가지지 못한다. </mark>
  - 클래스는 인스턴스를 만들 수 있으며, 인스턴스 변수들을 가질 수 있음
  - 인터페이스는 클래스 변수는 가질 수 있으나, 인스턴스 변수를 가질 순 없음
  - 클래스는 생성자를 가질 수 있지만, 인터페이스는 생성자를 가질 수 없음

---
