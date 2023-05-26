---
title: 자바 가상 머신의 원리와 이해 (4) - 가비지 컬렉션 알고리즘 (Java Virtual Machine - Garbage Collection Algorithms)
date: 2023-04-13
categories: [JAVA]
tags: [Java, JVM, Java Virtual Machine, Garbage Collection Algorithms]
---
*<sup>이 시리즈는 Hotspot JVM - [JSR-392(Java SE 17)](https://docs.oracle.com/javase/specs/jvms/se17/html/index.html)를 기준으로 작성되었습니다.</sup>*

이전 글에서 각 Garbage Collector에 대해 알아보았다. 이번 글에서는 우리가 보았던 GC들이 사용하는 알고리즘에 대해 정리한다. 

# Minor GC
Heap의 Young Generation 영역의 쓰레기를 수집하는 것을 Minor GC라고 한다.  
이 과정에서 모든 스레드가 멈추는 짧은 Stop the World 현상이 발생한다.
Young 영역은 Eden 영역과 2개의 Survivor 영역으로 구분된다는 사실을 기억하고 다음을 보자.

## Copy 알고리즘
![img-description](/assets/img/posts/jvm-gc-algo/y1.png)  

**1)** 새로운 객체는 Eden 영역에 할당된다. 이 때, 두 Survivor 영역은 비어있다.

![img-description](/assets/img/posts/jvm-gc-algo/y2.png)  

**2)** Eden 영역이 가득차면 Minor GC가 수행된다.

![img-description](/assets/img/posts/jvm-gc-algo/y3.png)  

**3)** 활성(참조되고 있는) 객체들은 Survivor0 영역으로 복사되고 Eden영역은 비워진다. 
여기서 생존한 객체들은 살아남은 시간을 파악하기 위해 카운팅을 하게 되는데 이를 Aging이라고 한다. 

![img-description](/assets/img/posts/jvm-gc-algo/y4.png)  

**4)** 다시 1~2번 과정이 일어나고 Eden, Survivor0 영역의 활성 객체들은 Survivor1 영역으로 복사되며 
생존한 객체들은 Aging된다. 그리고 Eden, Survivor1 영역은 비워진다.  
다음 Minor GC가 수행될 때는 다시 Survivor0 영역으로 활성 객체가 이동하고 Aging되며 Eden, Survivor1 영역이 비워지는 사이클을 반복한다.

![img-description](/assets/img/posts/jvm-gc-algo/y6.png)  

**5)** Age가 임계값을 넘은 객체는 Old 영역으로 복사된다.

<div class="white-space--dot"></div>

# Major GC
Old Generation을 대상으로 수행되는 GC이며 런타임 환경에서 Heap에 새 객체를 생성할 수 없는 경우 수행된다. 새 객체를 생성하는데 실패하면 GC를 실행한 후 다시 재할당을 시도한다. 그래도 실패한다면 OutOfMemoryError가 발생한다.

## Reference Counting
참조 카운팅 알고리즘은 각 객체의 헤더에 참조 카운트를 기록하여 참조 카운트가 0보다 크면 활성 상태로 간주하는 알고리즘이다. 참조가 될 떄마다 카운트가 증가하며 참조가 해제될 때마다 카운트가 감소한다. 이 알고리즘은 간단하지만 치명적인 단점이 있다. 순환 참조인 경우에는 수집할 방법이 없는 것이다. 

## Tri-Color Marking (삼색 마킹 알고리즘)
현재 Java의 GC는 이 알고리즘을 사용한다.

![img-description](/assets/img/posts/jvm-gc-algo/cycle1.png)  

**1)** RootsSet의 객체들을 검은색으로 마킹하고 나머지는 흰색으로 마킹한다.
<details>
<summary>
  <i style="font-size: 0.9rem; color: MediumOrchid;">RootSet</i>
</summary><hr>
<pre style="font-size: 0.9rem; color: MediumOrchid;">
    Tri-Color Marking 알고리즘이 적용된 GC는 참조되는 것이 확실히 보장되는 Roots로부터 
    활성 객체를 찾게 된다. JVM에서 활성 상태가 보장되는 Roots 객체들은 다음과 같다.
    - 정적 객체
    - 스택에 적재된 지역 변수 및 매개 변수
    - 현재 실행 중인 모든 스레드
    - JNI용 Java 객체
</pre>
</details><div class="b-space"></div>

![img-description](/assets/img/posts/jvm-gc-algo/cycle2.png)  
**2)** Roots에서 직접 참조되는 객체들은 회색으로 마킹한다.

![img-description](/assets/img/posts/jvm-gc-algo/cycle3.png)  
**3)** 회색 객체들은 순회하며 검은색으로 마킹하고 참조되는 객체들은 회색으로 마킹한다.

![img-description](/assets/img/posts/jvm-gc-algo/cycle4.png)  
**4)** 2~3 과정을 회색이 남지 않을 때까지 반복한다.  

## SATB(Snapshot at the beginning)  
Tri-Color Marking 알고리즘은 단점이 있다. 수집 스레드가 마킹해 나가면서 어떤 객체를 검은색으로 마킹했는데, 새롭게 생성되거나 참조가 변경되어 이 객체와 이어진 객체는 흰색으로 마킹된다. 
그럼 활성 객체로 간주되어야 함에도 불구하고, 비활성 객체로 판단되어 수집되어 버린다.  

SATB 알고리즘은 마킹을 시작할 때, 먼저 참조 셋의 스냅샷을 찍는다. 그리고 마킹 과정에서 새롭게 생성되거나 참조가 변경된 객체들은 전부 활성 객체로 간주하여 문제를 해결한다. 다만, 이 과정에서 죽은 객체도 활성 객체로 간주하는 Floating Garbage현상이 발생한다. 

## Mark-Sweep
Root Set으로부터 참조하고 있는 활성 객체들을 마킹(Mark)한 뒤 비활성 객체들은 삭제(Sweep)하는 알고리즘이다. 단편화가 발생하는 문제가 있다.  

Java 9에 deprecate되고 Java 14에서 삭제된 GC인 CMS(Concurrent Mark Sweep) GC에서 사용된다.

## Mark-Compact
Mark-Sweep의 방식에서 압축(compaction)이 추가된 알고리즘.

Root Set으로부터 참조하고 있는 활성 객체들을 마킹(Mark)한 뒤 비활성 객체들은 삭제(Sweep)한다. 
그 후 활성 객체들을 heap메모리의 가장 앞 부분부터 채워 단편화를 해결한다.  

보통 Serial GC, Parallel GC에서는 모든 스레드를 멈춘 후(Stop the World) Old 영역 전체를 대상으로 동작하는 반면에, G1 GC나 ZGC같은 경우 Heap 메모리를 일정한 단위로 나눈 영역(region)별로 수행하며 어플리케이션의 스레드를 멈추지 않고 동시적으로 동작한다.

# 참고
- [Java Garbage Collection Basics](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html){:target="_blank"}
- [Tracing garbage collection](https://en.wikipedia.org/wiki/Tracing_garbage_collection){:target="_blank"}
- [Guide to Garbage Collector Roots](https://www.baeldung.com/java-gc-roots){:target="_blank"}