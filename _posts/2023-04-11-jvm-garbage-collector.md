---
title: 자바 가상 머신의 원리와 이해 (3) - 가비지 컬렉터 (Java Virtual Machine - Garbage Collectors)
date: 2023-04-11
categories: [JAVA]
tags: [Java, JVM, Java Virtual Machine, Garbage Collectors]
---
*<sup>이 시리즈는 Hotspot JVM - [JSR-392(Java SE 17)](https://docs.oracle.com/javase/specs/jvms/se17/html/index.html){:target="_blank"}를 기준으로 작성되었습니다.</sup>*

이전 글에서 JVM을 구성하는 영역들에 대해 알아보았다. 이번 글에서는 JVM의 메모리를 관리하는 Garbage Collector(이하 GC)에 대해 알아본다. Java를 이용해 개발하는 프로그래머가 C/C++ 언어와 다르게 메모리에 대해 신경쓰지 않고 개발할 수 있는 이유가 바로 GC의 존재 덕분이다. 하지만 신경쓰지 않아도 개발이 된다고 해서 몰라도 되는 것은 아니다. 이 글에서는 어떤 GC가 있고 각각 어떻게 동작하는지 알아본다.

# Generations
다음 Java 객체의 수명에 대한 일반적인 분포를 나타내는 다음 그래프를 보자.  
x축은 객체의 수명을 나타내며, y축은 살아있는(이하 활성상태) 객체가 점유하고 있는 바이트를 나타낸다.

![img-description](/assets/img/posts/jvm-gc/gc-distribution.png) *Description of "Figure 3-1 Typical Distribution for Lifetimes of Objects"*

위 그래프를 보고 알 수 있는 사실은 대부분의 객체는 생성되고 나서 얼마 지나지 않아 소멸한다는 사실이다. 
GC는 이 사실에 기반하여 객체의 나이에 따라 여러 세대로(Generation)들로 나누어 메모리를 관리한다.  


> At startup, the Java HotSpot VM reserves the entire Java heap in the address space, 
> but doesn't allocate any physical memory for it unless needed. 
> The entire address space covering the Java heap is logically divided into young and old generations. 
> The complete address space reserved for object memory can be divided into the young and old generations.
>
> JVM은 실행될 때, 주소 공간 내 heap영역 전체를 미리 확보하지만 필요할 때까지 물리적으로 메모리를 할당하지 않는다.  
> Java heap영역을 포함한 전체 주소는 논리적으로 젊은 세대(Young영역)와 오래된 세대(Old영역)로 구분된다. 
> 또한, 객체들의 메모리를 위해 확보가 끝난 주소 공간도 젊은 세대와 오래된 세대로 구분할 수 있다.  
> *<sub>[Garbage Collector Implementation - Generations](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-collector-implementation.html#GUID-16166ED9-32C6-402D-BB22-FD85BCB04E57){:target="_blank"}</sub>*

JVM은 실행될 때, 미리 필요할 것으로 예상되는 heap 사이즈만큼 메모리 영역을 확보하지만 물리적인 할당은 실제 객체를 사용할 때 이루어진다. 
GC는 메모리 영역을 크게 Young Generation(이하 Young영역)과 Old Generation(이하 Old영역) 둘로 나누어 관리한다.  

# Garbage Collectors
과거부터 현재까지 Java의 새로운 버전이 계속 릴리즈되었고 그 과정에서 다양한 GC가 등장했다.  
각 버전마다 기본 GC는 다를 수 있지만 JVM 옵션을 통해 원하는 GC를 적용할 수 있다.
그럼 이제부터 각 GC에 대해 알아보도록 하자.


## Serial GC `-XX:+UseSerialGC`  
---
`Minor GC`: [Copy](/posts/jvm-garbage-collecting-algorithm/#copy-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98){:target="_blank"}  
`Major GC`: [Mark-Compact](/posts/jvm-garbage-collecting-algorithm/#mark-compact){:target="_blank"}  

단일 스레드를 이용하며 따라서 스레드간 통신하는 오버헤드가 없다.  
메모리 크기가 매우 작거나(약 100MB) 단일 프로세서에서 실행되는 환경에서 사용하는 GC이다.

<div class="white-space--dot"></div>

## Parallel(Throughput) GC `-XX:+UseParallelGC`  
---
`Minor GC`: [Copy](/posts/jvm-garbage-collecting-algorithm/#copy-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98){:target="_blank"}  
`Major GC`: [Mark-Compact](/posts/jvm-garbage-collecting-algorithm/#mark-compact){:target="_blank"}{:target="_blank"}  

java 8의 기본 GC이며 멀티 프로세서 환경의 큰 메모리를 처리하기 위한 GC다. 
여러 개의 스레드를 사용하여 Minor GC와 Major GC가 모두 병렬적으로 수행된다.
처리량이 우수하지만 긴 Stop the World(이하 STW) 현상이 발생한다.
이 GC는 최대 스레드의 개수, STW 시간, 처리량(Heap 크기) 등을 옵션으로 지정할 수 있다.

총 시간(어플리케이션 실행 시간 대비 쓰레기 수집 시간의 비율)의 98%가 쓰레기 수집에 소요되거나 
수집 후 사용 가능해진 heap 크기가 전체의 2% 미만이면 OutOfMemoryError가 발생한다.

<div class="white-space--dot"></div>

## Concurrent Mark Sweep(CMS) GC `-XX:+UseConcMarkSweepGC`
---
`Minor GC`: [Copy](/posts/jvm-garbage-collecting-algorithm/#copy-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98){:target="_blank"}  
`Major GC`: [Mark-Sweep](/posts/jvm-garbage-collecting-algorithm/#mark-sweep){:target="_blank"}  
[Java 9에서 Deprecate](https://openjdk.org/jeps/291){:target="_blank"}, [Java 14에서 삭제되었다.](https://openjdk.org/jeps/363){:target="_blank"}  

프로그램의 작업을 최대한 멈추지 않고 동시적으로 처리하여 STW 현상을 최소화하기 위한 GC다.  
다음 내용을 보고 CMS GC가 어떻게 동작하는지 살펴보자.  

> The CMS collector pauses an application twice during a concurrent collection cycle. 
> The first pause is to mark as live the objects directly reachable from the roots 
> (for example, object references from application thread stacks and registers, static objects and so on) 
> and from elsewhere in the heap (for example, the young generation). 
> This first pause is referred to as the initial mark pause. 
> The second pause comes at the end of the concurrent tracing phase and finds objects 
> that were missed by the concurrent tracing due to updates by the application threads of references 
> in an object after the CMS collector had finished tracing that object. 
> This second pause is referred to as the remark pause
>
> CMS 컬렉터는 동시적인 수집 사이클 중 2번 멈춘다. 첫 번째는 Root(스레드 스택, 레지스터 혹은 static 객체) 
> 혹은 heap의 다른 곳(예를 들면, Young영역)에서 직접 참조되며 활성 객체들을 표시하면서 일어나는데, 
> 이것이 Initial Mark 단계다.  
> 두 번째는 동시적 추적 단계가 끝날 때 일어나며, CMS 컬렉터가 객체 추적을 완료한 뒤 쓰레드에 의해 변경되어 
> 동시 추적에서 누락된 객체를 찾는다. 이것이 Remark 단계다.  
> *<sub>[8 Concurrent Mark Sweep (CMS) Collector - Pauses](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#concurrent_mark_sweep_cms_collector){:target="_blank"}</sub>*

CMS GC는 수행 시 2번 멈추는 현상이 발생하는데, Initial Mark와 Remark라는 단계로 구분되는데 
멈추는 시간(STW)이 짧은 것이 특징이다.  

그리고 해당 문서에는 Concurrent Phases(Concurrent Marking)에 대한 내용이 있는데, 종합해보면 다음과 같으며 
Initial Mark, Remark 단계를 제외하곤 모두 동시적으로 처리된다.

### GC 과정

![img-description](/assets/img/posts/jvm-gc/gc-cms.png)  

1. **Initial Mark** (STW)  
Root Set(스레드 스택, 정적 객체, 레지스터 등)을 찾는다. 이 때, 매우 짧은 STW가 발생한다.
2. **Concurrent Mark** (Concurrent)  
[Tri-Color Marking 알고리즘](/posts/jvm-garbage-collecting-algorithm/#tri-color-marking-%EC%82%BC%EC%83%89-%EB%A7%88%ED%82%B9-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98){:target="_blank"}을 이용해 활성 객체들을 마킹해나간다. 
이 과정은 어플리케이션의 스레드와 리소스를 공유하며 동시적으로 진행되기 때문에 어플리케이션의 성능이 일부 저하된다.
3. **Concurrent Preclean** (Concurrent)  
카드 테이블을 이용해 2번 과정 도중 스레드의 영향을 받은 마킹 정보를 체크한다.
4. **Remark** (STW)  
객체 추적이 완료된 후 놓친 객체가 없는지 체크한다.
5. **Concurrent Sweep** (Concurrent)  
사용하지 않는 객체를 수집하며 CMF를 방지하기 위해 연속된 빈 블록을 하나로 합친다.
6. **Concurrent Reset** (Concurrent)  
다음 동시 수집을 준비한다.

<details>
<summary>
  <i style="font-size: 0.9rem; color: MediumOrchid;">카드 테이블?</i>
</summary><hr>
<pre style="font-size: 0.9rem; color: MediumOrchid;">
    JMV에서 관리하는 byte 배열. Old 영역의 객체가 Young 객체를 참조하는 정보를 
    기록한다.
</pre>
</details>

<details>
<summary>
  <i style="font-size: 0.9rem; color: MediumOrchid;">CMF?</i>
</summary><hr>
<pre style="font-size: 0.9rem; color: MediumOrchid;">
    Young영역의 새 객체 할당이 Old영역의 처리보다 너무 빠르거나, 단편화가 반복되어 
    공간이 부족해지면 CMF(concurrent mode failure, 동시 모드 실패)가 일어난다.
</pre>
</details><div class="b-space"></div>

총 시간(어플리케이션 실행 시간 대비 쓰레기 수집 시간의 비율)의 98%가 쓰레기 수집에 소요되거나 
수집 후 사용 가능해진 heap 크기가 전체의 2% 미만이면 OutOfMemoryError가 발생하며 CMF가 발생하면 
병렬 수집기로 Full GC 처리한다.

<div class="white-space--dot"></div>

## Garbage-First (G1) Garbage GC `-XX:+UseG1GC`
---
`Minor GC`: [Copy](/posts/jvm-garbage-collecting-algorithm/#copy-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98){:target="_blank"}  
`Major GC`: [Mark-Compact](/posts/jvm-garbage-collecting-algorithm/#mark-compact){:target="_blank"}  

Java 9부터 기본 GC다.
대용량 메모리를 가진 멀티 프로세서 시스템에서 높은 처리량과 함께 STW 현상을 최소화하기 위해 설계되었다. 
우선 수집 과정을 알아보기 전에 먼저 구조를 보자.

![img-description](/assets/img/posts/jvm-gc/gc-g1.png)  

- Heap을 일정한 크기의 영역(region)로 분할한다. 이 때 각 영역은 가상 메모리의 연속 범위다.
- Young영역(Eden, Survivor)과 Old영역을 논리적으로 분할된다.
- 객체는 Young영역(Eden)에 할당되는데, 만약 객체의 크기가 영역 크기의 절반보다 크다면 Old영역의 거대한 영역(연속적인 영역의 집합)에 할당된다. 
Young영역이 가득 차면 GC가 수행되는데 상황에 따라 Young영역과 Old영역이 동시에 수집(mixed collection)되기도 한다.
- CMS의 카드 테이블처럼 외부에서 Heap 영역 내부를 참조하는 레퍼런스를 관리하는 HashTable 구조의 Rset(Remember Set)이 영역 별로 있다.

### GC 과정

이처럼 기존의 GC의 Heap 구조와는 판이하게 다른 구조를 확인했다. 그럼 이런 구조에서 G1 GC는 어떤 과정을 통해 쓰레기를 수집하는지 살펴보자.

1. **Initial marking** (STW)  
Young영역에 Minor GC가 수행되며 STW가 발생할 때, 해당 영역을 [SATB](/posts/jvm-garbage-collecting-algorithm/#satbsnapshot-at-the-beginning){:target="_blank"} 알고리즘을 이용해 마킹한다.
2. **Root region scanning** (Concurrent)  
1에서 마킹한 영역(Survivor)을 스캔하여 Old영역에 참조하고 있는 객체가 있는지 찾고 있으면 객체를 마킹한다. 
이 작업은 다음 Minor GC가 일어나기 전에 완료되어야 한다.
3. **Concurrent marking** (Concurrent)  
Heap 전체에서 참조되는 활성 객체들을 찾으며 이 대상에는 [SATB](/posts/jvm-garbage-collecting-algorithm/#satbsnapshot-at-the-beginning){:target="_blank"}로 찍은 스냅샷의 객체도 포함된다. 이 작업은 Minor GC의 STW에 의해 중단될 수 있다.
4. **Remark** (STW)
STW가 발생하며 [SATB](/posts/jvm-garbage-collecting-algorithm/#satbsnapshot-at-the-beginning){:target="_blank"}의 버퍼를 비우고, 아직 식별하지 못한 객체들을 전부 추적한다.
5. **Cleanup** (STW, Concurrent)  
STW가 발생하며 G1 GC는 비어있는 영역과 GC 대상 영역을 식별한다. 이후 동시적으로 GC가 수행되며 이 과정에서 compaction도 진행된다.  

이처럼 G1 GC는 STW를 최소화하여 GC를 수행한다.  
그런데 이 방식은 GC가 쓰레기를 수집하여 메모리를 확보하는 속도보다 어플리케이션의 작업 스레드에서 더 빠르게 객체를 할당할 수도 있다는 가능성을 갖고 있다.  

이렇게 여유 메모리가 부족해지는 현상을 할당 실패(Allocation Failure, Evacuation Failure)라고 하며 이 때는 STW가 발생하여 GC를 수행한다.

<div class="white-space--dot"></div>

## ZGC `-XX:+UseZGC`
---
`Major GC`: [Mark-Compact](/posts/jvm-garbage-collecting-algorithm/#mark-compact){:target="_blank"}  

Java 11부터 실험적으로 도입되고 Java 15부터 프로덕션용으로 지원되는 GC이며 다음 목표를 달성하기 위해 설계되었다. 참고로 64bit환경에서만 사용이 가능하다.
- 1ms 이내의 STW 
- Heap 크기에 상관없는 STW 시간
- 최대 16테라 크기의 이르는 Heap도 지원

### GC 과정
1. **Marking Start** (STW)  
Root Set에서 [SATB](/posts/jvm-garbage-collecting-algorithm/#satbsnapshot-at-the-beginning){:target="_blank"} 알고리즘을 이용해 마킹한다.
2. **Marking** (Concurrent)  
동시적으로 참조하는 활성 객체들을 마킹해나간다. 
3. **Marking End** (STW)  
새롭게 추가되는 등 마킹되지 않은 객체 그래프가 있다면 2번 과정부터 반복하여 마킹한다.
4. **Prepare Relocate** (Concurrent)  
마킹한 활성 객체들의 영역들을 RelocationSet에 넣고 각 객체들의 재배치되는 새 주소는 Heap 외부에 있는 Forwarding Table에 기록된다.
5. **Relocate Start** (STW)  
재배치 세트의 Root를 재배치한다.
6. **Relocate** (Concurrent)  
재배치 세트를 기반으로 재배치하며 참조를 수정한다. 비워진 영역들은 다른 스레드의 재배치나 할당 영역으로 사용될 수 있다.  

위 과정 중 6번은 어플리케이션의 작업과 동시적으로 처리된다. 만약 재배치의 참조를 수정하는 도중 어플리케이션에서 재배치 대상을 참조하려고 하면 어떻게 처리할까? 이를 이해하기 위해서 Color Pointer, Load Barrier에 대해 알아보자.

### Color Pointer  
Heap 객체에 대한 메모리 주소와 해당 객체의 상태를 알려주는 메타 데이터를 가지고 있다.

![img-description](/assets/img/posts/jvm-gc/zgc-color-pointer.png)  

색상 포인터는 항상 64bit를 사용하는데 32bit로는 최대 4기가바이트의 크기밖에 표현할 수 없기 때문이다. 이것이 64bit환경에서만 사용이 가능한 이유다.  

메타 데이터는 다음과 같은 역할을 하는 4개의 비트로 구성되어 있다.
- **Finalizable**  
   참조되지 않는 비활성 객체 여부를 판단하는 비트다.
- **Remapped**  
    재배치 참조가 최신인지 여부를 판단하는 비트다.
- **Marked0, Marked1**  
    활성 객체인지 여부를 판단하는 비트이다.

### Load Barrier
재배치는 동시적으로 수행되기 때문에 어플리케이션이 재배치되어야 할 객체에 접근할 수 있다. 이 문제를 Load Barrier가 다음과 같이 해결해준다.
1. Remapped가 1이면 참조를 반환한다.
2. 0이면 RelocationSet에 대상 객체가 있는지 확인한다. 존재하지 않는다면 재배치 대상이 아니므로 Remapped 비트를 1로 수정하고 참조를 반환한다.
3. 이 단계까지 왔다는 것은 찾는 객체가 재배치 대상 객체임을 뜻한다. 해당 객체의 주소 정보가 전달 테이블(Forwarding Table)에 없다면 새 주소를 추가한다.
4. 전달 테이블에서 주소를 찾아 수정한 뒤 Remapped 비트를 1로 수정하고 참조를 반환한다.

<div class="white-space--dot"></div>

# 후기
이렇게 전체적으로 Java의 GC의 종류와 동작 방식을 살펴보았다.  

공부하면서 어려웠던 점은 ZGC에 대한 설명은 공식 문서들이 추상적으로 설명한 경우가 많아 구체적인 과정으로 풀어서 이해하는게 난해했다는 점이다. 인터넷을 통해 검색해봐도 포스팅마다 동작 방식의 표현이 상이한 경우가 많아 더욱 어려웠다.  

그래서 결국 여러 공식 문서와 정리가 잘 된 것으로 생각되는 포스팅들을 기반으로 공통 분모들을 추려 큰 흐름이라도 이해하고 넘어갈 수 있도록 노력했다. 더 깊은 이해가 있었다면 보다 명확하게 파악할 수 있었을 거란 아쉬움이 든다.

이번 주제는 내게 JVM의 메모리가 어떻게 관리되는지 더 깊게 이해할 수 있게 해주었고, 반면 나 자신에게 부족함을 느껴 성장에 대한 자극을 더욱 받게 한 경험이기도 했다.


# 참고
- [Java SE, HotSpot Virtual Machine Garbage Collection Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/collectors.html){:target="_blank"}
- [Java Garbage Collection - NAVER D2](https://d2.naver.com/helloworld/1329){:target="_blank"}
- [JEP 439: Generational ZGC](https://openjdk.org/jeps/439){:target="_blank"}
- [ZGC. What's new in JDK 16](https://malloc.se/blog/zgc-jdk16){:target="_blank"}
- [An Introduction to ZGC](https://www.baeldung.com/jvm-zgc-garbage-collector){:target="_blank"}
- [JVM과 Garbage Collection - G1GC vs ZGC](https://huisam.tistory.com/entry/jvmgc){:target="_blank"}