---
title: 직관적인 Vue 아키텍처 설계
date: 2023-09-14
categories: [Vue]
tags: [Vue, Architecture]
---

이번에 진행한 프로젝트는 기존 JSP 환경에서 프론트엔드 프레임워크를 도입해 생산성을 늘려보자는 취지로 사내에서 처음 Vue를 도입한 프로젝트였다.
Vue를 사용해 본 사람이 필자 밖에 없는 상태에서 시작했던 터라 1년 전 설계에 많은 고민을 했던 기억이 나 프로젝트가 성공적으로 끝난 지금 간단한 포스팅으로 회고해본다.

## 기존 환경의 문제
회사에서 진행하는 프로젝트들의 오류는 클라이언트 영역에서 가장 많이 발생했다. 특히 수없이 반복되는 비슷비슷한 마크업, CSS 충돌, 화면마다 수없이 존재하는 비슷비슷한 코드들의 오타 등 사소한 실수들이 기능 자체에 영향을 끼치는 일이 비일비재했다. 예를 들어, form element의 clear()가 hidden 필드는 초기화시키지 않는다는 사실을 간과하거나 하는 등의 실수다. 문제는 이같은 원인들이 금방 발견되어 해결되면 좋겠지만 신입이 주를 이루는 우리 팀의 여건 상 결국 <span class="very-danger">수많은 야근과 일정 지연이 발생</span>한다는 점이었다.

입사 후 첫 프로젝트에서 상기의 문제점을 인지하고 두 번째 프로젝트를 준비할 때 프론트엔드 프레임워크의 도입을 팀장님께 요청드렸다. 

## Vue 도입
마침 경영진에서도 Angular의 도입을 검토 중이었기 때문에 Vue에 대한 논의도 자연스럽게 함께 이루어졌다. React, Vue는 경험이 있었지만 Angular는 생소했기에 약 3주간 공식 문서를 읽으며 미니 프로젝트를 진행했고 다음과 같은 사유로 Vue가 가장 적합하다는 결론을 내렸다. 

- <span class="emphasis-weak">JSP 환경에 익숙한 팀원들이 보다 쉽게 습득할 수 있다.</span>
- <span class="emphasis-weak">새로 조인한 팀원이 담당자가 되었을 때 마찬가지 이유로 쉽게 접근이 가능하다.</span>

단순하지만 신입이 대다수이고 프로젝트 일정 준수가 중요한 상황에서 무엇보다 중요한 이유였다. 
이후 기존에 존재하면 화면을 Angular, React, Vue의 코드로 작성하여 비교 및 공유하였고 Vue를 도입하게 되었다.

## 설계의 중점
Vue 도입을 기뻐하기도 잠시, 프로젝트 킥오프 일정에 맞춰 설계를 완료하고 팀원들이 즉시 개발할 수 있도록 만들어야 했다. 가장 큰 문제는 팀원들에게 Vue에 대한 학습 기간이 주어지지 않는 일정이었기 때문에 최악의 경우, Vue에 대해 아무것도 모르는 상태로 개발을 시작할 수도 있는 상황이었다. 팀원들의 개인 시간에 공부해오기를 종용할 수 없기에 최악의 경우를 대비해야 했다.

요점은 <span class="emphasis">Vue를 몰라도 개발에 지장이 없어야 한다</span>였다. 하지만 전혀 모른채로 개발을 할 수는 없기에 현실적으로<span class="emphasis">Basic한 문법 몇 가지만 알아도 개발을 할 수 있는 **직관적인 프로젝트**</span>라는 명제를 세운 후 이를 달성하기 위해 고민했다.

## 프로젝트 구조
가장 중요하게 생각한 부분은 <span class="emphasis">팀원들이 화면의 각 컴포넌트 간 데이터 흐름을 직관적으로 쉽게 이해 하는 것</span>이었고 Container / Presentational 패턴을 적용했다. *Dan Abramov*가 소개한 이 패턴은 [React Hooks가 등장한 이후 별다른 이유없이 사용하지 말라](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)고 했지만 데이터의 흐름을 직관적으로 가장 쉽게 이해할 수 있는 패턴이라고 생각한다. 

따라서 최종적으로 다음과 같은 가이드 라인을 정하고 프로젝트 구조를 설계했다.

### 설계 원칙
- Vue 3를 사용하지만 Vue 2 문법 만을 사용
    > + Composition API 등의 문법은 러닝 커브가 높음  
    > + Reference가 많다
- Container / Presentational 패턴
- 상태 관리 및 외부 데이터는 Container 컴포넌트에서 관리
- Presentational Component는 데이터를 Container 컴포넌트에서 주입
- Presentational Component는 최대 3 depth. 가급적 2 depth를 넘기지 않도록 구성
    > + depth가 깊어지면 복잡도가 급격히 상승
- 컴포넌트간 통신은 반드시 props down, events up
    > + 명시적으로 데이터의 흐름을 표현하여 흐름을 추적하기 용이
- UI를 구성하는 정적 데이터는 Container 컴포넌트와 분리하여 관리
    > + Form의 각 label, 그리드의 컬럼명, 메뉴 제목, 버튼 목록 등 렌더링된 이후 변하지 않는 데이터
    > + 코드의 성격을 명확히 구분해 관리하며 가독성 및 생산성 향상
- API, 외부 플러그인들은 인터페이스를 추상화해 구현이 달라지더라도 비즈니스 코드에 영향이 없도록 처리
    > + 사용하는 라이브러리가 변경되어도 플러그인을 사용한 코드에 영향이 가지 않음
- 전자 결재 등 특정 도메인에 대한 로직은 컴포넌트와 별도로 분리

### 디렉토리 구성
```javascript
- api/
- component/         // Presentational Components
- domain/            // 특정 도메인 로직 관리
- page/              // Container Components
  - sample/
    - values/
      - sample.js
    - Sample.vue
  ...
- plugin/            // 외부 라이브러리 및 공통, 도메인 로직의 플러그인 처리
- router/
- store/             // vuex
...
```

### Data Flow
![img-description](/assets/img/posts/vue-declarative-architecture/flow.png){:target="_blank"}  

<div class="white-space--dot"></div>

## 직관적인 개발
앞서 언급했듯이 프로젝트 설계의 중점은 <span class="emphasis">Vue를 잘 모르더라도 개발이 가능한 직관적인 프로젝트</span>를 만드는 것이었지만 문제가 발생했다. 이를 설명하기 앞서 각 컴포넌트의 역할은 다음과 같다.

- **Container Component**  
Stateful하며 1개의 메뉴다. 해당 메뉴에 필요한 데이터를 호출하고 관리하며 인터렉션을 처리하는 역할을 담당

- **Presentational Component**  
Stateless하며 Container 컴포넌트로부터 데이터를 주입받아 렌더링하고 이벤트를 통해 다시 Container 컴포넌트로 데이터를 보내는 역할을 담당한다.

<div class="b-space"></div>

Presentational Component이 주입받는 데이터는 크게 서버에서 가져오는 동적 데이터와 렌더링 이후 변하지 않는 정적 데이터로 나눌 수 있고 이 데이터들은 props로 주입받아 events를 통해 상위 컴포넌트로 전달된다.

문제는 <span class="very-danger">동적, 정적 데이터를 정의하고 컴포넌트에 전달하며 핸들링하려면 Vue의 다양한 문법에 대한 이해가 필수였다.</span> 물론 문법을 전혀 모른채로 개발할 수는 없지만 이런 지식들의 <span class="emphasis-weak">학습을 최소화할 수 있게 컴포넌트를 추상화해 제공했다.</span> 


### 예시 1

![img-description](/assets/img/posts/vue-declarative-architecture/container-component-view-1.png){:target="_blank"}  

```javascript
const sample = {
  static: {
    title: '예제',
    countPerRow: 4,
  },
  buttons: () => [
      { name: 'request', label: '의뢰' },
      { name: 'select', label: '조회', type: 'search' },
  ],
  forms: () => FormBuilder.builder()
      .Hidden('code')
      .Input('name', '이름')
        .required().validator(value => value !== '')
      .InputNumber('age', '나이')
        .readonly()
      .Select('year', '연도')
        .disabled()
      .multiple('capacity', '용량', FormBuilder.builder()
          .Input('amount')
            .spanCol(5)
          .Input('unit')
            .spanCol(5)
          .build(),
      )
      .Datepicker('birth', '출생일자', { value: today })
        .spanCol(2).spanRow(2)
      .DatepickerTwinWithSwitch('period', '기간', { value: [weekAgo, today] })
        .spanCol(2)
      .Textarea('contents', '내용', { rows: 2 })
        .spanCol(4)
      .build(),
  columns: () => ColumnBuilder.builder()
      .col('id', 'ID')
      .col('info', '개인정보', {
        children: ColumnBuilder.builder()
          .col('age', '나이')
          .col('birth', '출생일자')
          .build(),
      })
      .build(),
};
```

```javascript
<template>
  <SearchGrid
    v-bind="sample"
    @button-click="onButtonEvent"
    @enter="search"
  />
</template>
```


### 예시 2
![img-description](/assets/img/posts/vue-declarative-architecture/container-component-view-2.png){:target="_blank"}  

```javascript
<template>
  <SearchGrid
    v-bind="product"
    @button-click="onProductButtonEvent"
  />

  <Horizontal :spans="[5, 1, 12]">
    <GridWithHeader
      v-bind="cart"
    />

    <ExchangePanel
      direction="vertical"
      @click="onExchangeEvent"
    />

    <GridWithHeader
      v-bind="item"
    />
  </Horizontal>
</template>
```

빌더라는 유틸 클래스를 만들어 정적 데이터와 유효성 체크 등을 수행하고, 빌더를 통해 만들어진 객체를 핸들링하는 편의 메서드들을 제공해 빠르고 오류없이 개발할 수 있도록 했다. 이는 코드를 간소화하고 가독성을 향상시키는 효과도 가져왔다.

## 결과
Vue 도입을 결정하고 나서도 다들 내심 걱정했다는 말이 무색하게 개발이 무난히 진행됐으며 도입하기를 잘했다는 평가도 나왔기에 성공적인 시도가 아니었나 싶다. 도입부에 언급했던 문제들이 전부 해결되어 오류 발생률과 개발 속도가 크게 개선됐는데, <span class="emphasis">클라이언트 오류는 서버 API 호출 관련을 제외하면 거의 발생하지 않았으며 화면 개발 속도도 기존에 평균 5일 정도가 소요되던 것에 비해 평균 3일 가량으로 개선됐다.</span>