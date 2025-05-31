---
{"dg-publish":true,"permalink":"/프로젝트/TripMate/관광지 페이지네이션 성능 최적화(캐시 미적용 vs Ehcache vs Redis)/","created":"2025-05-30T17:19:51.431+09:00"}
---

# Contents
- ## 문제 상황
- ## 해결 방법: Spring Cache 적용
- ## nGrinder 테스트 환경 및 스크립트 구성
- ## Ehcache 성능 분석
- ## 캐시 성능 분석
- ## 결론
---
# 문제 상황

## 관광지 목록 API는 다음과 같은 기능을 제공한다:
- **지역, 시군구, 관광지 타입 조건에 따른 필터링**
- **총 5만 건 이상의 관광지 데이터를 페이지 단위로 제공**

## 문제 상황: 반복되는 전체 개수 조회로 인한 DB 부하
페이지네이션 구현을 위해 클라이언트에 **총 페이지 수**를 알려줘야 하므로,  
`SELECT COUNT(*)` 쿼리를 통해 전체 개수를 조회해야 한다. 하지만 이 쿼리는 사용자마다 반복 호출되는 문제점이 있다:
- **DB에 과도한 부하**: 동일한 조건이라도 매번 COUNT 쿼리 실행
- **불필요한 중복 쿼리**: 캐싱이 없다면 매번 동일 쿼리 수행

---
# 해결 방법: Spring Cache 적용

## 캐시 적용 이유
- 관광지 데이터는 **변동이 적고 정적**
- 조건 조합(areaCode + sigunguCode + contentTypeId)을 **캐시 키**로 설정
- 중복 호출 시 DB 접근 없이 캐시로 응답
- 등록/수정/삭제 시 **@CacheEvict**를 통해 정합성 유지

##  구현 코드

### `build.gradle`
```gradle
implementation 'org.ehcache:ehcache:3.10.8'
implementation 'org.springframework.boot:spring-boot-starter-cache'
```

### `CacheConfig.java`
```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public JCacheManagerCustomizer cacheManagerCustomizer() {
        return cm -> {
            if (!cm.getCacheNames().contains("attractionCount")) {
                cm.createCache("attractionCount", new MutableConfiguration<>()
                    .setExpiryPolicyFactory(CreatedExpiryPolicy.factoryOf(new Duration(TimeUnit.MINUTES, 10)))
                    .setStoreByValue(false)
                    .setStatisticsEnabled(true));
            }
        };
    }
}
```

### `AttractionCountServiceImpl.java`
```java
@Service
public class AttractionCountServiceImpl implements IAttractionCountService {
    @Autowired
    private AttractionDAO dao;

    @Override
    @Cacheable(
        value = "attractionCount",
        key = "T(String).format('%s_%s_%s', #requestDto.areaCode, #requestDto.sigunguCode, #requestDto.contentTypeId)"
    )
    public long countAttractionsByAreaAndSigunguAndType(final AttractionRequestDto requestDto) {
        return dao.countAttractionsByAreaAndSigunguAndType(
            requestDto.getAreaCode(),
            requestDto.getSigunguCode(),
            requestDto.getContentTypeId()
        );
    }
}
```

### `AttractionServiceImpl.java`
```java
@Slf4j
@Service
@RequiredArgsConstructor
public class AttractionServiceImpl implements IAttractionService {

    private final AttractionDAO dao;
    private final AttractionCountServiceImpl attractionCountService;

    @Override
    public AttractionPageResponseDto fetchAttractionsByAreaAndSigunguAndTypeWithPaging(
        final AttractionRequestDto requestDto,
        final Pageable pageable
    ) {
        long totalItems = attractionCountService.countAttractionsByAreaAndSigunguAndType(requestDto);

        return buildPagedResponse(
            () -> dao.fetchAttractionsByAreaAndSigunguAndTypeWithPaging(
                requestDto.getAreaCode(), requestDto.getSigunguCode(), requestDto.getContentTypeId(), pageable),
            totalItems,
            pageable
        );
    }

    @Override
    @CacheEvict(value = "attractionCount", allEntries = true)
    public int addAttraction(AttractionCreateDto dto) {
        // 관광지 등록 로직
    }

    @Override
    @CacheEvict(value = "attractionCount", allEntries = true)
    public int deleteAttraction(int no) {
        // 관광지 삭제 로직
    }
}
```

---

# nGrinder 테스트 환경 및 스크립트 구성

## 테스트 목적
- 캐시 적용 전후 `count()` 쿼리 성능 변화 측정
- TPS, 평균 응답 시간, 에러율 비교

## 테스트 환경
| 항목            | 값                                                    |
| ------------- | ---------------------------------------------------- |
| 테스트 도구        | nGrinder 3.5.9, Scouter                              |
| 테스트 API       | `/api/attractions/search-with-paging`                |
| 검색 조건         | `{ areaCode: 1, sigunguCode: 2, contentTypeId: 12 }` |
| 요청 방식         | POST (JSON Body)                                     |
| 사용자 수 (vUser) | 10 / 99 / 198                                        |
| 테스트 시간        | 각 10분                                                |


## nGrinder 테스트 스크립트 요약
```groovy
@Test
public void test() {
    String url = "http://localhost:8080/api/attractions/search-with-paging?page=0&size=10";

    def requestBody = new JsonBuilder([
        areaCode      : 1,
        sigunguCode   : 2,
        contentTypeId : 12
    ]).toString();

    long startTime = System.currentTimeMillis();
    HTTPResponse response = request.POST(url, requestBody.getBytes("UTF-8"));
    long responseTime = System.currentTimeMillis() - startTime;

    if (responseTime > 600) {
        fail("응답 시간이 600ms를 초과했습니다: " + responseTime + "ms");
    } else {
        assertThat(response.statusCode, is(200));
    }
}
```
### ==600ms 초과 시 실패 처리==하여 사용자 경험 기준 반영함

## 성능 지표
Ehcache 기반 캐시를 적용한 후, nGrinder + Scouter를 통해 실제 성능을 측정했으며, 성능 지표는 다음과 같다:
- **TPS (초당 처리 건수)**
- **처리량 (총 요청 수)**
- **응답시간 (평균 Elapsed Time)**
- **Heap Used (메모리 사용량)**
- **XLog (요청별 응답 시간 분포)**

---
# Ehcache 성능 분석
## 결과

| 구분   | vUser | TPS    | 평균 응답시간(ms) | 총 요청 수  | 에러 수  |
| ---- | ----- | ------ | ----------- | ------- | ----- |
| 캐시 X | 10    | 81.3   | 122.83      | 48,449  | 0     |
| 캐시 O | 10    | 1258.4 | 7.82        | 749,217 | 0     |
| 캐시 O | 99    | 965.2  | 102.34      | 573,617 | 10    |
| 캐시 O | 198   | 918.9  | 211.17      | 545,374 | 3,166 |
> 캐시 미적용 상태에서 vUser 99 이상으로 부하를 주면 테스트 도중 서버 과부하로 중단됨
###  Spring Cache(Ehcache)를 활용해 `count()` 쿼리를 캐싱하는 것만으로도  ==응답시간을 94% 단축하고, TPS를 15배 향상==시킬 수 있다.
---
## vUser 10 기준 비교

### 캐시 적용 X  
![캐시 미적용 TPS](https://i.imgur.com/PGrtcei.png)
 
### 캐시 적용 O  
![캐시 적용 TPS](https://i.imgur.com/RLg9pEj.png)

- **TPS**: 약 1258.4 → **15배 증가**
- **처리량**: 749,217 → **약 15.5배 증가**
- **응답 시간**: 7.82ms → **94% 감소**

---

## vUser 99 vs vUser 198 비교

### vUser 99 (캐시 적용)
![TPS 99](https://i.imgur.com/Lz2LjJn.png)  
![Scouter 99](https://i.imgur.com/ftzY0C5.png)

- **TPS**: 약 965 → 안정적
- **응답 시간**: 평균 102.34ms → 실사용 범위
- **Heap Used**: 100~300MB 구간에서 안정적 (평균 약 250MB)
- **XLog**: 대부분 100~200ms 내에 응답

#### **해석**:  
- TPS와 응답 시간이 균형 있게 유지되고 있음  
- XLog 분포도 정상적이며 시스템 안정성 확보 가능  
- **Heap 사용량이 일정 범위에서 안정적으로 유지**되어 GC 지연이 발생하지 않음

---

### vUser 198 (캐시 적용)
![TPS 198](https://i.imgur.com/sbAupSO.png)  
![Scouter 198](https://i.imgur.com/7bNugJO.png)

- **TPS**: 약 918.9 → 큰 하락 없이 유지
- **응답 시간**: 평균 211.17ms → 두 배 증가
- **에러 수**: 3,166건 발생 (약 0.6%)
- **Heap Used**: 100~400MB 구간, 평균 약 300MB로 증가
- **XLog**: 일부 요청이 1초 이상 소요

#### **해석**:  
- CPU 및 메모리 리소스가 임계치에 가까워지며 GC 또는 스레드 병목 가능성이 커짐  
- **Heap 사용량이 평균 300MB 수준으로 증가**, 처리 요청이 많아질수록 객체 생성과 캐시 데이터가 메모리에 축적됨  
- GC가 비동기적으로 수행되면서 순간적인 응답 지연이 발생 → 일부 요청이 1초 이상 소요됨  
- TPS는 유지되나, 응답시간 증가 + 에러 발생은 시스템 한계에 도달했다는 신호

---

# Redis 성능 분석

Spring Cache는 다양한 구현체(Ehcache, Redis 등)를 추상화하여 사용할 수 있는 유연한 구조이다. 이번 실험에서는 기존 Ehcache 기반의 캐시를 **Redis 기반 분산 캐시**로 전환한 후 성능을 측정하고 비교 분석해본다.

## 결과

| 구분         | vUser | TPS    | 평균 응답시간(ms) | 총 요청 수 | 에러 수 |
|--------------|--------|--------|-------------------|------------|---------|
| Redis 적용    | 10     | 900.0  | 10.91             | 536,559    | 0       |
| Redis 적용    | 99     | 901.2  | 109.68            | 535,454    | 4       |
| Redis 적용    | 198    | 904.4  | 214.35            | 538,509    | 3,273   |

> 💡 Ehcache에 비해 TPS는 소폭 낮지만, **분산 구조로 인한 확장성과 안정성에서 유리**

---

## vUser: 10 기준

![nGrinder TPS](https://i.imgur.com/A9ftCeq.png)  
![Scouter 상태](https://i.imgur.com/cBoFx8J.png)

- **TPS**: 900.0 → 안정적
- **응답 시간**: 평균 10.91ms
- **Heap Used**: 100~200MB로 낮게 유지
- **XLog**: 대부분 10~50ms 내 처리

#### **해석**:  
- Ehcache(1258.4 TPS)보다 약간 느리지만, Redis는 **외부 메모리 기반**이라 Heap 사용량이 현저히 낮음  
- 네트워크 오버헤드가 있지만 실 사용에는 큰 영향 없음

---

## vUser: 99 기준

![TPS 99](https://i.imgur.com/jSBiguS.png)  
![Scouter 99](https://i.imgur.com/SBUiisi.png)

- **TPS**: 901.2 → 꾸준한 처리 유지
- **응답 시간**: 109.68ms
- **Heap Used**: 평균 약 150MB
- **XLog**: 대부분 100~300ms 분포

#### **해석**:  
- Redis는 JVM Heap에 캐시를 저장하지 않기 때문에 Ehcache보다 메모리 사용량이 적음  
- TPS와 응답 시간도 **Ehcache와 유사 수준으로 안정적**

---

## vUser: 198 기준

![TPS 198](https://i.imgur.com/8G1MYCz.png)  
![Scouter 198](https://i.imgur.com/vtgcibt.png)

- **TPS**: 904.4 → 큰 하락 없이 유지
- **응답 시간**: 평균 214.35ms
- **에러 수**: 3,273건 (0.6%)
- **Heap Used**: 평균 250MB, 최대 300MB 근접
- **CPU 사용률**: 90% 이상 → 과부하 경고 다수
- **XLog**: 일부 요청은 1초 이상 지연

#### **해석**:  
- Redis도 vUser 198 수준에서는 CPU 및 GC의 영향을 받으며 처리 지연 발생  
- 다만, TPS는 일정하게 유지되어 **부하 분산 구조의 안정성**을 확인할 수 있음

---

## 캐시 전략별 성능 종합 비교

| 항목             | 캐시 미적용             | Ehcache                  | Redis                          |
| ---------------- | ----------------------- | ------------------------ | ------------------------------ |
| TPS (10 vUser)   | 81.3                    | **1258.4**               | 900.0                          |
| TPS (198 vUser)  | 테스트 중단             | 918.9                    | **904.4**                      |
| 평균 응답시간    | 122.83ms                | **7.82ms**               | 10.91ms                        |
| 처리량           | 48,449                  | 749,217                  | 536,559                        |
| 에러 수 (198)    | 테스트 실패             | 3,166                    | 3,273                          |
| Heap 사용량      | 비교적 낮음 (단순 쿼리) | 최대 400MB (평균 300MB)  | **최대 300MB (평균 250MB)**    |
| 캐시 저장 위치   | 없음                    | JVM 내부 메모리          | **외부 Redis 서버 (네트워크)** |
| 확장성           | 매우 낮음               | 단일 서버 수준           | **다중 서버/클러스터 가능**    |
| 장애 대응        | 해당 없음               | 서버 재시작 시 캐시 소멸 | **Replication, HA 구성 가능**  |
| 인스턴스 간 공유 | 불가능                  | 불가능                   | **가능**                       |

---

# 결론

- **캐시 미적용**: 단순한 구조이지만, 고트래픽에서 부하를 견디지 못하고 성능 저하

- **Ehcache**: 빠른 성능과 간편한 설정 → **개발 초기/소규모 서비스에 최적**
    
- **Redis**: TPS는 약간 낮지만 메모리 효율, 확장성, 다중 인스턴스 대응 능력 탁월 → **운영 환경/고부하 서비스에 적합**

> 캐시 성능을 최대한 끌어올리기 위해서는 애플리케이션 수준뿐 아니라 인프라 자원(스레드, 커넥션 등)의 튜닝도 함께 고려되어야 한다.