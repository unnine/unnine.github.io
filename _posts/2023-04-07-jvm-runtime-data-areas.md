---
title: 자바 가상 머신의 원리와 이해 (2) - 런타임 데이터 영역 (Java Virtual Machine - Runtime Data Areas)
date: 2023-04-07
categories: [Java]
tags: [Java, JVM, Java Virtual Machine, Runtime Data Areas]
---
*<sup>이 시리즈는 Hotspot JVM - [JSR-392(Java SE 17)](https://docs.oracle.com/javase/specs/jvms/se17/html/index.html){:target="_blank"}를 기준으로 작성되었습니다.</sup>  

JVM은 각각의 역할을 가진 여러 영역들로 구성되어 있다. 이전 글에서 살펴보았던 각 자료형의 데이터들이 저장되는 공간도 따로 정해져 있다. 이번 글에서는 JVM에 어떤 영역들이 있고 각각 어떤 역할을 하는지 알아보자.  

# PC Register
JVM은 멀티 스레드를 지원하고 각 스레드에는 PC(program counter) 레지스터가 존재한다. 스레드가 Non-Native 메서드를 수행하고 있다면 현재 JVM 스레드의 명령어 주소를 저장한다. 만약 Native 메서드를 수행 중이라면 값을 넣지 않는다.  

# JVM Stacks
JVM의 각 스레드에는 개별로 JVM 스택이 생성되는데, 이 스택에 후입선출(LIFO) 방식으로 프레임(Frame)이 쌓이게 된다. 메서드가 실행될 때 생성되는 피연산자 스택과는 다른 스택이기 때문에 혼동해선 안 된다. **JVM 스택에는 프레임만 push, pop된다.**

- JVM 스택은 각 스레드 별로 생성되므로 Thread safety하다.
- 만약 허용된 크기보다 더 큰 스택이 필요하면 StackOverflowError가 발생한다.
- 초기화 혹은 동적 확장 시점에 메모리가 부족한 경우 OutofMemoryError가 발생한다.

## Frame
---
메서드가 호출될 때 생성되어 JVM 스택에 push되며 종료 후 pop되어 소멸한다.  
프레임은 다음과 같은 정보를 가지고 있다.

- **지역 변수 배열**  
    - 컴파일 시점에 크기가 결정된다.
    - 파라미터, 지역 변수, [returnAddress](/posts/jvm-data-types/#returnaddress-type){:target="_blank"}가 저장된다.
    - 첫 번째 요소는(index 0) 호출된 메서드의 객체 참조를 담는데 사용된다. (객체의 `this`)
- **피연산자 스택**  
    - 컴파일 시점에 크기가 결정된다.
    - 생성 시점에는 비어있다.
    - 명령어 실행에 필요한 피연산자와 연산 결과를 담는다. ([이 곳의 코드 블럭](/posts/jvm-data-types/#pass-by-value){:target="_blank"}을 참고.)
    - 반환값이 있는 메서드인 경우 메서드가 종료되기 전에 반환값이 push된다.
- **메서드 반환값**  
    - 정상적인 종료 시 피연산자 스택에서 pop하여 호출자에게 전달 된다.
    - 예외가 발생하여 종료되었을 경우, 호출자에게 전달되지 않는다.
- **예외 정보**  
    - 예외 목록에 선언한 예외 및 발생한 예외에 대한 정보가 저장된다.
- **런타임 상수 풀**  
    - 현재 메서드가 위치한 클래스의 런타임 상수 풀을 동적 연결을 이용해 참조한다.

<details>
<summary>
  <i style="font-size: 0.9rem; color: MediumOrchid;">동적 연결(Dynamic Linking)</i>
</summary><hr>
<pre style="font-size: 0.9rem; color: MediumOrchid;">
    클래스 코드는 필요한 런타임 상수 풀의 메서드나 변수를 Symbolic Reference를 통해
    참조한다. Symbolic Reference는 클래스의 이름으로 표현된다. 

    Reference Type 변수를 재정의하지 않은 toString()으로 호출하면 
    `Symbolic Reference` + `@` + `16진수로 변환한 hashcode의 값`으로 출력된다.

    ex) com.playground.my.Member@22a637e7
</pre></details><div class="b-space"></div>  

# Heap
JVM의 모든 스레드가 공유하는 런타임 영역이다. 이 곳에 모든 클래스의 객체(인스턴스) 및 배열들이 런타임에 할당된다. 힙은 JVM 시작 시 생성되며 비활성(참조되지 않는) 객체를 쓰레기로서 처리해주는 [Garbage Collector](http://localhost:4000/posts/jvm-garbage-collector/){:target="_blank"}에 의해 관리된다.

어떤 GC를 사용하느냐에 따라 메모리 영역의 구조가 달라지게 되는데, 일반적으로 Young Generation 영역, Old Generation 영역이 논리적으로 구분한다.

- Yong Generation  
새로운 객체가 할당되는 Eden과 [Copy 알고리즘](http://localhost:4000/posts/jvm-garbage-collecting-algorithm/#copy-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98){:target="_blank"}을 위한 2개의 Survivor로 구성된다. Minor GC의 대상이 된다.  

- Old Generation(Tenured Space)
Young Generation에서 일정 시간 이상 생존한 객체들이 복사되는 메모리 영역이다. Major GC의 대상이 된다.  


- 정해진 크기의 메모리가 가득 차고 더 많은 공간을 요구할 경우, OutOfMemoryError가 발생한다.  

# Method Area
JVM의 모든 스레드가 공유하는 영역이이며 힙에 생성된다. 이 영역에는 다음 항목들이 저장된다.  
- 클래스의 인스턴스 필드
- 정적 객체
- 리터럴로 선언되거나 intern된 String 정보
- 생성자를 비롯한 메서드 정보
- 런타임 상수 풀  

힙에 생성되는 이 영역에 정적 객체가 저장된다. 따라서 static을 과하게 사용하면 메모리가 부족해지게 된다.

- 정해진 크기의 메모리로 할당을 전부 할 수 없는 경우 OutOfMemoryError가 발생한다.  

# Runtime Constant Pool
Method Area에 생성되는 영역이며 Primitive Type, Reference Type을 포함한 다양한 타입이 저장된다. 각 데이터들은 다음 시점에 저장된다.  

- JVM 클래스로더에 의해 각 클래스, 인터페이스가 로드될 때
- Reflection에 의해 동적으로 로드될 때  

저장되는 데이터들의 목록이 궁금하다면 [이 곳](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-4.html#jvms-4.4){:target="_blank"}에서 확인할 수 있다. 


- 클래스 혹은 인터페이스를 생성할 때 Method Area의 크기보다 더 큰 메모리가 필요할 때 OutOfMemoryError가 발생한다.  

# Native Method Stacks
JVM의 각 스레드가 생성될 때 개별적으로 스택이 생성된다. JVM은 JNI(Java Native Interface) 기능을 지원하는데 Native 메서드가 호출될 때 쌓이는 스택 영역이다. 

<details>
<summary>
  <i style="font-size: 0.9rem; color: MediumOrchid;">JNI?</i>
</summary><hr>
<pre style="font-size: 0.9rem; color: MediumOrchid;">
    JVM에서 실행 중인 Java 코드에서 Java 외 다른 언어로 작성된
    라이브러리들을 호출하거나, 호출될 수 있게 한다.
</pre></details><div class="b-space"></div>


- 만약 허용된 크기보다 더 큰 스택이 필요하면 StackOverflowError가 발생한다.
- 초기화 혹은 동적 확장 시점에 메모리가 부족한 경우 OutofMemoryError가 발생한다.

# Permanent Generation과 Metaspace
Java 7까지는 Permanent Genration 영역에 클래스의 메타 데이터, String 정보, 정적 객체 등이 저장되었다.  
하지만 이 영역은 Java 8부터 삭제되었는데, 이유는 다음과 같다.  
> This is part of the JRockit and Hotspot convergence effort. JRockit customers do not need to configure the permanent generation (since JRockit does not have a permanent generation) and are accustomed to not configuring the permanent generation.  
> <sub>*[JEP 122: Remove the Permanent Generation](https://openjdk.org/jeps/122#Motivation){:target="_blank"}*</sub>  

또다른 Java 구현체인 JRockit과 융합하기 위한 노력의 일환으로 JRockit에 없는 Permanent영역을 제거했다고 한다.  

Java8부터 Permanent에 저장되던 String, 정적 객체의 정보는 힙(Method Area)에 위치하도록 변경되었고, 클래스 메타데이터의 정보는 Metaspace라는 Native 메모리 영역에 저장된다.  

# 참고
- [Run-Time Data Areas](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-2.html#jvms-2.5){:target="_blank"}
- [HotSpot Runtime Overview](https://openjdk.org/groups/hotspot/docs/RuntimeOverview.html){:target="_blank"}
- [JEP 122: Remove the Permanent Generation](https://openjdk.org/jeps/122){:target="_blank"}
- [Metaspace](https://wiki.openjdk.org/display/HotSpot/Metaspace){:target="_blank"}