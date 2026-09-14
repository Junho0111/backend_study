# API 연동 및 데이터 매핑 

프로젝트를 진행하면서 새롭게 배우고 적용한 부분에대해서 다시 한번 보고 정리를 하고자 정리글을 작성하였습니다
<br>
이 정리글은 흔히 사용하는 카카오, 네이버, 구글 API 뿐 아니라 공공데이터포털 API 연동 법 나아가 응답을 어떻게 객체로 변환하는지까지 정리하였습니다.

<br>

## 1. 외부 API 연결 방법

### 문제 상황

기본적인 로그인/회원가입을 하기위해 구글이나 카카오 등에 API 요청을 보내어 토큰이나 유저의 정보를 가져와야하는 상황입니다.
기존에는 오래된 동기식 API인 `RestTemplate`를 사용을 하였으며, `WebClient`라는 비동기 처리를 하는 의존성을 추가하여야하는 번거로움과 부담이 있었습니다.

### 해결
Spring Boot 3.2부터 도입된 최신 동기식 HTTP 클라이언트인 **`RestClient`**를 사용하였습니다. 
> `RestClient`는 Fluent API(메서드 체이닝)를 제공하여 코드가 간결하고 직관적이며, 외부 라이브러리 의존성 없이 Spring Web 모듈만으로 사용할 수 있음.

<br>

### RestClient 사용 방법
1. `RestClient` 객체를 생성자를 통해 주입받습니다.
2. `restClient.post()` 또는 `restClient.get()`을 사용하여 HTTP 메서드를 지정합니다.
3. `.uri()`로 요청할 엔드포인트를 지정합니다.
4. `.contentType()`, `.header()` 등으로 HTTP 헤더를 설정합니다.
5. POST 요청인 경우 `.body()`를 통해 요청 본문(Body) 데이터를 삽입합니다. (예: `MultiValueMap`을 활용한 폼 데이터)
6. `.retrieve()`를 호출하여 실제 요청을 실행합니다.
7. `.body(클래스타입.class)`를 호출하여 응답을 원하는 Java 객체(Record 등)로 즉시 반환받습니다.

### [예시 코드1] (`GoogleOAuthClient.java`)
```java
// 구글 액세스 토큰 요청 예시
private String getAccessToken(String authorizationCode) {
    // 1. 요청 Body 데이터 구성 (Form 데이터 전송을 위해 MultiValueMap 사용)
    MultiValueMap<String, String> body = new LinkedMultiValueMap<>();
    body.add("grant_type", "authorization_code");
    body.add("client_id", clientId);
    body.add("client_secret", clientSecret);
    body.add("redirect_uri", redirectUri);
    body.add("code", authorizationCode);

    // 2. RestClient를 사용한 외부 API 통신
    GoogleTokenResponse response = restClient.post()
            .uri("https://oauth2.googleapis.com/token")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED) // 헤더 설정
            .body(body) // 바디 데이터 삽입
            .retrieve() // 요청 실행
            .body(GoogleTokenResponse.class); // Jackson을 통해 즉시 객체로 매핑

    if (response == null || response.accessToken() == null) {
        throw new BusinessException(ErrorCode.INVALID_INPUT_VALUE, "구글 액세스 토큰을 가져올 수 없습니다.");
    }
    return response.accessToken();

    private record GoogleTokenResponse(@JsonProperty("access_token") String accessToken) {}
}
```

<br>

### [예시 코드2] (`GoogleOAuthClient.java`)
```java
// 구글 유저정보 요청 예시
private OAuthUserInfoDto fetchUserInfo(String accessToken) {
    // RestClient를 사용한 외부 API 통신
        GoogleUserResponse googleUserResponse = restClient.get()
                .uri("https://www.googleapis.com/oauth2/v2/userinfo")
                .header("Authorization", "Bearer " + accessToken) // 헤더 설정
                .retrieve() // 요청 실행
                .body(GoogleUserResponse.class); // Jackson을 통해 즉시 객체로 매핑

        if (googleUserResponse == null) {
            throw new BusinessException(ErrorCode.INVALID_INPUT_VALUE, "구글 유저 정보를 가져올 수 없습니다.");
        }

        String providerId = googleUserResponse.id();
        String email = googleUserResponse.email();
        String nickname = googleUserResponse.name();

        return new OAuthUserInfoDto(providerId, email, nickname);
    }

    private record GoogleUserResponse(String id, String email, String name) {}
```

<br>


## 2. API 응답(JSON)을 Java 객체로 받는 법

### 문제 상황
공공데이터 API(Tour API)나 카카오톡 OAuth 등의 응답 결과는 다음과 같이 깊고 복잡한 계층형 JSON 형태로 전달됩니다.
>각각의 API의 응답 형식이 다르니, 응답 스펙을 확인하여야함.
```json
{
  "response": {
    "body": {
      "items": {
        "item": [
          {"contentid": "123", "title": "장소이름" ...}
        ]
      },
      "totalCount": 10
    }
  }
}
```
이러한 JSON을 `Map<String, Object>` 형태로 파싱해서 사용하거나 `JsonNode`를 사용하게되면 가독성이 떨어지며, 타입 안정성이 보장되지 않아 런타임 오류가 발생할 수 있습니다.

### 해결 
Jackson 라이브러리의 **`@JsonProperty`** 어노테이션과 Java 14에 도입된 불변 객체 **`Record`**를 적극 활용하여 DTO 클래스를 구성합니다.
> 굳이 `Record`를 사용하지 않아도 되며, 해당 프로젝트의 경우 `Record`의 불변성과 응답 객체가 쓰일 특징이 맞아 떨어져 사용하게됨.

<br>

`@JsonProperty`는 JSON의 키 이름과 자바 필드명이 다를 때(예: snake_case vs camelCase) 손쉽게 매핑할 수 있게 해주며, 중첩 클래스를 통해 복잡한 JSON 계층을 깔끔하고 직관적으로 매핑 할 수 있습니다.

<br>

### 복잡한 API 응답(JSON)을 Java객체로 변환
1. 최상위 JSON 응답을 감쌀 바깥 DTO 클래스를 생성합니다.
2. JSON의 내부 중첩 객체(Object)에 대응하는 내부 `record` 클래스들을 계층적으로 선언합니다.
3. JSON의 키(key) 이름과 자바 변수명이 다른 경우, 필드 앞에 `@JsonProperty("실제_JSON_키")`를 붙여줍니다.
4. JSON 배열 데이터는 `List<T>` 타입으로 받습니다.
5. 불필요한 계층은 캡슐화(`private`)하고, 비즈니스 로직에 실질적으로 필요한 데이터(`List<PoiItem>` 등)만 반환하는 `getter` 메서드를 상위 클래스에 정의하여 외부 사용성을 높입니다.

### 예시 코드

**[예시 1] 공공데이터 API의 복잡한 중첩 JSON 매핑 (`AreaBasedListResponse.java`)**
```java
public class AreaBasedListResponse {

    @JsonProperty("response")
    private Response response;

    // 외부에 필요한 핵심 데이터만 꺼내서 제공 (캡슐화 및 Null 안정성 처리)
    public List<PoiItem> getItems() {
        if (response == null || response.body == null || response.body.items == null) {
            return List.of();
        }
        return response.body.items.item;
    }

    // JSON의 계층 구조를 그대로 표현한 내부 record 클래스들 (private 선언으로 은닉)
    private record Response(@JsonProperty("body") Body body) {}

    // JSON의 계층 구조를 그대로 표현한 내부 record 클래스들 (private 선언으로 은닉)
    private record Body(
            @JsonProperty("items") Items items,
            @JsonProperty("totalCount") int totalCount
    ) {}

    private record Items(@JsonProperty("item") List<PoiItem> item) {
        // 빈 문자열 대응 등을 위한 커스텀 JsonCreator 추가 가능
        @JsonCreator
        public static Items fromString(String val) {
            return new Items(List.of());
        }


        //해당 API의 경우 없는 값을 null이나 []이 아닌 ""로 주며 이때 Jackson은 객체자리에 문자열이 오는것에 대해 MismatchedInputException 예외를 터뜨림. 
        //때문에 값이 ""이면 []로 변환해주는 커스텀기능을 추가한것.
    }


    // 실제 우리가 사용하고자 하는 핵심 데이터 모델 객체. 잘보면 실제 비즈니스 로직에서 호출되어야 하므로 public임
    public record PoiItem(
            @JsonProperty("contentid") String contentId,
            @JsonProperty("contenttypeid") String contentTypeId,
            @JsonProperty("title") String title,
            @JsonProperty("addr1") String addr1,
            @JsonProperty("mapy") String lat,
            @JsonProperty("mapx") String lng,
            @JsonProperty("firstimage") String firstImage,
            @JsonProperty("firstimage2") String firstImage2,
            @JsonProperty("sigungucode") String sigunguCode
    ) {}
}
```

<br>

**[예시 2] OAuth 응답의 필드명 불일치 매핑 (`KakaoOAuthClient.java` 내부 DTO)**
```java
// 카카오 API 응답 JSON 계층 구조에 맞춘 record 선언
private record KakaoUserResponse(
        Long id, 
        // JSON에서는 snake_case로 넘어오지만, 자바에서는 camelCase로 받기 위해 매핑
        @JsonProperty("kakao_account") KakaoAccount kakaoAccount
) {
    public record KakaoAccount(String email, Profile profile) {
        public record Profile(String nickname) {}
    }
}
```

