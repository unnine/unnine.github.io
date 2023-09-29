---
title: 자바 가상 머신의 원리와 이해 (1) - 데이터 유형 (Java Virtual Machine - Data Types)
date: 2023-04-05
categories: [Java]
tags: [Java, JVM, Java Virtual Machine]
---
*<sup>이 시리즈는 Hotspot JVM - [JSR-392(Java SE 17)](https://docs.oracle.com/javase/specs/jvms/se17/html/index.html){:target="_blank"}를 기준으로 작성되었습니다.</sup>*

<!-- https://cr.openjdk.org/~iris/se/17/spec/fr/java-se-17-fr-spec/ -->
프로그래머가 작성한 자바 코드는 Java Compiler에 의해 Byte Code로 변환된 후 JVM에 의해 실행된다. 어떤 환경이든 JVM이 있다면 Byte Code로 프로그램을 실행할 수 있는 것이 Java의 플랫폼 이식성이 뛰어난 이유다. 
결국 우리가 작성한 코드는 JVM에 의해 동작하기 때문에 Java에 대해 깊게 이해하기 위해서는 먼저 JVM에 대해 알아야 한다. 이번 포스팅에서는 JVM의 데이터 유형에 대해 알아본다.


# Java Specification
먼저 JVM의 사양이 어떻게 정해지는지 알아보자.  
Java의 사양은 JCP(Java Community Process)에서 표준화 과정을 거쳐 정해진다. 이 과정에서 Java 사양 공식 스펙 문서인 JSR(Java Specification Request)이 기술되며 이렇게 제안된 사양이 최종 승인이 되면 각 벤더사에서 해당 사양에 맞게 구현하게 된다. 이번 포스팅부터 이어지는 JVM 시리즈는 [JSR-392(Java SE 17)](https://docs.oracle.com/javase/specs/jvms/se17/html/index.html){:target="_blank"}를 기준으로 작성한다.  

<div class="white-space--dot"></div>

# JVM의 데이터 유형
Java의 각 Primitive Type의 크기는 몇 byte이어야 하고 Reference Type의 기본값은 null이어야 한다 등 우리가 공부하며 마주하는 Java의 데이터 유형에 대한 정보가 JSR에 기술되어 있으며 크게 **Primitive Type**과 **Reference Type**, 두 가지의 유형이 존재한다.  

이 글에서는 다음의 항목을 차례대로 알아본다.
- ~~**IEEE-754**~~ (내용이 길어 별도의 포스팅으로 분리하였다. [이 곳에서 살펴보자.](/posts/ieee-754/){:target="_blank"})
- **returnAddress Type**
- **Reference Type**

> Primitive Type들의 크기 범위 등 기본 명세는 [여기](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-2.html#jvms-2.3)에 간단명료하게 정리되어 있으므로 본 포스팅에서는 따로 작성하지 않는다.

<div class="white-space--dot"></div>

## returnAddress Type
---
JSR에서 소개하는 Primitive Type에는 생소한 타입이 하나 있다.

> The primitive data types supported by the Java Virtual Machine are the numeric
> types, the boolean type, and the returnAddress type. The numeric types consist of the integral types and the floating-point types.
>
> JVM이 지원하는 primitive 타입은 숫자형과 부울형, returnAddress형이 있으며 숫자형은 정수형과 부동 소수점 유형으로 구성된다.  
> <sub>*JSR-392(2.3 Primitive Types and Values)*</sub>  

숫자형이나 부울형은 익숙한데 returnAddress 타입은 도대체 뭘까?  
다음 내용을 보자.

> The returnAddress type is used by the Java Virtual Machine's jsr, ret, and jsr_w
> instructions (§jsr, §ret, §jsr_w).  
> The values of the returnAddress type are pointers to the opcodes of Java Virtual Machine instructions.
> Unlike the numeric primitive types, the returnAddress type does not correspond to any Java programming
> language type and cannot be modified by the running program.
>
> returnAddress타입은 JVM의 jsr, ret, jsr_w 명령어에 의해 사용된다.  
> returnAddress 타입은 JVM 명령의 opcode(명령어 코드)들에 대한 포인터이다.  
> 숫자형 기본 타입과 다르게 Java의 언어 타입이 아니며 실행 중인 프로그램에서 수정 할 수 없다.  
> <sub>*JSR-392(2.3.3 The returnAddress Type And Values)*</sub>

JVM이 수행하는 명령어는 opcode(명령 코드)와 operand(피연산자)로 구성되는데, returnAddress타입은 opcode인 [jsr](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-6.html#jvms-6.5.jsr){:target="_blank"}, [jsr_w](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-6.html#jvms-6.5.jsr_w){:target="_blank"}, [ret](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-6.html#jvms-6.5.ret){:target="_blank"}과 관련이 있다고 한다.  
jsr, jsr_w는 서브 루틴을 호출하는 명령이고, ret은 서브 루틴이 종료된 후 복귀할 때 호출되는 명령이며 지역 변수에서 주소를 가져온다. 이 3가지 명령에 의해 어떻게 사용되는 것인지 좀 더 자세히 알아보자.  

<details>
<summary>
  <i style="font-size: 0.9rem; color: MediumOrchid;">서브 루틴이란?</i>
</summary><hr>
<pre style="font-size: 0.9rem; color: MediumOrchid;">
  Java의 경우 메서드와 동일하다고 생각하면 된다.  
  Java의 서브 루틴은 정적(static) 서브루틴, 비정적(non-static) 서브루틴으로 구분하는데
  정적 서브루틴은 클래스의 멤버이고, 비정적 서브루틴은 객체의 멤버로 동작한다.
</pre>
</details><br/>

> When a jsr("jump to subroutine") instruction that invokes the subroutine is
> executed, it pushes its return address, the address of the instruction after the jsr that
> is being executed, onto the operand stack as a value of type returnAddress. The
> code for the subroutine stores the return address in a local variable. At the end of
> the subroutine, a ret("return from subroutine") instruction fetches the return address from the local variable
> and transfers control to the instruction at the return address.
>
> 서브 루틴을 호출하는 jsr 명령이 실행될 때, 실행 중인 jsr의 작업이 끝나고 나서 수행될 명령의 주소인 반환 주소를 returnAddress 타입의 값으로 피연산자 스택에 push한다. 
> 서브 루틴 코드는 지역 변수에 returnAddress값을 저장하고 서브 루틴이 끝날 때 ret 명령어가 수행되며 지역 변수에서 반환 주소를 가져오며 해당 주소로 제어 흐름을 넘긴다.  
> <sub>*JSR-392(4.10.2.5 Exceptions and filnally)*</sub>

말이 좀 복잡한데, 정리하면 다음과 같다.
1. 명령어를 처리해나가다가 제어 흐름 명령 코드인 jsr, jsr_w을 만나면 서브 루틴이 호출된다.
2. 서브 루틴이 호출될 때, 종료된 후 되돌아올 반환 주소를 서브 루틴의 피연산자 스택에 push한다. 이 때, 반환 주소는 지역 변수 returnAddress의 값이 된다.
3. 서브 루틴이 끝날 때, ret 명령은 지역 변수 returnAddress에서 반환 주소를 가져와 해당 주소로 제어 흐름을 넘긴다.

여기까지만 보면 이름도 비슷한 return과 관련이 있나 싶은 생각이 든다. 그런데 다음을 보자.

> Note that jsr (§jsr) pushes the address onto the operand stack and ret gets it out of a local variable. This asymmetry is intentional.
> The ret instruction should not be confused with the return instruction (§return). A return instruction returns control from a method to its invoker, without passing any value back to the invoker.
>
> jsr은 피연산자 스택에 주소를 push하고 ret는 지역 변수에서 주소를 꺼내온다.
> ret은 return과 혼동해선 안 된다. return은 호출자(현재 메서드를 호출한)로 어떤 값도 전달하지 않고 제어를 반환하는 명령이다.  
> <sub>*JSR-392(6.5. Instructions - ret)*</sub>

return은 메서드를 종료하면서 현재 메서드를 호출한 메서드에 제어를 반환하는 명령이고, returnAddress는 그 호출자의 주소를 저장하는 타입이라는 것이다. 마지막으로 returnAddress가 언급되는 설명을 하나 더 보자.

> The Java Virtual Machine can support many threads of execution at once (JLS §17). Each Java Virtual Machine thread has its own pc (program counter) register. At any point, each Java Virtual Machine thread is executing the code of a single method, namely the current method (§2.6) for that thread. If that method is not native, the pc register contains the address of the Java Virtual Machine instruction currently being executed. If the method currently being executed by the thread is native, the value of the Java Virtual Machine's pc register is undefined. The Java Virtual Machine's pc register is wide enough to hold a returnAddress or a native pointer on the specific platform.
> 
> JVM은 한 번에 많은 쓰레드의 실행을 지원한다. 각 JVM의 쓰레드는 하나의 PC(program counter) 레지스터가 있으며 current method인 단일 메서드(current frame)의 코드를 실행한다. 이 메서드가 Native 메서드가 아니면 PC 레지스터에 현재 실행 중인 JVM의 명령어 주소가 있고, Native 메서드라면 값이 정의되지 않는다. JVM의 PC 레지스터는 returnAddress나 특정 플랫폼의 포인터를 보유하기에 충분히 넓다.  
> <sub>*JSR-392(2.5.1. The pc Register)*</sub>

JVM은 멀티 스레드를 지원하는데 각 스레드에는 PC(program counter) 레지스터가 있으며 현재 실행 중인 명령어의 주소가 할당되는데, 이 PC 레지스터는 returnAddress 값을 가질 수 있다.  

종합해보면, 호출된 메서드가 끝날 때 ret 명령을 통해 지역 변수 returnAddress의 값(호출자의 주소)을 가져와 PC 레지스터에 저장하고 return 명령을 통해 메서드를 종료하면서 PC 레지스터에 저장된 호출자의 주소를 이용하여 호출자의 메서드로 복귀해 실행을 이어나가게 되는 것이다.

따라서 <span class="emphasis-weak">JVM의 returnAddress 타입은 프로그램 실행 도중 호출된 메서드(서브 루틴)가 종료된 후 호출했던 메서드로 복귀하기 위한 주소값을 저장하는데 사용되는 타입</span>이다.


<div class="white-space--dot"></div>

## Reference Type
---
이번엔 Reference Type에 대해 알아보자. 다음은 Reference Type을 설명하는 부분에 나오는 내용이다.
> An object is either a dynamically allocated class instance or an array. A reference to an object is considered to have Java Virtual Machine type reference.
>
> 객체란 메모리에 동적으로 할당된 클래스의 인스턴스 혹은 배열이다. 객체에 대한 참조는 JVM의 타입 reference로 간주하며 타입 reference의 값은 객체의 포인터로 생각할 수 있다.  
> <sub>*JSR-392(2.2 Data Types)*</sub>

구현에 상세한 부분은 각 벤더사에게 맡기기 때문에 가능성을 열어두는 표현을 사용한 것으로 보인다.  
Java를 공부하면서 참조란 무엇인가에 대한 명확한 답이 떠오르지 않았었는데 이 부분을 읽고 의문이 해소되었다. 위 내용을 정리하면 다음과 같다.

<span class="emphasis-weak">객체란 메모리에 동적으로 할당된 클래스의 인스턴스 혹은 배열을 말하며 JVM의 참조 타입은 메모리에 할당된 객체의 주소값을 가진다.</span>

<div class="b-space" ></div>
### Pass by Reference
> Objects are always operated on, passed, and tested via values of type reference.  
> 객체들은 언제나 참조 타입의 값(주소값)들을 통해 동작되고 전달된다.  
> <sub>*JSR-392(2.2 Data Types)*</sub>  

이 문구를 보고 든 생각은 뭔가 추상적이라는 느낌이었다. 단순히 생각하면 주소값을 가진 참조가 변수들에 할당되면서 동작한다 정도로 이해해도 되겠지만 이 참조값이 정확히 어떻게 할당되고 파라미터로 전달되는지 더 명확히 이해하고 싶었다.

그래서 방법을 찾던 중 자바 바이트 코드를 디어셈블할 수 있는 javap 커맨드와 Intellij의 디어셈블 기능을 알게 되었다. 이제 위 문구처럼 동작하는 모습을 직접 눈으로 확인해보자. 백문이 불여일견이다.  

먼저 Java 파일을 작성하고 Intellij의 디어셈블 기능을 이용해 각 코드 라인의 명령어가 어떻게 처리되는지 상세히 살펴볼 것이다.  
*<span style="color: grey;">~~javap로 변환한 코드에 설명을 쓰다가 intellij로 변환한게 작성하기 더 편해서 바꿨다.~~</span>*  
만약 핵심만 빠르게 짚고 넘어가고자 한다면 [여기](/posts/jvm-data-types/#pass-by-reference)로 바로 넘어가자.  


<details>
<summary>
  <i style="font-size: 0.9rem; color: MediumOrchid;">바이트코드 디컴파일하는 법이 궁금하다면</i>
</summary><hr>
<pre style="font-size: 0.9rem; color: MediumOrchid;">
  .class파일은 javap 커맨드를 통해 디컴파일이 가능하다. Intellij에서도 비슷한 기능을 제공한다.
  다만, Intellij의 경우 상수 풀과 LineNumberTable에 대해 명확하게 표현되어 있는 편이 아니다.
  두 방식은 출력되는 형태에 차이가 있어 눈에 잘 들어오는 걸 쓰면 된다.

  - javap
    1. 자바 코드를 작성하고 javac로 컴파일한다.
    2. 생성된 class를 대상으로 javap 커맨드를 실행한다. 
       옵션에 따라 출력되는 내용이 다르기 때문에 javap --help를 참고하자.

  - Intellij
    1. 자바 코드를 작성하고 build를 통해 컴파일한다.
    2. 해당 파일을 선택하고 도구바에서 View -> Show ByteCode를 선택한다.

</pre></details><br/>

```java
// 
//
public class Service {

    public void start() {
        Member member = new Member();
        Member temp = member;
        int num = ageOf(temp);
    }

    private int ageOf(Member mem) {
        return mem.getAge(); // 회원의 나이인 20을 반환한다.
    }

}
```

아래 명령어 코드에서 언급되는 **피연산자 스택**은 **JVM 스택**(메서드가 호출될 때 적재되는 호출 스택)과 **다른 것**임을 주의한다. 그럼 다음 명령어 코드를 위 코드와 번갈아 확인하며 살펴보자.  

<details>
<summary>
  <i style="font-size: 0.9rem; color: MediumOrchid;">JVM 스택? 피연산자 스택?</i>
</summary><hr>
<pre style="font-size: 0.9rem; color: MediumOrchid;">
➤ JVM 스택
  메서드가 호출 될 때 프레임(Frame)이 생성되어 JVM 스택에 추가하고 매서드 종료 시 꺼내지며 소멸된다.  
  프레임은 지역 변수 배열, 피연산자 스택, 런타임 상수 풀에 대한 정보를 가진다.

➤ 피연산자 스택(Operand Stack)
  메서드 실행 중 필요한 상수나 값을 저장한다.
  명령어가 실행될 때 피연산자 스택에서 피연산자를 꺼내 연산하고 결과를 피연산자 스택에 추가한다.  

자세한 내용은 다음 포스팅 `자바 가상 머신의 원리와 이해 (2) - 런타임 데이터 영역`에서 설명한다.
</pre>
</details><br/>

```java
/**
 * 다음 코드를 읽을 때 LINENUMBER는 원본 소스의 줄 번호를 나타낸다.
 * 이 정보는 LineNumberTable이라는 속성에 저장되어 있다.
 * 
 * LineNumberTable은 배열의 인덱스에 해당하는 원본 소스의 행 번호가 변경되는 지점들을 담아놓는다.
 * 각 항목은 start_pc와 line_number값이 존재한다.
 * start_pc: 원본 소스의 새 줄에 대한 코드가 시작되는 배열 인덱스가 저장되어 있다.
 * line_number: 원본 소스의 줄 번호가 저장되어 있다.
 * 
 * Intellij로 변환한 코드는 LineNumberTable과 배열의 인덱스의 표현이 명확하지 않아 이해하기 어려울 수 있다.
 * 무슨 말인지 이해되지 않는다면 javap 커맨드로 디어셈블해 살펴보면 보다 쉽게 알 수 있을 것이다.
 * javap로 변환한 코드는 상수 풀에 대한 정보도 자세히 표현되므로 여유가 된다면 한번 쯤 확인해 보는 것도 좋다.
 **/
public class com/playground/my/Service {

  /**
   * Service 객체의 기본 생성자
   * 
   * 문서에서는 인스턴스 초기화 메서드라고 표현한다.
   **/
  public <init>()V
   L0
    LINENUMBER 3 L0
    /**
     * this의 값(Service의 참조)을 피연산자 스택에 push
     * 
     * 수퍼 클래스를 초기화 하기 위함이며 super 키워드는
     * Java언어 수준의 식별자이기 때문에 바이트 코드에서는 
     * 사용되지 않으며 이런 방식으로 동작한다.
     **/
    ALOAD 0
    /**
     * 피연산자 스택의 최상위 요소를 꺼낸 뒤 수퍼 클래스 Object의 생성자 호출
     **/
    INVOKESPECIAL java/lang/Object.<init> ()V 
    /**
     * 반환형이 void인 메서드의 반환 opcode.
     * 
     * 메서드가 종료되면서 프레임이 소멸되면서
     * 피연산자 스택 등의 정보도 모두 소멸된다.
     **/
    RETURN
   L1
    LOCALVARIABLE this Lcom/playground/my/Service; L0 L1 0
    MAXSTACK = 1   // 피연산자 스택의 크기. 컴파일 시점에 초기화된다
    MAXLOCALS = 1  // 지역 변수 배열의 크기. 컴파일 시점에 초기화된다

  public start()V
   L0
    LINENUMBER 6 L0
    NEW com/playground/my/Member // Member 객체를 생성하고 참조를 피연산자 스택에 push
    /**
     * 스택의 최상위 요소(여기서는 Member의 참조)를 복제한뒤 피연산자 스택에 push
     * 
     * 다음 명령어인 INVOKESPECIAL은 스택의 최상위 요소인 Member의 참조를 꺼내 생성자를 실행한다.
     * 하지만 생성자는 참조를 반환하지 않으므로 스택에 다시 참조를 넣을 수가 없다.
     * 이 참조는 다시 사용될 수 있기 때문에 미리 복제해 놓는 것이다.
     **/
    DUP
    // 스택의 최상위 요소를 꺼내서 Member의 생성자 실행
    INVOKESPECIAL com/playground/my/Member.<init> ()V
    ASTORE 1 // member 변수에 Member의 참조를 할당
   L1
    LINENUMBER 7 L1
    ALOAD 1  // member 변수의 값(Member의 참조)을 피연산자 스택에 push
    ASTORE 2 // 피연산자 스택의 최상위 요소(Member 객체의 참조)를 꺼내서 temp 변수에 할당
   L2
    LINENUMBER 8 L2
    ALOAD 0 // this의 값(Service의 참조)을 피연산자 스택에 push
    ALOAD 2 // temp 변수의 값을 피연산자 스택에 push
    /**
     * 피연산자 스택에서 this와 인스턴스 메서드 ageOf의 파라미터 개수만큼 pop한 뒤 ageOf에 인자를 넘기며 호출
     * 
     * 인스턴스 메서드 ageOf 전달받아야 하는 파라미터값 혹은 참조들과 this가 피연산자 스택에서 pop.
     * 인스턴스 메서드 ageOf 호출할 때 Member의 참조가 인자로 전달되며 새로운 프레임이 생성된다.
     * 새롭게 생성된 프레임은 호출 스택에 push 된다. (피연산자 스택이 아님을 주의)
     * 이 때, pc(program couter)에 ageOf메서드의 처음 명령어의 opcode가 설정 됨.
     **/
    INVOKEVIRTUAL com/playground/my/Service.ageOf (Lcom/playground/my/Member;)I
    ISTORE 3 // int형 변수 num에 ageOf의 int형 반환값을 저장
   L3
    LINENUMBER 9 L3
    RETURN // start 메서드 종료
   L4
    LOCALVARIABLE this Lcom/playground/my/Service; L0 L4 0
    LOCALVARIABLE member Lcom/playground/my/Member; L1 L4 1
    LOCALVARIABLE temp Lcom/playground/my/Member; L2 L4 2
    LOCALVARIABLE num I L3 L4 3
    MAXSTACK = 2
    MAXLOCALS = 4

  private ageOf(Lcom/playground/my/Member;)I
    // parameter  mem
   L0
    LINENUMBER 12 L0
    ALOAD 1 // 피연산자 스택에 파라미터 mem의 값(Member의 참조)을 push 
    /**
     * 피연산자 스택의 최상위요소(Member의 참조)를 꺼내 인스턴스 메서드 getAge 호출.
     * 반환형이 존재하는 메서드이므로 반환된 값은 피연산자 스택에 push
     **/
    INVOKEVIRTUAL com/playground/my/Member.getAge ()I
    /**
     * 반환형이 int형인 메서드의 반환 opcode
     * 피연산자 스택의 최상위 요소를 꺼내 반환
     **/
    IRETURN
   L1
    LOCALVARIABLE this Lcom/playground/my/Service; L0 L1 0
    LOCALVARIABLE mem Lcom/playground/my/Member; L0 L1 1
    MAXSTACK = 1
    MAXLOCALS = 2
}
```  

위 명령어 코드의 51 ~ 62라인을 보면 Member 객체를 생성한 후 객체의 주소값을 가진 참조를 피연산자 스택에 넣고, 다시 꺼내서 member 변수에 할당하는 과정을 확인할 수 있다. 이 때 객체는 heap 영역 메모리에 생성되는데 이와 관련된 자세한 내용은 [다음 포스팅](/posts/jvm-runtime-data-areas){:target="_blank"}에서 설명한다.  

그리고 72, 85라인을 보면 ageOf()를 호출할 때 전달되는 값은 Member 객체의 참조다.  
원본 코드에서 Member 객체가 할당된 변수들을 toString()으로 출력해보면 모두 동일한 참조를 가진 것을 확인 할 수 있다.
```java
com.playground.my.Member@5abca1e0 // member (start())
com.playground.my.Member@5abca1e0 // temp
com.playground.my.Member@5abca1e0 // member (ageOf())
``` 
우리는 앞서 참조 타입의 값은 객체의 메모리 주소라는 것을 확인했다. 따라서 위 모든 변수들은 동일한 메모리 주소를 갖고 있다.  

만약 ageOf()의 파라미터 member로 setAge 메서드를 호출해 나이 값을 20에서 10으로 바꾼 후, start()의 temp 변수로 나이를 출력해보면 10으로 출력된다. 동일한 메모리 주소의 객체를 참조하고 있기 때문에 발생한 현상이다.  

이러한 특성을 <span class="emphasis-weak">Pass by Reference</span>라고 하며 참조 타입을 다룰 때는 항상 이 점을 유념해야 한다.

<div class="b-space" ></div>

### Pass by Value
그럼 Primitive Type은 참조 타입과 동일하게 동작할까? 아니다. Primitive Type은 참조 타입과 다르게 참조가 아닌 값 자체가 전달된다. 이런  특성을 <span class="emphasis-weak">Pass by Value</span>라고 한다.  

명령어 코드를 통해 직접 확인해보자. 각 명령어의 자세한 사항은 위에서 어느 정도 설명했으므로 여기서는 필요한 부분만 설명한다. 마찬가지로 빠르게 핵심만 짚고 넘어가려면 [여기](/posts/jvm-data-types/#pass-by-value)로 바로 넘어가자.
```java
public class Service {

    public void start() {
        int age = 10;
        int temp = age;
        printAge(temp);
    }

    private void printAge(int n) {
        System.out.println(n);
    }

}
```
```java
public class com/playground/my/Service {

  public <init>()V
   L0
    LINENUMBER 3 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Lcom/playground/my/Service; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  public start()V
   L0
    LINENUMBER 6 L0
    BIPUSH 10 // 피연산자 스택에 숫자값 10을 push
    ISTORE 1 // 변수 age에 저장
   L1
    LINENUMBER 7 L1
    ILOAD 1 // 변수 age의 값을 가져와 피연산자 스택에 push
    ISTORE 2 // 피연산자 스택에서 pop한 값을 변수 temp에 저장
   L2
    LINENUMBER 8 L2
    ALOAD 0 // this의 값(Service의 참조)을 가져와 피연산자 스택에 push
    ILOAD 2 // temp의 값(int값 10)을 가져와 스택에 push
    /**
     * 피연산자 스택에서 this와 인스턴스 메서드 printAge 파라미터 개수만큼 pop한 뒤 printAge에 인자를 넘기며 호출
     * 여기서 인자는 int값 10이다.
     **/
    INVOKEVIRTUAL com/playground/my/Service.printAge (I)V
   L3
    LINENUMBER 9 L3
    RETURN
   L4
    LOCALVARIABLE this Lcom/playground/my/Service; L0 L4 0
    LOCALVARIABLE age I L1 L4 1
    LOCALVARIABLE temp I L2 L4 2
    MAXSTACK = 2
    MAXLOCALS = 3

  private printAge(I)V
    // parameter  n
   L0
    LINENUMBER 12 L0
    // 정적 클래스 영역에서 참조를 가져와 피연산자 스택에 push
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    ILOAD 1
    INVOKEVIRTUAL java/io/PrintStream.println (I)V
   L1
    LINENUMBER 13 L1
    RETURN
   L2
    LOCALVARIABLE this Lcom/playground/my/Service; L0 L2 0
    LOCALVARIABLE n I L0 L2 1
    MAXSTACK = 2
    MAXLOCALS = 2
}
```  

위 명령어 코드의 21, 22라인을 보면 참조가 아닌 값 자체를 피연산자 스택에 할당한 후 꺼내서 지역 변수 temp에 전달한다. 따라서 Primitive Type은 각각 독립적인 메모리 영역에 값을 갖게 되므로 특정 변수의 값의 변경이 다른 변수까지 영향을 주지 않는다.

<div class="white-space--dot"></div>

# 후기
이렇게 JVM의 데이터 유형 사양에 대해 알아보았다.  
JVM의 데이터 유형을 이해하기 위해 Java Compiling, 각 opcode의 동작, 운영체제의 CPU 명령어 처리 과정 등을 공부하면서 생각보다 훨씬 많은 시간을 쏟아붓게 되었다. 그 과정에서 실수 표준인 IEEE 754에 대해서도 작성하고 싶었는데, 내용이 길어질 듯 해 아예 독립적인 포스팅으로 작성할 생각이다.

분명한 것은 데이터 유형에 대해 공부하기 이전의 나보다 지금의 내가 자바 코드에 대해 깊게 이해하고 있다는 것이다.
그리고 무엇보다 만족스러운 점은, 그동안 이론으로만 공부했던 운영 체제의 지식이 내가 작성하는 자바 코드에 어떻게 녹아 있는지 알게 된 것이다.