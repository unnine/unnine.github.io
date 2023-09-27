---
title: Join 방식에 따른 비용, 성능 변화
date: 2023-08-21
categories: [Database]
tags: [Database, SQL, Join, Nested Loop Join, Hash Join]
---

## 개요
최근 업무 중 특정 메뉴의 목록이 출력되지 않는다는 문의를 받았다.
확인해보니 해당 목록을 가져오는 요청의 평균 응답 시간이 매우 느린 상태였다. 확인 결과 슬로우 쿼리와 페이징 처리없이 대량의 데이터를 가져와 매퍼에서 객체로 생성하는 시간이 오래 걸리는 문제였고 결국 쿼리 튜닝과 페이지네이션을 적용해 문제를 해결했다.

이 때 쿼리 튜닝은 조인 방식을 변경하여 성능을 향상시키게 되었는데 이와 관련해 간단하게 정리한다.

<div class="white-space--dot"></div>

데이터베이스는 Oracle이며 튜닝의 핵심이 되었던 테이블은 총 4개이고 연관 관계는 다음과 같다.  
- *ITEM* : *ITEM_INFO* (1 : 1)
- *ITEM* : *ITEM_INFO_SAP* (1 : 1)
- *ITEM* : *ITEM_SPEC* (1 : N)  

> 각 테이블의 CODE, NO 컬럼은 복합키로 인덱스가 생성되어 있다.

## 기존 쿼리
> 예제에 있는 테이블명 및 컬럼명은 임시로 작성했으며 튜닝에 핵심이 되었던 부분만 작성한다.

```sql
SELECT 
    ...
FROM ITEM I             -- 약 30만건의 데이터
JOIN ITEM_INFO INFO     -- 약 30만건의 데이터
    ON I.PLANT = INFO.PLANT AND I.NO = INFO.NO
JOIN ITEM_INFO_SAP SAP  -- 약 30만건의 데이터
    ON I.PLANT = SAP.PLANT AND I.NO = SAP.NO
JOIN (
    SELECT
        MAX(VERSION) AS FINAL_VERSION,
        ...
    FROM ITEM_SPEC -- 약 40만건의 데이터
    GROUP BY
        ...
) SPEC
    ON I.PLANT = SPEC.PLANT AND I.NO = SPEC.NO
WHERE
    I.PLANT = #{plant}
AND I.USE = 'Y'
```
위 쿼리의 실행 계획을 조회하면 다음과 같이 나타났다.

> 1. PLANT와 USE로 필터링된 ITEM(I)과 GROUP BY의 결과로 생성된 인라인 뷰(SPEC)가 Hash Join
> 2. 1번의 결과셋과 ITEM_INFO_SAP(SAP)과 Nested Loop Join
> 3. 2번의 결과셋과 ITEM_INFO(INFO)와 Nested Loop Join

- 
1번 과정에서 ITEM 테이블은 PLANT와 USE 컬럼으로 필터링을 거치게 되는데 플랜트 데이터의 종류는 2개이고 데이터의 비율이 동일하기 때문에 15만건이 도출된다.

- 
2번 과정에서는 1번의 결과셋이 Driving 테이블이 되어 NL Join을 수행하게 된다.

- 
3번 과정에서도 2번 과정의 결과셋이 Driving 테이블이 되어 NL Join을 수행한다.

### 개선 방안
2, 3번 과정의 JOIN 컬럼은 복합키이기 때문에 인덱스 유니크 스캔이 일어난다. 

하지만 15만건이나 되는 Driving 테이블에서 30만건의 Driven 테이블로 랜덤 액세스가 발생하기 때문에 성능이 좋지 않을 것이라 생각했다. 

<span class="emphasis">따라서 NL Join으로 수행되는 부분을 Hash Join으로 유도하면 성능이 향상될 것으로 예상했다.</span> 다만, Hash Join으로 변경하면 CPU와 메모리를 더 사용하게 되지만 데이터의 크기를 보았을 때 문제될 수치는 아니라고 판단했다.

<div class="white-space--dot"></div>

## 변경된 쿼리
```sql
SELECT 
    /*+ USE_HASH (INFO, SAP) */
    ...
FROM ITEM I             -- 약 30만건의 데이터
JOIN ITEM_INFO INFO     -- 약 30만건의 데이터
    ON I.PLANT = INFO.PLANT AND I.NO = INFO.NO
JOIN ITEM_INFO_SAP SAP  -- 약 30만건의 데이터
    ON I.PLANT = SAP.PLANT AND I.NO = SAP.NO
JOIN (
    SELECT
        MAX(VERSION) AS FINAL_VERSION,
        ...
    FROM ITEM_SPEC -- 약 40만건의 데이터
    GROUP BY
        ...
) SPEC
    ON I.PLANT = SPEC.PLANT AND I.NO = SPEC.NO
WHERE
    I.PLANT = #{plant}
AND I.USE = 'Y'
```
Hash Join을 타도록 힌트를 추가하고 다시 실행 계획을 조회해보았다.
> 1. PLANT와 USE로 필터링된 ITEM(I)과 GROUP BY의 결과로 생성된 인라인 뷰(SPEC)가 Hash Join
> 2. 1번의 결과셋과 ITEM_INFO_SAP(SAP)과 Hash Join
> 3. 2번의 결과셋과 ITEM_INFO(INFO)와 Hash Join


## 결과

|CPU Cost|IO Cost|실행 시간|
|---|---|---|
|11% 증가|6% 증가|35% 감소|


예상한 것처럼 CPU와 IO Cost가 증가하는 모습을 보였고 실행 속도는 크게 향상된 결과를 얻었다.


