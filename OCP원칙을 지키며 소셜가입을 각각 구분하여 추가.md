# OAuth 로그인 다형성과 팩토리 패턴을 이용한 추상화

프로젝트를 진행하면서 새롭게 배우고 적용한 부분에 대해서 다시 한번 보고 정리를 하고자 정리글을 작성하였습니다.
<br>
이 글은 카카오 또는 네이버와 같은 소셜 로그인을 구현할 때, <br>
어떻게 각각의 소셜 로그인별로 구분을 하였는지와 소셜 로그인이 추가될때(예: 페이스북 로그인/회원가입 추가) 기존 코드의 수정없이 유연하게 확장을 할 수있는지에대해 정리하였습니다.

<br>

## 1. OAuth 제공자 확장을 위한 추상화

### 문제 상황

구글, 카카오, 네이버 등 여러 소셜 로그인 기능을 제공해야 하는 상황입니다.
만약 로그인 요청을 처리하는 서비스 계층에서 분기문(if-else)을 사용하여 각 제공자(Provider)에 맞는 로직을 처리한다면, 소셜 로그인별로 구현을 해야하며 그때마다 수정을하고 코드가 길어지고 복잡해질 것 입니다.

<br>

```java
@Transactional
public OAuthResponse oauthLogin(Provider provider, String authorizationCode) {
    OAuthUserInfoDto userInfo = null;

    if (provider == Provider.GOOGLE) {
        // 구글로 액세스 토큰 요청
        // 구글 유저 정보 조회 및 userInfo 변환
    } else if (provider == Provider.KAKAO) {
        // 카카오로 액세스 토큰 요청
        // 카카오 유저 정보 조회 및 userInfo 변환
    } else if (provider == Provider.NAVER) {
        // 네이버로 액세스 토큰 요청
        // 네이버 유저 정보 조회 및 userInfo 변환
    } 
                    .
                    .
                    .

    // 이후 로그인/회원가입 처리...
    }       
```
또한, 추후에 페이스북, 인스타그램 등 새로운 OAuth 제공자를 추가하거나 기존 제공자를 삭제할 때마다 핵심 비즈니스 로직(서비스 계층)의 코드를 직접 수정해야 하므로 매우 번거로워지며 **OCP**(개방 폐쇄 원칙)을 위배하게 됩니다.

### 해결
**전략 패턴**과 **팩토리 패턴**을 도입하여 외부 API 통신 및 응답 처리를 추상화하였습니다.
> `OAuthClient`라는 공통 인터페이스를 만들어 각 소셜 제공자(Google, Kakao, Naver)마다 이를 기준으로 구현하도록 설계하였으며,
> `OAuthClientFactory` 필드에 `OAuthClient`인터페이스를 받아 요청에 맞는 해당 인터페이스 구현체를 런타임에 동적으로 주입하여 사용.

<br>

## 2. 패턴 구현 방법

### 1. 공통 인터페이스 정의 (`OAuthClient.java`)
모든 OAuth 클라이언트가 공통으로 가져야 할 인터페이스로 정의합니다.
해당 인터페이스는 자신이 어떤 제공자(`Provider`)인지 식별하는 메서드와 인가 코드를 받아 유저 정보를 반환하는 메서드를 정의하며 구현체는 각자의 방식으로 이를 구현해야합니다.
>이를 구현하는 구현체들은 꼭 스프링 컨테이너에 등록을 해주어야 아래 설명에서와같이 DI기능을 활용할 수 있음.

```java
public interface OAuthClient {
    // 해당 클라이언트가 어떤 경로로 소셜 로그인(구글, 카카오, 네이버 등)을 하였는지
    Provider provider();

    // 공통된 응답 객체(OAuthUserInfoDto)로 유저 정보를 추상화하여 반환
    OAuthUserInfoDto getUserInfo(String authorizationCode);
}
```
<br>

### 2. 팩토리 클래스를 통한 동적 라우팅 (`OAuthClientFactory.java`)
Spring의 의존성 주입을 활용하여, `OAuthClient`를 구현한 모든 빈을 `List`로 주입받습니다.
이후 `Provider`를 키로, 구현체(Client)를 값(Value)으로 하는 `Map`을 구성하여 
요청이 들어왔을 때 분기문 없이 O(1)의 시간 복잡도로 바로바로 알맞은 클라이언트를 찾아 반환합니다.

```java
@Component
public class OAuthClientFactory {

    private final Map<Provider, OAuthClient> clients;

    // Spring이 OAuthClient를 구현한 모든 클래스(GoogleOAuthClient, KakaoOAuthClient 등)를 List로 주입
    public OAuthClientFactory(List<OAuthClient> clientList) {
        this.clients = clientList.stream()
                .collect(Collectors.toMap(
                        OAuthClient::provider, // Key: 구글, 카카오, 네이버 등
                        client -> client       // Value: 해당 구현체 객체
                ));
    }

    public OAuthClient getClient(Provider provider) {
        OAuthClient client = clients.get(provider);
        if (client == null) {
            throw new BusinessException(ErrorCode.UNSUPPORTED_OAUTH_PROVIDER, "지원하지 않는 OAuth Provider입니다: " + provider);
        }
        return client;
    }
}
```

<br>

### 3. 클라이언트 구현체 생성 (`GoogleOAuthClient.java` 예시)
실제 각 소셜 로그인의 API 규격에 맞춰 통신하고, 결과를 공통 객체인 `OAuthUserInfoDto`로 변환하여 반환하는 책임을 가집니다.

```java
@Component
public class GoogleOAuthClient implements OAuthClient {

    @Override
    public Provider provider() {
        return Provider.GOOGLE;
    }

    @Override
    public OAuthUserInfoDto getUserInfo(String authorizationCode) {
        // 1. 구글에 인가 코드를 보내 액세스 토큰 발급 로직
        String accessToken = getAccessToken(authorizationCode);
        
        // 2. 액세스 토큰으로 유저 정보 조회 및 공통 DTO 반환 로직
        return fetchUserInfo(accessToken);
    }
    
    // API 요청 등 세부 구현 생략...
}
```

<br>

### 4. 컨트롤러와 서비스 계층에서의 활용 
이렇게 추상화된 팩토리와 인터페이스 덕분에, 실제 API 요청을 받는 컨트롤러와 비즈니스 로직을 처리하는 서비스 계층에서는 **어떤 OAuth 제공자인지 신경 쓸 필요 없이 일관된 로직**을 유지할 수 있습니다.

**[Controller 계층]**
어떤 제공자(구글, 카카오, 네이버)로 로그인을 시도하든, `@PathVariable`을 통해 문자열로 제공자 이름을 받아 Enum으로 변환한 뒤 서비스로 넘겨주기만 하면 됩니다.
```java
@PostMapping("/oauth/{provider}")
@ResponseStatus(HttpStatus.OK)
public OAuthResponse oauthLogin(
        @PathVariable("provider") String providerName, 
        @RequestBody @Valid OAuthRequest request
) {
    Provider provider = Provider.from(providerName); // 문자열을 Enum으로 변환
    return authService.oauthLogin(provider, request.getAuthorizationCode());
}
```

<br>

**[Service 계층]**
서비스 계층에서는 `OAuthClientFactory`에 제공자(`Provider`)를 넘겨주면, 내부적으로 알맞은 구현체(구글, 카카오 등)가 반환됩니다. 이후 공통 메서드인 `getUserInfo()`를 호출하면 끝입니다. 불필요한 `if-else` 분기문이 전혀 없습니다.
```java
@Transactional
public OAuthResponse oauthLogin(Provider provider, String authorizationCode) {
    // 팩토리 패턴을 이용해 알맞은 클라이언트를 가져오고 유저 정보를 받아옴 (다형성)
    OAuthUserInfoDto userInfo = oAuthClientFactory
            .getClient(provider)
            .getUserInfo(authorizationCode);

    // 이후 로그인 또는 회원가입 처리 로직 (제공자에 무관하게 동일하게 동작)
    Optional<Member> existingMember = memberRepository
            .findByProviderAndProviderId(provider, userInfo.getProviderId());

    boolean isNewUser = existingMember.isEmpty();

    Member member = existingMember.orElseGet(() -> {
        if (userInfo.getEmail() != null && memberRepository.existsByEmail(userInfo.getEmail())) {
            throw new BusinessException(ErrorCode.ALREADY_REGISTERED_EMAIL);
        }

        return memberRepository.save(Member.builder()
                .email(userInfo.getEmail())
                .nickname(userInfo.getNickname())
                .provider(provider)
                .providerId(userInfo.getProviderId())
                .build());
    });

    String accessToken = jwtTokenProvider.generateAccessToken(member.getId(), member.getRole());
    String refreshToken = issueRefreshToken(member.getId());

    return new OAuthResponse(accessToken, refreshToken, isNewUser);
}
```

<br>

## 3. 이 패턴의 장점 (새로운 OAuth 경로를 추가해야하는 상황)

만약 서비스가 확장되어 **페이스북** 로그인을 도입해야 하는 상황일때, 
이러한 구조에서는 기존 컨트롤러나 서비스 계층의 로직을 수정할 필요가 없습니다 
>어떤 경로로, 어떤 제공자의 요청이 들어와도 추상화 패턴 덕분에 동일하게 처리되기 때문

아래와 예시와 같이 새로운 구현체를 만들어주기만 하면 됩니다.

```java
@Component
public class newOAuthClient implements OAuthClient {
    
    @Override
    public Provider provider() {
        return Provider.새로운경로; // 새로운 제공자 타입 명시
    }

    @Override
    public OAuthUserInfoDto getUserInfo(String authorizationCode) {
        // new API 연동 및 유저 정보 조회 로직 구현
        return new OAuthUserInfoDto(...); 
    }
}
```

Spring이 애플리케이션 컨텍스트를 띄울 때 `@Component` 어노테이션이 붙은 `newOAuthClient`를 자동으로 빈으로 등록하고, 이후 `OAuthClientFactory`의 `List<OAuthClient>`에 주입됩니다.

결과적으로 다형성과 팩토리 패턴을 활용하여, 어떤 경로로 들어와도 유연하게 처리할 수 있게되었으며, 새로운 기능이 추가될 때 기존 코드는 전혀 변경하지 않고 확장만 가능한 OCP 원칙을 지킬 수 있었습니다.
