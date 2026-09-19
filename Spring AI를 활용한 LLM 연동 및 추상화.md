# Spring AI를 활용한 LLM 연동 및 추상화

프로젝트를 진행하면서 새롭게 배우고 적용한 부분에 대해서 다시 한번 보고 정리를 하고자 정리글을 작성하였습니다.
<br>
이 글은 Spring AI를 도입하여 LLM을 프로젝트에 어떻게 연동하였는지와, <br>
AI 모델이 추가되거나 변경될 때 기존 코드의 수정 없이 유연하게 확장을 할 수 있도록 어떻게 설계하였는지에 대해 정리하였습니다.

<br>

## 1. LLM 확장을 위한 추상화

### 문제 상황

현재 프로젝트에 AI 여행 일정 생성 기능을 위해 Gemini 모델을 연동해야 하는 상황입니다.
만약 외부 API를 서비스 계층에서 직접 호출하여 연동한다면, 요청/응답 형식이 해당 모델(Gemini)에 강하게 종속됩니다.

그리고 이는 나중에 어떠한 이유로 모델을 변경할때 핵심 비즈니스 로직(서비스 계층) 내부의 API 통신 코드와 DTO를 모두 뜯어고쳐야 하므로 매우 번거로워지며 **OCP**(개방 폐쇄 원칙)를 위배하게 됩니다. 과거 소셜 로그인을 연동할 때 제공자마다 규격이 달라 추상화를 하여 해결한것과 같은 상황이라고 볼 수 있습니다.

### 해결
**Spring AI**의 `ChatClient`를 도입하여 외부 API 통신을 추상화하였습니다.
> 과거 `OAuthClient`라는 공통 인터페이스를 만들어 다형성을 활용했던 것과 동일한 원리입니다. Spring AI는 이미 `ChatClient`라는 추상화 인터페이스를 제공하므로, 서비스 로직에서는 구체적인 구현체(Gemini인지 GPT인지)를 모른 채 `ChatClient` 인터페이스만 주입받아 사용하도록 설계하였습니다.

<br>

## 2. 패턴 구현 방법 및 연동

### 1. ChatClient 빈 등록 (`AiConfig.java`)
가장 먼저 해야 할 일은 Spring AI가 제공하는 `ChatClient`를 사용할 수 있도록 설정(Config) 클래스를 만들어 빈(Bean)으로 등록하는 것입니다.

```java
@Configuration
public class AiConfig {

    // ChatClient를 사용하기 위해 Builder를 주입받아 빈으로 등록
    @Bean
    public ChatClient aiChatClient(ChatClient.Builder builder) {
        return builder.build();
    }
}
```

<br>

### 왜 이렇게 따로 Config 클래스를 만들어 빈으로 등록해야 할까요?

단순히 서비스 계층에서 `new` 키워드로 객체를 생성하지 않고 빈으로 등록하는 이유는 스프링의 의존성 주입을 활용하기 위함입니다(원치 않으면 서비스 계층에서 위의 코드를 정의해서 사용하면 됩니다.)

이렇게 설정해 두면, `build.gradle`에 추가한 의존성(`spring-ai-starter-model-google-genai`)과 `application.yml`의 설정값을 스프링 부트가 자동으로 읽어들입니다.
>만약 claude 또는 gpt로 교체를 해야한다면 의존성 및 yaml의 설정만 바꿔주면 되며, 자동으로 의존성과 설정을 읽고 그에 맞는 llm모델 구현체를 동적으로 주입해줌.

그리고 런타임에 구글 Gemini와 통신할 수 있는 구현체를 만들어 `ChatClient` 인터페이스에 동적으로 주입해 줍니다. 
결과적으로 개발자는 `AiConfig`에서 공통된 인터페이스만 등록해두면, 서비스 로직 어디서든 일관된 방식 즉 다형성을 활용하며, LLM을 호출할 수 있게 됩니다.

<br>

### 2. 응답 데이터 매핑 (`BeanOutputConverter`)
두 번째는 LLM이 응답한 데이터를 자바 객체로 바꾸는 과정입니다.
기존에 카카오나 공공데이터 API 등을 연동할 때는 API가 정해진 스펙대로 JSON 구조를 반환했기 때문에 `@JsonProperty`와 `Record`를 활용해 쉽게 Java 객체로 매핑할 수 있었습니다.

하지만 LLM의 응답은 기본적으로 단순한 **문자열**입니다. 여기서 특정 데이터(예: 추천 장소 ID, 시간 등)만 정규식으로 뽑아내는 것은 런타임 오류의 원인이 됩니다.

그리고 이러한 문제를 해결하기 위해 **`BeanOutputConverter`**를 사용합니다.

```java
// 1. 응답을 받을 Java Record 객체를 타입으로 지정하여 Converter 생성
BeanOutputConverter<AiScheduleResult> converter = new BeanOutputConverter<>(AiScheduleResult.class);

// 2. 프롬프트에 Converter가 자동 생성한 JSON 형식(Format) 가이드를 주입
// promptAndCallLlm 메서드 내의 코드
String prompt = """
    당신의 임무는 제공된 모든 장소들을 최적의 일일 일정으로 배치하는 것입니다.
    ...
    %s <- 이 자리에 시스템이 요구하는 JSON 스키마 가이드가 들어갑니다.
    """;
String finalPrompt = String.format(prompt, ..., converter.getFormat());

return aiChatClient.prompt()
                .user(finalPrompt)
                .call()
                .content();
```

<br>

### 이 Converter는 구체적으로 어떤 역할을 할까요?
`converter.getFormat()`을 호출하면, 우리가 지정한 `AiScheduleResult` 클래스의 필드 구조를 분석해서 LLM에게 "이러한 형태의 JSON 스키마로 대답해 줘"라는 프롬프트 지시어를 자동으로 생성해 줍니다.

따라서 LLM은 우리가 원하는 완벽한 JSON 포맷으로 대답하게 되며, 응답을 받은 즉시 `converter.convert(response)` 메서드를 통해 LLM의 응답 문자열을 Java 객체로 안전하게 매핑 할 수 있습니다. 

<br>

### 3. 서비스 계층에서의 활용 및 비즈니스 로직 분리 (`AiItineraryService.java`)
이제 앞서 만든 `ChatClient`와 `BeanOutputConverter`를 실제 서비스 로직에 조합합니다.
이때 주의할 점은, 메인 비즈니스 코드의 흐름과 외부 API 통신(LLM 연동) 코드가 한 메서드에 섞이지 않도록 역할은 분담하여 나누도록 하는 것입니다.

```java
@Transactional
public CreateAiItineraryResponse createAiItinerary(Long memberId, CreateAiItineraryRequest request) {

    // 1. 데이터 조회 (핵심 비즈니스 로직)
    List<Poi> pois = poiRepository.findAllById(request.getPoiIds());
    if (pois.isEmpty()) { ... }

    // 2. LLM 컨텍스트 준비 (데이터 가공 분리)
    String poiJsonData = preparePoiContext(pois);

    // 3. LLM API 통신 (외부 통신 로직 분리)
    BeanOutputConverter<AiScheduleResult> converter = new BeanOutputConverter<>(AiScheduleResult.class);
    String response = promptAndCallLlm(totalDays, request, poiJsonData, converter);

    // 4. LLM 응답 검증 및 파싱 (데이터 검증 로직 분리)
    AiScheduleResult result = parseAndValidateResponse(response, converter, pois);

    // 5. 파싱된 결과를 바탕으로 DB 저장 및 후처리 (핵심 비즈니스 로직)
    Itinerary itinerary = Itinerary.builder()...
    itineraryRepository.save(itinerary);
}

// 캡슐화된 실제 LLM 호출 로직
private String promptAndCallLlm(long totalDays, CreateAiItineraryRequest request, String poiJsonData, BeanOutputConverter<AiScheduleResult> converter) {
    String finalPrompt = ... ; // 프롬프트 세팅
    
    // Config에서 빈으로 주입받은 ChatClient를 사용하여 통신 (다형성)
    return aiChatClient.prompt()
            .user(finalPrompt)
            .call()
            .content();
}
```

<br>

### 왜 이렇게까지 메서드를 분리해야 할까요?

`createAiItinerary`라는 메인 메서드는 '여행 일정을 생성한다'는 핵심 도메인 흐름을 담당합니다.
만약 이 안에 프롬프트 문자열을 조합하고, LLM을 호출하고, JSON을 파싱하고 검증하는 외부 종속적인 코드들이 모두 섞여 있다면, 코드를 읽는 사람은 비즈니스 로직의 큰 틀을 파악하기 매우 힘들어집니다. (비즈니스 코드와 llm을 호출하는 코드가 섞여 비즈니스 로직아 오염 됨)

따라서 외부 통신 및 파싱(매핑)에 대한 책임을 프라이빗 메서드로 분리하면, 메인 메서드가 마치 시나리오 흐름을 읽는거와같이 직관적으로 읽히게 됩니다.
또한 추후 프롬프트를 수정하거나 통신 방식을 변경할 때는 `promptAndCallLlm` 메서드만 수정하면 되므로 유지보수를 하는데있어 불편함이 없어집니다.(역할 분담의 중요성)

<br>

## 3. 이 패턴의 장점 (새로운 AI 모델을 적용해야하는 상황)

만약 서비스가 더욱 확장되어 현재의 Gemini 대신 OpenAI(GPT)를 도입해야 하는 상황이 온다고 가정해 보겠습니다.
현재와 같은 구조에서는 기존 서비스 계층(`AiItineraryService`)의 로직을 **단 한 줄도 수정할 필요가 없습니다.** 

> 어떤 모델의 연동 요청이 들어와도 `ChatClient`라는 인터페이스를 통해 추상화되어 있으므로, 스프링 컨테이너가 런타임에 알아서 교체된 객체를 주입하고 실행해주기 때문입니다.

개발자가 해야 할 일은 단지 `build.gradle`에서 의존성을 `spring-ai-starter-openai`로 변경하고, `application.yml`에 있는 구글 API 키 대신 OpenAI API 키 설정 값만 교체해 주는 것뿐입니다.

결과적으로 다형성과 추상화(인터페이스, 의존성 주입)를 활용하여, 어떤 외부 AI 모델로 변경되어도 유연하게 대처할 수 있게 되었습니다. 이는 새로운 모델이나 기능이 추가될 때 기존 코드는 전혀 변경하지 않고 설정만으로 확장이 가능하다는, OCP원칙을 지킨것이라고 볼 수 있습니다.

<br>

## +추가: 코드 예시의 프라이빗 메서드 및 설정 보충 설명

본문의 예시 코드(`AiItineraryService`)를 보면 Spring AI 연동 외에도 `preparePoiContext`, `parseAndValidateResponse` 같은 메서드들이 등장합니다. 이 글의 주제인 Spring AI 연동 자체에는 직접적인 연관이 없지만, 코드를 보면서 이 메서드들이 구체적으로 어떤 비즈니스 로직을 수행하는지 궁금해하실 분들을 위해 추가로 내용을 정리했습니다.
비즈니스 로직이 별로 궁금하지 않으시면 읽지 않으셔도 됩니다.

<br>

### 1. 데이터 가공: `preparePoiContext` 메서드의 역할
DB에서 엔티티 객체(`Poi`)를 통째로 LLM에 던지면, 불필요한 데이터(DB 내부 식별자, 생성일자, 작성자 등)까지 넘어가게 되어 토큰(비용)을 낭비하게 됩니다. 또한 프롬프트가 지저분해져서 LLM이 혼란을 겪을 수도 있으며, 이것이 만일 민감한 정보일시에는 문제가 될 수 있습니다.

이 메서드는 DB에서 가져온 데이터 중 LLM이 동선을 짜는 데 딱 필요한 핵심 정보(이름, 좌표, 영업시간)만 추출하여 하나의 JSON 문자열 컨텍스트로 직렬화(가공)하는 역할을 담당합니다.
<details>
<summary><b>preparePoiContext 보기 (클릭)</b></summary>

```java
private String preparePoiContext(List<Poi> pois) {
        List<Map<String, Object>> poiList = pois.stream()
                .map(poi -> Map.of(
                "poiId", poi.getId(),
                "name", poi.getName(),
                "latitude", poi.getCoordinate() != null ? poi.getCoordinate().getLat() : null,
                "longitude", poi.getCoordinate() != null ? poi.getCoordinate().getLng() : null,
                "openHours", poi.getOpenHours() != null ? poi.getOpenHours() : "Unknown",
                "category", poi.getCategory()
                ))
                .collect(Collectors.toList());
        try {
            return aiObjectMapper.writeValueAsString(poiList);
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize POI context", e);
            return "[]";
        }
    }
```
</details>

<br>

### 2. 데이터 검증: `parseAndValidateResponse` 메서드의 역할
LLM이 `BeanOutputConverter`를 통해 JSON 포맷을 반환했더라도, **데이터의 '내용'이 올바르다는 보장은 없습니다.** 
예를 들어, 우리가 "장소 A, B, C를 일정에 배치해 줘"라고 지시했는데, LLM이 임의로 "장소 D"를 만들어 반환(환각 현상 = Hallucination)할 수도 있습니다.

이 메서드는 단순히 응답을 객체로 파싱하는 것을 넘어, **LLM이 반환한 일정 속 장소(POI ID)들이 우리가 처음에 요청한 장소 리스트 안에 실제로 존재하는지 무결성을 검증**하는 매우 중요한 비즈니스 검증 로직입니다.
<details>
<summary><b>parseAndValidateResponse 보기 (클릭)</b></summary>

```java
private AiScheduleResult parseAndValidateResponse(String response, BeanOutputConverter<AiScheduleResult> converter, List<Poi> pois) {
        if (response == null) {
            throw new BusinessException(ErrorCode.INTERNAL_SERVER_ERROR, "LLM 응답이 비어있습니다.");
        }

        AiScheduleResult result = converter.convert(response);

        if (result == null || result.days() == null) {
            throw new BusinessException(ErrorCode.INTERNAL_SERVER_ERROR, "LLM 응답을 파싱할 수 없습니다.");
        }

        Set<Long> validPoiIds = pois.stream()
                .map(Poi::getId)
                .collect(Collectors.toSet());

        for (AiScheduleResult.AiScheduleDay day : result.days()) {
            for (AiScheduleResult.AiScheduleItem item : day.items()) {
                if (!validPoiIds.contains(item.poiId())) {
                    throw new BusinessException(ErrorCode.INVALID_INPUT_VALUE,
                            "LLM이 요청하지 않은 poiId를 반환했습니다: " + item.poiId());
                }
            }
        }
        
        return result;
    }
```
</details>
<br>

### 3. 환경 설정(Config)에서의 비즈니스 정책 분리 (`aiObjectMapper`)
이러한 분리 원칙은 서비스 클래스뿐만 아니라 설정(Config) 클래스에도 똑같이 적용됩니다. `AiConfig.java`를 보면 `ChatClient` 외에도 다음과 같은 설정이 추가로 존재합니다.

```java
@Bean
public ObjectMapper aiObjectMapper() {
    return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
}
```

#### 이 `aiObjectMapper`는 왜 필요할까요?
LLM은 확률 기반으로 텍스트를 생성하므로, 우리가 지시한 JSON 포맷 외에 가끔 환각(Hallucination)을 일으켜 우리가 요청하지 않은 엉뚱한 필드를 섞어서 응답할 때가 있습니다.

만약 기본 설정의 `ObjectMapper`를 그대로 사용한다면, Java 객체에 없는 필드가 들어왔을 때 즉시 에러(`UnrecognizedPropertyException`)를 터뜨리고 사용자에게 500 에러를 반환해버립니다.

하지만 우리 서비스의 **비즈니스 정책** 상, "LLM이 핵심 데이터(장소 ID, 시간 등)만 잘 줬다면, 쓸데없는 필드를 몇 개 덧붙였더라도 에러를 내지 말고 유연하게 넘어가서 일정을 만들어 주자"라고 결정할 수 있습니다.

이러한 유연한 방어 정책을 코드에 반영하기 위해 `FAIL_ON_UNKNOWN_PROPERTIES, false` (모르는 속성이 들어와도 에러를 내지 않고 무시함) 옵션을 켠 전용 `ObjectMapper`를 빈으로 등록한 것입니다. 

그리고 이것은 핵심 서비스 로직 안에서 매번 파싱 로직을 `try-catch`로 감싸며 코드를 더럽히는 대신, Config 클래스에서 전역적인 정책(Bean)으로 깔끔하게 분리해둔 것 입니다.


