# Spring Security와 JWT 기반 인증 인가 구현 3 - Token 발급과 Refresh Token

이전 글들에서는 Spring Security의 기본 개념과 SecurityConfig 설정, 인증 필터 구현 방법에 대해 다루었습니다.
<br>
실제 로그인 시 토큰(Access Token, Refresh Token)을 발급하는 로직과, 
<br>만료된 Access Token을 갱신하기 위한 **Refresh Token**의 관리 및 비즈니스 로직에 대해 정리하였습니다.

<br>

## 1. Access Token과 Refresh Token의 도입

### 문제 상황
JWT는 한 번 발급되면 서버에서 제어할 수 없습니다. 만약 탈취당한다면, 토큰이 만료될 때까지 누구나 해당 사용자로 접근할 수 있는 보안상 치명적인 단점이 있습니다.
그렇다고 Access Token의 유효 기간을 너무 짧게(예: 30분) 설정하면, 사용자는 30분마다 다시 로그인을 해야 하는 불편함을 겪게 됩니다.

### 해결
유효 기간이 짧은 **Access Token**과, 이 토큰을 재발급받기 위한 용도의 유효 기간이 긴 **Refresh Token**을 함께 사용하는 방식을 도입했습니다.
> 평소 통신에는 Access Token을 사용하다가 만료되면, 안전하게 보관해 둔 Refresh Token을 서버로 보내 새로운 Access Token을 발급받습니다. Refresh Token은 서버의 DB에 저장해두고 관리하므로 필요시 강제로 로그아웃 처리하여 탈취당한 토큰을 무효화할 수 있습니다.

<br>

## 2. 패턴 구현 방법 및 연동

### 1. JWT 토큰 생성 및 검증 (`JwtTokenProvider.java`)

```java
@Component
public class JwtTokenProvider {

    private final SecretKey secretKey;
    private final long accessTokenExpiryMs;
    private final long refreshTokenExpiryMs;

    public JwtTokenProvider(@Value("${jwt.secret}") String secret,
                            @Value("${jwt.access-token-expiry-ms}") long accessTokenExpiryMs,
                            @Value("${jwt.refresh-token-expiry-ms}") long refreshTokenExpiryMs) {
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessTokenExpiryMs = accessTokenExpiryMs;
        this.refreshTokenExpiryMs = refreshTokenExpiryMs;
    }

    // Access Token 발급
    public String generateAccessToken(Long memberId, Role role) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + accessTokenExpiryMs);

        return Jwts.builder()
                // --- [페이로드 (Payload) 영역] ---
                .subject(String.valueOf(memberId)) // 유저 식별자
                .claim("role", role.name())       // 권한 정보
                .issuedAt(now)
                .expiration(expiry)
                // --- [서명 및 비밀키 (Signature) 영역] ---
                .signWith(secretKey)
                // --- [헤더 (Header) 자동 생성 및 최종 결합] ---
                .compact();
                // 헤더 자동 생성 + 페이로드 + 서명을 점(.)으로 결합
    }

    // Refresh Token 발급 (페이로드 최소화)
    public String generateRefreshToken(Long memberId) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + refreshTokenExpiryMs);

        return Jwts.builder()
                .subject(String.valueOf(memberId))
                .issuedAt(now)
                .expiration(expiry)
                .signWith(secretKey)
                .compact();
    }

    // 사용자 식별자(memberId) 추출
    public Long extractMemberId(Claims claims) {
        return Long.parseLong(claims.getSubject());
    }

    // 토큰 파싱 및 검증
    public Claims validateToken(String token) {
        return Jwts.parser()
                .verifyWith(secretKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

<br>

### Refresh Token에는 왜 데이터를 적게 넣을까요?
Refresh Token은 오직 Access Token을 재발급받는 용도로만 사용됩니다. 이 토큰을 들고 직접 API 자원을 요청할 수 없기 때문에 굳이 권한 정보를 가질 필요가 없습니다.
따라서 `role` 같은 권한 정보는 넣지 않으며, 만료시간과 누구의 토큰인지만 알면 충분합니다. 

<br>

### 2. 비즈니스 계층에서의 토큰 발급 및 재발급 (`AuthService.java`)
로그인 혹은 회원가입 시 두 가지 토큰을 모두 발급하여 반환하고, Refresh Token은 DB에 저장하여 관리합니다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class AuthService {

    private final MemberRepository memberRepository;
    private final RefreshTokenRepository refreshTokenRepository;
    private final JwtTokenProvider jwtTokenProvider;
    
    @Transactional
    public LoginResponse login(String email, String password) {
        // ... 사용자 및 패스워드 검증 로직 생략
        
        // 1. 기존 Refresh Token 삭제 (중복 로그인 방지 또는 갱신)
        refreshTokenRepository.deleteByMemberId(member.getId());

        // 2. 새로운 토큰 쌍 발급
        String accessToken  = jwtTokenProvider.generateAccessToken(member.getId(), member.getRole());
        String refreshToken = issueRefreshToken(member.getId()); // DB 저장 포함

        return LoginResponse.of(accessToken, refreshToken);
    }

    @Transactional
    public TokenRefreshResponse refresh(String rawRefreshToken) {
        // 1. DB에 저장된 Refresh Token 조회 및 유효성 검증
        RefreshToken stored = refreshTokenRepository.findByToken(rawRefreshToken)
                .orElseThrow(() -> new BusinessException(ErrorCode.REFRESH_TOKEN_INVALID));

        if (stored.isRevoked() || stored.getExpiresAt().isBefore(LocalDateTime.now())) {
            throw new BusinessException(ErrorCode.REFRESH_TOKEN_INVALID);
        }

        Member member = memberRepository.findById(stored.getMemberId())
                .orElseThrow(() -> new BusinessException(ErrorCode.MEMBER_NOT_FOUND));

        // 2. 사용 후 폐기 (RTR 방식 - Refresh Token Rotation)
        refreshTokenRepository.deleteByMemberId(member.getId());

        // 3. 새로운 토큰 쌍 발급
        String newAccessToken  = jwtTokenProvider.generateAccessToken(member.getId(), member.getRole());
        String newRefreshToken = issueRefreshToken(member.getId());

        return TokenRefreshResponse.of(newAccessToken, newRefreshToken);
    }

    private String issueRefreshToken(Long memberId) {
        String rawToken = jwtTokenProvider.generateRefreshToken(memberId);
        LocalDateTime expiresAt = jwtTokenProvider.getRefreshTokenExpiry();

        refreshTokenRepository.save(RefreshToken.builder()
                .memberId(memberId)
                .token(rawToken)
                .expiresAt(expiresAt)
                .build());

        return rawToken;
    }
}
```

<br>

### 토큰 재발급시 왜 Refresh Token까지 새로 발급할까요?
토큰 재발급을 요청할 때 Access Token만 갱신하지 않고 Refresh Token도 같이 발급하며 이전 것을 즉시 삭제합니다. 
<br>
이 방식을 **RTR (Refresh Token Rotation)** 이라고 합니다. 
<br>
이렇게 하면 만약 해커가 Refresh Token을 탈취하더라도, 정상적인 사용자가 한 번 토큰을 갱신하면 해커가 가진 토큰은 DB에서 삭제되어 무효화되므로 보안성이 더욱 강화됩니다.

<br>

## 3. 로그인 요청이 들어왔을 때 벌어지는 일

클라이언트가 로그인 요청을 보낼 때 서버에서 벌어지는 과정

```java
@PostMapping("/login")
@ResponseStatus(HttpStatus.OK)
public LoginResponse login(@RequestBody @Valid LoginRequest request) {
    return authService.login(request.getEmail(), request.getPassword());
}
```

위와 같이 컨트롤러로 로그인 요청이 들어오면 아래와 같은 순서로 인증과 토큰 발급이 이루어집니다.

1. ### **요청 도달 및 필터 통과:**
   클라이언트가 이메일과 비밀번호를 담아 로그인 API를 호출합니다.
   <br>
   이 요청은 가장 먼저 스프링 시큐리티의 **필터 체인**을 거칩니다. 우리가 설정한 `JwtAuthenticationFilter`가 동작하지만, 아직 로그인 전이라 헤더에 토큰이 없으므로 컨텍스트에 아무것도 저장하지 않고 다음 단계로 넘어갑니다. 
   <br>
   (`SecurityConfig`에서 로그인 경로를 `permitAll()`로 설정했기 때문에 인증 없이도 에러 없이 컨트롤러로 진입할 수 있습니다.)

2. ### **컨트롤러 -> 서비스 계층 위임:**
   `AuthController`는 클라이언트가 보낸 데이터(이메일, 비밀번호)를 `AuthService`의 `login` 메서드로 넘깁니다.

3. ### **회원 조회 및 비밀번호 검증 (인증):**
   `AuthService`는 DB에서 해당 이메일을 가진 회원을 조회합니다. 회원이 존재하면, 입력한 평문 비밀번호와 DB에 저장된 암호화된 비밀번호를 `PasswordEncoder`의 `matches()` 기능을 이용해 비교 검증합니다.

4. ### **기존 토큰 정리:**
   비밀번호가 일치하여 인증에 성공하면, 혹시 모를 중복 토큰 이슈를 막기 위해 DB에 해당 회원 아이디로 저장되어 있던 기존 Refresh Token을 삭제합니다.

5. ### **신규 토큰 생성 및 DB 저장:**
   `JwtTokenProvider`를 호출하여 새로운 **Access Token**(비즈니스 로직 및 API 접근에 사용, 유효기간 짧음)과 **Refresh Token**(토큰 갱신용, 유효기간 김)을 발급합니다. 발급된 Refresh Token은 만료시간 등과 함께 DB에 저장됩니다.

6. ### **응답 반환:**
   발급된 두 개의 토큰을 `LoginResponse` 객체에 담아 컨트롤러로 반환하고, 최종적으로 클라이언트에게 JSON 형태로 토큰이 전달됩니다.

7. ### **이후의 권한이 필요한 API 요청:**
   이후 클라이언트는 방금 발급받은 Access Token을 HTTP 헤더(`Authorization: Bearer ...`)에 담아 API를 호출합니다. 그러면 다시 1번 단계의 `JwtAuthenticationFilter`가 이 토큰을 가로채어 정상적인 토큰인지 파싱(검증)하고, 유효하다면 사용자를 식별하여 접근을 허용하게 됩니다.

<br>

## +추가 코드 예시의 보충 설명

### DB를 통한 Refresh Token 관리 (`RefreshToken` 엔티티)
토큰을 데이터베이스에서 엔티티로 관리함으로써, 강제 로그아웃이나 회원 탈퇴 시 토큰을 무효화하는 로직을 쉽게 구현할 수 있습니다.
또한 탈취 의심 시 `is_revoked` 플래그를 true로 바꾸어 토큰을 강제 정지하는 정책도 구현할 수 있습니다.

<details>
<summary><b>RefreshToken 엔티티 보기 (클릭)</b></summary>

```java
@Entity
@Table(name = "refresh_token")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class RefreshToken {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "member_id", nullable = false)
    private Long memberId;

    @Column(name = "token", nullable = false, unique = true)
    private String token;

    @Column(name = "expires_at", nullable = false)
    private LocalDateTime expiresAt;

    @Column(name = "is_revoked", nullable = false)
    private boolean revoked = false;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();

    @Builder
    public RefreshToken(Long memberId, String token, LocalDateTime expiresAt) {
        this.memberId = memberId;
        this.token = token;
        this.expiresAt = expiresAt;
    }

    public void revoke() {
        this.revoked = true;
    }
}
```
</details>

