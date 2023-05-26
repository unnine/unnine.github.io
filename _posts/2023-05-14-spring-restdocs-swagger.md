---
title: Spring Boot API Documentation (Spring REST Docs & Swagger UI)
date: 2023-05-14
categories: [JAVA]
tags: [Java, Spring Boot, Spring REST Docs, Swagger UI, API Documentation]
---

# 개요
도메인 주도 설계로 진행하는 펫 프로젝트에서 API 문서화를 어떻게 할까 고민하던 와중 REST Docs와 Swagger를 함께 적용하게 되었다.  

과거 Swagger를 적용했을 때, 비즈니스 코드보다 Swagger에 관한 코드가 훨씬 많이 들어가는 것을 경험했던 터라 다른 좋은 방안이 없나 살펴보던 과정에서 Spring REST Docs를 알게 되었다.  

결론부터 말하자면 **Spring REST Docs와 Swagger를 함께 사용**하는 방식으로 결정하게 되었는데, 그 이유는 각 라이브러리가 단독으로 쓰기에는 뭔가 하나씩 아쉬웠기 떄문이다.


# Swagger와 REST Docs
## [Swagger UI](https://swagger.io/tools/swagger-ui/){:target="_blank"}
Swagger UI는 대표적인 API 문서화 도구이며 작성된 API에 해당하는 컨트롤러로 요청을 전송할 수 있는 기능도 제공한다.  

[springdoc-openapi](https://github.com/springdoc/springdoc-openapi){:target="_blank"}를 사용하면 별다른 코드 작성 없이도 Swagger UI로 기본적인 API 문서를 자동으로 생성해준다. 그 정도의 문서로 충분하다면 괜찮겠으나...  

실제로 개발하다 보면 그 외에 다양한 정보를 작성해야 한다. 그리고 이렇게 작성하다 보면 다음과 같은 모습을 마주치게 된다.  

![img-description](/assets/img/posts/spring-docs-swagger/swagger1.png){:target="_blank"}  

얼핏 봐도 문서화에 대한 코드가 핵심 코드에서 시선을 뺏어버리는 안타까운 상황이다.  

비즈니스 코드에 전혀 관련없는 코드가 많은 영역을 차지해 핵심 코드를 알아보기 힘들게 만들며, 컨트롤러를 수정한 후 개발자가 수정하는 것을 까먹으면 API 문서와 실제 기능이 불일치하는 상황이 발생한다.  

<br/>

## [Spring REST Docs](https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/){:target="_blank"}
스프링에서 제공하는 API 문서화 도구인데 asciidoctor나 markdown 형식으로 문서화가 가능하다.  

API문서를 구성하는 Snippet을 **테스트 코드로 작성해야 한다**는 점이 특징인데, 비즈니스와 관련된 테스트 코드와 Snippet에 대한 테스트 코드가 모두 통과되지 않으면 **테스트가 실패함으로써 문서의 정확성을 보장**한다. 이 점이 Spring REST Docs를 사용해야 겠다고 결정한 가장 큰 이유다.  

하지만 역시나 Spring REST Docs에도 아쉬운 점이 존재했다.
- 빌드 시 생성된 문서 파일들을 별도의 [템플릿 파일에 작성해서 관리](https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/#getting-started-using-the-snippets){:target="_blank"}해야 함
- API 호출 기능이 없음. 따라서 curl, postman, httpie같은 도구를 따로 활용해야 함.

테스트 코드를 강제하게 하는 점은 좋았으나, 별도의 템플릿 관리가 필요하고 API 호출 기능이 없어 따로 Postman같은 도구를 사용해야 하는 점이 아쉬웠다.  

<br/>

## Swagger vs Spring REST Docs 정리

<table style="border: 2px;">
  <tr>
    <td></td>
    <td> <b>Swagger</b> </td>
    <td> <b>Spring REST Docs</b> </td>
  </tr>
  <tr>
    <td> <b>장점</b> </td>
    <td style="width:100px;">
        - API 직접 호출 기능 <br/>
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
        - API 직접 호출은 별도의 도구를 써야 함.
    </td>
  </tr>
</table>

<div class="white-space--dot"></div>

# Swagger + Spring REST Docs 
각 도구의 장점만 취할 수는 없을까 고민하던 중, 앞서 이런 고민을 해결했던 좋은 선례들이 있어 이를 참고하여 적용해 보았다.  

Spring REST Docs는 단위 테스트에 적합한 `MockMvc`, WebFlux나 스트리밍의 테스트도 지원하는 `WebTestClient`, 통합 테스트에 적합한 `REST Assured`가 있지만, 현재 프로젝트에서 컨트롤러는 단위 테스트만 수행할 것이기 때문에 `MockMvc`를 선택했다.  

먼저 환경 구성은 다음과 같다.  

- Java 17
- Spring Boot - 2.7.11
- restdoc-apis-spec - 0.16.4
- springdoc-openapi-ui - 1.7.0

## 1. build.gradle에 의존성 추가
```gradle
buildscript {
    ext {
        restdocsVersion = '0.16.4'
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
    implementation(
        ...

        'org.springdoc:springdoc-openapi-ui:1.7.0', // swagger 의존성
    )

    ...

    testImplementation(
        'org.springframework.boot:spring-boot-starter-test',
        'org.springframework.restdocs:spring-restdocs-mockmvc',
        "com.epages:restdocs-api-spec-mockmvc:${restdocsVersion}"
    )
}
```

<br/>

## 2. build.gradle에 openapi3 및 task 설정
프로젝트 빌드 시 API 문서를 생성하도록 task를 추가한다. 또한, 개발 진행 시 로컬 환경에서도 확인할 수 있도록 bootRun할 때 프로젝트 소스로 copy한다.  

```gradle
tasks.named('test') {
    useJUnitPlatform()
}

// Spring REST Docs의 결과를 openapi3 명세에 맞게 출력하기 위한 정보 입력
openapi3 {
    server = 'http://localhost:8080'
    description = 'Spring REST Docs with SwaggerUI.'
    version = '0.1.0'
    outputFileNamePrefix = 'openapi3'
    outputDirectory = "${project.buildDir}/resources/main/static/docs"
    format = 'yaml'
}

// Local 환경에서 확인하기 위해 프로젝트 소스로 복사
task generateOpenApi {
    dependsOn 'openapi3'

    delete file('src/main/resources/static/docs/')
    copy {
        from "${project.buildDir}/resources/main/static/docs"
        into "src/main/resources/static/docs/"
    }
}

bootRun {
    dependsOn 'generateOpenApi'
}

bootJar {
    dependsOn 'openapi3'
}
```

이 후 gradle reload를 해주면 설정이 완료된다.

<pre style="font-size: 0.9rem; color: MediumOrchid;">
➤ 스프링이 실행되기 전에 먼저 Build를 실행하게 하면 항상 최신 API 문서를 확인할 수 있다.  
(IntelliJ: Run/Debug Configurations >> Before launch >> Run Gradle task: build)
</pre>

<br/>

## 어떻게 달라졌을까?
이제 기존 Swagger로 작성된 코드를 Spring REST Docs에 맞게 수정한 뒤 차이를 보자.  

앞서 Swagger 파트에서 보았던 컨트롤러는 다음과 같이 핵심 코드만 남게 되었다.  

![img-description](/assets/img/posts/spring-docs-swagger/springdocs1.png){:target="_blank"}  

<br/>

그리고 문서화 코드는 테스트 코드로 이동되었고 컨트롤러가 변경되었을 때 문서화 코드도 그에 맞게 변경하지 않으면 테스트가 깨지게 되어 문서화의 정확성을 보장하게 되었다.  

> 아래 코드는 공통 부분을 추상화한 코드이기 때문에 정확한 작성 방법은 쉽게 설명되어 있는 [공식 문서](https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/#documenting-your-api){:target="_blank"}를 참고하자.

![img-description](/assets/img/posts/spring-docs-swagger/springdocs2.png){:target="_blank"}  

<br/>

또한, Swagger UI로 문서를 볼 수 있어 API 호출 기능 같은 유용한 기능도 사용할 수 있게 되었다.  

![img-description](/assets/img/posts/spring-docs-swagger/springdocs3.png){:target="_blank"}  


# 정리
 이렇게 Swagger에서 아쉬웠던 핵심 코드를 가리는 문제, 기능과 문서화의 불일치가 일어날 수 있는 문제를 해결하였다.  
 또한, Spring REST Docs에서 별도의 템플릿을 관리해야 하는 문제, API 직접 호출이 불가능한 문제를 해결하여 각 도구의 장점 만을 얻을 수 있었다.  
 
 다만, 한 가지의 도구만 사용할 지, 통합하여 사용할 지는 항상 상황에 맞게 판단하는 것이 좋다고 생각한다.  

# 참고
- [Spring REST Docs](https://docs.spring.io/spring-restdocs/docs/3.0.0/reference/htmlsingle/){:target="_blank"}
- [restdocs-api-spec](https://github.com/ePages-de/restdocs-api-spec){:target="_blank"}
- [Spring Rest Docs 적용(우아한 기술블로그)](https://techblog.woowahan.com/2597/){:target="_blank"}
- [Generate Swagger UI from Spring REST Docs](https://blog.jdriven.com/2021/10/generate-swagger-ui-from-spring-rest-docs/){:target="_blank"}
- [SwaggerUI + Spring REST Docs 함께 사용하기(feat. Rest Assured)](https://jwkim96.tistory.com/274){:target="_blank"}

