---
title: Spring Boot 3 API Documentation (Spring REST Docs & Swagger UI)
date: 2023-05-14
categories: [Spring]
tags: [Java, Spring Boot 3, Spring REST Docs, Swagger UI, API Documentation]
---

사이드 프로젝트의 환경을 구성하며 API 문서화를 위해 Swagger와 Spring REST Docs 사이에서 고민했다. Swagger는 컨트롤러 코드에 문서화 코드가 적지 않게 들어가는 것을 경험했던 터라 Spring REST Docs를 사용하는 것으로 마음이 기울었는데, 결국 **Spring REST Docs와 Swagger를 함께 사용**하게 되었다. 둘 다 단독으로 쓰기에는 아쉬운 단점이 있기 때문이다.

<br/>

## [<span class="link">Swagger UI</span>](https://swagger.io/tools/swagger-ui/){:target="_blank"}
Swagger UI는 대표적인 API 문서화 도구이며 API를 테스트해 볼 수 있는 기능도 제공한다.  
코드를 굳이 작성하지 않아도 Swagger UI는 기본적인 API 문서를 생성해준다. 그걸로 충분하다면 괜찮겠으나 개발하다 보면 다양한 정보를 작성할 필요가 생기며 어느샌가 다음과 같은 모습을 마주치게 된다.  

![img-description](/assets/img/posts/spring-docs-swagger/swagger1.png){:target="_blank"}  

비즈니스 코드와 관련없는 문서화 코드가 많은 영역을 차지해 핵심 코드를 알아보기 힘들게 만든다. 더불어 개발자가 비즈니스 코드를 변경한 뒤 문서화 코드에 반영하는 것을 지나치면 **<span class="danger">문서와 기능이 불일치하는 상황이 발생</span>**한다.

<br/>

## [<span class="link">Spring REST Docs</span>](https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/){:target="_blank"}

스프링에서 제공하는 API 문서화 도구이며 asciidoctor나 markdown 형식을 지원한다.  
특징은 문서화 코드를 **테스트 코드로 작성한다**는 점이며 실제 API의 요청, 응답 객체와 다를 경우 테스트가 실패한다. 따라서 **기능과 문서의 동기화를 보장**한다. 이 점이 Spring REST Docs로 마음이 기울었던 이유다. 하지만 Spring REST Docs에도 아쉬운 점이 존재했다.  
- 빌드 시 생성된 문서 파일들을 별도의 [템플릿 파일에 작성하여 관리](https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/#getting-started-using-the-snippets){:target="_blank"}
- API 테스트 기능이 없음. 따라서 curl, postman, httpie같은 도구를 별도 사용.

<br/>

## SwaggerUI vs Spring REST Docs

<table style="border: 2px;">
  <tr>
    <td></td>
    <td> <b>SwaggerUI</b> </td>
    <td> <b>Spring REST Docs</b> </td>
  </tr>
  <tr>
    <td> <b>장점</b> </td>
    <td style="width:100px;">
        - API 테스트 기능 <br/>
        - 컨트롤러와 문서 내용을 한 눈에 확인 가능
    </td>
    <td>
        - 테스트를 통한 정확성 보장 <br/>
        - 문서화 코드를 추상화할 수 있다.
    </td>
  </tr>
  <tr>
    <td> <b>단점</b> </td>
    <td>
        - 핵심 코드가 가려질 수 있다. <br/>
        - 기능과 문서화의 불일치가 발생할 수 있다.
    </td>
    <td>
        - 별도의 템플릿을 관리해야 함 <br/>
        - API 테스트는 별도의 도구를 써야 함.
    </td>
  </tr>
</table>

위 단점들이 너무 아쉬워 장점만 취할 수는 없을까 고민하던 중, 앞서 같은 문제로 고민했던 좋은 레퍼런스들을 통해 방법을 찾게 되었다. 다만, 레퍼런스들은 Spring Boot 2.x 환경이었기 때문에 이번 글을 통해 Spring Boot 3.x 환경에 적용하는 방법을 알아본다.

<br/>

## SwaggerUI와 Spring REST Docs 통합

SwaggerUI와 Spring REST Dodcs를 통합하는 방법은 다음과 같다.

1. 오픈소스 [<span class="link thick">restdocs-api-spec</span>](https://github.com/ePages-de/restdocs-api-spec){:target="_blank"}을 이용해 Spring REST Docs의 코드를 [<span class="link thick">openapi</span>](https://www.openapis.org/){:target="_blank"} 사양의 파일로 생성한다.
2. 생성된 파일의 경로를 swagger의 설정 파일 경로로 지정한다.

간단한 방법이니 금방 적용할 것이란 기대와 다르게 문제가 발생했는데, 이 문제에 대해서는 마지막에 확인하기로 하고 우선 통합하는 방법을 알아보자.  
<span class="weak">(문제를 해결하려고 한나절을 고생한 끝에 어이없을 정도로 허무한 결말을 맞았다.)</span>  

<div class="b-space"></div>

### 0. Dependencies
현재 프로젝트의 의존성 구성은 다음과 같다.  
- java 17
- spring boot 3.1.0
- spring-restdocs-mockmvc
- restdocs-api-sepc 0.18.2
- springdoc-openapi-starter-webmvc-ui:2.1.0

>[<span class="link thick">Spring REST Docs</span>](https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/#getting-started-documentation-snippets){:target="_blank"}는 MVC 단위 테스트에 적합한 <span class="soft">MockMvc</span>, WebFlux의 테스트도 지원하는 <span class="soft">WebTestClient</span>, 통합 테스트에 적합한 <span class="soft">REST Assured</span>를 소개한다. 현재 프로젝트는 MVC기반이고 API는 단위 테스트만 작성할 것이라 <span class="soft">MockMvc</span>를 선택했다.  

>[<span class="link thick">restdocs-api-spec</span>](https://github.com/ePages-de/restdocs-api-spec){:target="_blank"}의 공식 문서는 다음과 같이 버전을 안내하고 있다.
> ![img-description](/assets/img/posts/spring-docs-swagger/apispec1.png){:target="_blank"}  


<div class="b-space"></div>

### 1. build.gradle에 의존성 추가
```gradle
buildscript {
    ext {
        restdocsVersion = '0.18.2'
    }
}

plugins {
    ...
    id 'com.epages.restdocs-api-spec' version "${restdocsVersion}"
}

...

repositories {
    mavenCentral()
}

dependencies {

    ...
    
    // swagger
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.1.0'


    ...

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc'
    testImplementation "com.epages:restdocs-api-spec-mockmvc:${restdocsVersion}"
}
```

<div class="b-space"></div>

### 2. build.gradle에 openapi3 및 task 설정
```gradle
tasks.named('test') {
    useJUnitPlatform()
}

// Spring REST Docs의 코드를 openapi3 사양의 파일로 생성하기 위한 정보 입력
openapi3 {
    server = 'http://localhost:8080'
    description = 'API description'
    version = '0.1.0'
    outputFileNamePrefix = 'openapi3'
    outputDirectory = "${project.buildDir}/resources/main/static/docs"
    format = 'yaml'
}

// Local 환경에서 접근하기 위해 프로젝트 경로로 복사
tasks.register('generateOpenApi') {
    dependsOn 'clean'
    dependsOn 'openapi3'

    doFirst {
        delete file('src/main/resources/static/docs/')
    }

    doLast {
        copy {
            from "${project.buildDir}/resources/main/static/docs/"
            into 'src/main/resources/static/docs/'
        }
    }
}
```
> 서버 실행 전 generateOpenApi를 실행하면 항상 최신 API 문서를 생성할 수 있다.  
> [IntelliJ] Run/Debug Configurations >> Before launch >> Run Gradle task >> generateOpenApi

<div class="b-space"></div>

### 3. swagger 설정
마지막으로 swagger에 openapi 파일 경로를 지정해주면 완료된다.
```yaml

...

springdoc:
  swagger-ui:
    url: /docs/openapi3.yaml
```


<br/>

## 어떻게 달라졌을까?
앞서 봤던 Swagger 코드를 Spring REST Docs 코드로 변경하면 컨트롤러에는 핵심 코드만 남게 된다.
![img-description](/assets/img/posts/spring-docs-swagger/springdocs1.png){:target="_blank"}  

그리고 테스트 코드에 문서화 코드가 존재하게 되었고 컨트롤러를 변경할 때마다 동기화하지 않으면 테스트가 깨지게 되어 문서화의 정확성이 보장되었다.  

![img-description](/assets/img/posts/spring-docs-swagger/springdocs2.png){:target="_blank"}  
> 위 코드는 공통 부분을 추상화한 코드이기 때문에 정확한 작성 방법은 쉽게 설명되어 있는 [공식 문서](https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/#documenting-your-api){:target="_blank"}를 참고하자.  

또한, Swagger UI로 문서를 볼 수 있어 API 테스트 기능도 사용할 수 있게 되었다.  

![img-description](/assets/img/posts/spring-docs-swagger/springdocs3.png){:target="_blank"}  

이렇게 Swagger와 Spring REST Docs을 통합함으로써 각 도구의 장점들을 얻을 수 있었다.  

<br/>

## '그' 문제
중간에 언급했던 허무했던 결말을 맞은 '그' 문제를 남긴다. 나를 제외하고 이런 실수를 하는 분은 없을 것이다.  

SwaggerUI와 Spring REST Docs를 통합하는 방법을 찾은 뒤 적용을 했다. 그랬더니 클래스를 찾지 못하는 게 아닌가?  
<span class="weak">당시 코드를 따로 캡처하지 못해 임시 코드로 대체해 재현했다.</span>

![img-description](/assets/img/posts/spring-docs-swagger/problem-1.png){:target="_blank"}  

공식 문서에 안내하는 대로 적용했음에도 불구하고 에러가 사라지지 않아서 도대체 왜 에러가 나는지 감이 잡히지 않았다.  

Gradle에 모듈이 정확히 인식되어 있는지, IntellJ 프로젝트 세팅에 인식이 제대로 되어 있는지 체크도 해보고 캐시도 날려보고 온갖 처리를 해봐도 그대로였다. [restdocs-api-spec의 Issuse](https://github.com/ePages-de/restdocs-api-spec/issues){:target="_blank"}를 뒤져 [Spring Boot 3.x 호환 문제](https://github.com/ePages-de/restdocs-api-spec/issues/211){:target="_blank"}가 있었지만 [해결된 것](https://github.com/ePages-de/restdocs-api-spec/pull/225){:target="_blank"}도 확인했다.  

그러던 중 Gradle에 모듈이 정확히 인식되어 있던 것을 다시 기억해내고 혹시나 하는 마음에 테스트 코드를 실행해보니...
![img-description](/assets/img/posts/spring-docs-swagger/problem-2.png){:target="_blank"}  
잘 통과하는 것이 아닌가?  

그렇다. 원인은 IntelliJ의 버전 문제였다. 작업 중인 데스크탑의 IntelliJ는 2021 버전이었고 지금껏 업데이트를 하지 않고 있었다. 하지만 IntelliJ는 2022 버전부터 Spring Boot 3 버전을 정식으로 지원하니 이런 문제가 발생한 것이었고 IntelliJ의 버전을 업데이트하는 것으로 해결되었다.


# 참고
- [Spring REST Docs](https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/){:target="_blank"}
- [restdocs-api-spec](https://github.com/ePages-de/restdocs-api-spec){:target="_blank"}
- [Spring Rest Docs 적용(우아한 기술블로그)](https://techblog.woowahan.com/2597/){:target="_blank"}
- [SwaggerUI + Spring REST Docs 함께 사용하기(feat. Rest Assured)](https://jwkim96.tistory.com/274){:target="_blank"}

