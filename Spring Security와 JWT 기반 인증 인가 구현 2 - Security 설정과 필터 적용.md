# Spring Security와 JWT 기반 인증 인가 구현 2 - Security 설정과 필터 적용

프로젝트를 진행하면서 새롭게 배우고 적용한 부분에 대해서 다시 한번 보고 정리를 하고자 작성하였습니다.
<br>
이 글은 REST API 서버 환경에서 Spring Security와 JWT를 도입하여 인증/인가를 어떻게 구현했는지와, <br>
Spring Security의 기본 세션 방식이 아닌 Stateless한 필터 기반의 인증 구조를 어떻게 설계했는지에 대해 정리하였습니다.

<br>

## 1. REST API와 JWT 방식의 도입

### 문제 상황

일반적인 스프링 웹 애플리케이션은 서버의 메모리에 세션을 저장하여 사용자의 로그인 상태를 유지합니다.
하지만 현재 프로젝트는 프론트엔드(React/Vue 등)와 백엔드가 분리된 REST API 서버 형식으로 동작하며, 여러 서버 인스턴스를 띄우거나 확장성을 고려할 때 상태를 유지하는 세션 보다는 토큰이 더 적합하여 JWT를 사용하여야 하는 상황입니다.

### 해결
세션을 사용하지 않는 **Stateless**한 특징을 가진 **JWT**를 도입하여 인증을 처리하였습니다.
> 클라이언트가 로그인하면 서버는 토큰(JWT)을 발급하고, 이후 요청마다 클라이언트는 헤더에 토큰을 담아 보냅니다. 서버는 세션을 조회할 필요 없이 토큰의 유효성만 검증하여 사용자를 식별합니다.

<br>

## 2. 패턴 구현 방법 및 연동

### 1. Spring Security 환경 설정 (`SecurityConfig.java`)
가장 먼저 해야 할 일은 Spring Security의 기본 동작을 본인 프로젝트에 맞게(세션을 끄고, 폼 로그인을 끄는 등) 수정해야합니다.

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        http
                .cors(cors -> cors.configurationSource(corsConfigurationSource()))
                .csrf(AbstractHttpConfigurer::disable) // 1. CSRF 비활성화(세션 환경에서는 켜야함)

                // 2. 세션을 사용하지 않도록 STATELESS 설정
                .sessionManagement(session ->
                        session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

                // 3. URI별 접근 권한 설정
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/error").permitAll()//임시
                        .requestMatchers("/api/v1/auth/**").permitAll()
                        .requestMatchers(HttpMethod.GET,
                                "/api/v1/regions/**",
                                "/api/v1/pois/**",
                                "/api/v1/search/**",
                                "/api/v1/share/**").permitAll()
                        .anyRequest().authenticated())

                // 4. 예외 처리
                .exceptionHandling(ex -> ex
                        .authenticationEntryPoint((request, response, authException) ->
                                response.sendError(401, "Unauthorized")))

                // 5. 커스텀 JWT 필터 등록
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

<br>

### 왜 세션을 끄고 CSRF를 비활성화 할까요?
REST API 환경에서는 클라이언트가 매 요청마다 토큰을 헤더에 담아 보내기 때문에 서버가 상태를 기억할 필요가 없습니다. 
따라서 `SessionCreationPolicy.STATELESS`로 설정하여 불필요한 세션 생성을 막습니다.
또한 CSRF(Cross-Site Request Forgery) 공격은 주로 세션/쿠키 기반 인증에서 발생하므로, 토큰 기반 인증에서는 비활성화(`disable()`)하는 것이 일반적입니다.

<br>

### 2. JWT 인증 필터 구현 (`JwtAuthenticationFilter.java`)
Spring Security의 필터 체인에서 가장 핵심적인 역할을 할 커스텀 필터입니다. 요청이 들어올 때마다 헤더에서 JWT를 꺼내 검증하고, 유효하다면 SecurityContext에 인증 객체를 담습니다.

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    protected void doFilterInternal(final HttpServletRequest request,
                                    final HttpServletResponse response,
                                    final FilterChain filterChain)
            throws ServletException, IOException {

        try {
            // 1. 요청 헤더에서 토큰 추출
            String token = AuthorizationExtractor.extract(request);
            
            // 2. 토큰 검증 및 클레임(데이터) 조회
            Claims claims = jwtTokenProvider.validateToken(token);
            Long memberId = jwtTokenProvider.extractMemberId(claims);

            // 3. 인증 객체 생성 및 SecurityContext에 저장
            UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(memberId, null, List.of());
            SecurityContextHolder.getContext().setAuthentication(authentication);

        } catch (Exception e) {
            // 토큰이 없거나 유효하지 않으면 컨텍스트 초기화
            SecurityContextHolder.clearContext();
        }

        // 다음 필터로 이동
        filterChain.doFilter(request, response);
    }
}
```

<br>

### 필터는 구체적으로 어떤 역할을 할까요?
클라이언트가 요청을 보내면 가장 먼저 `JwtAuthenticationFilter`를 거칩니다. (설정에서 `UsernamePasswordAuthenticationFilter` 이전에 동작하도록 등록했음)
헤더에 유효한 토큰이 있다면 `SecurityContextHolder`에 해당 유저의 정보(`memberId`)를 담습니다. 이렇게 하면 이후의 컨트롤러나 서비스 계층에서 `@AuthenticationPrincipal` 혹은 컨텍스트 조회를 통해 현재 로그인한 사용자가 누구인지 바로 알 수 있습니다. 토큰이 잘못되었다면 인증 객체를 저장하지 않으므로, SecurityConfig의 설정에 따라 401(Unauthorized) 에러가 발생하게 됩니다.

<br>

## +추가 코드 예시의 보충 설명

<br>

### 1. CORS(Cross-Origin Resource Sharing) 설정
프론트엔드와 백엔드 서버의 도메인/포트가 다를 경우 브라우저는 기본적으로 API 요청을 차단합니다. 이를 해결하기 위해 `SecurityConfig`에 CORS 정책을 설정해야 합니다.

<details>
<summary><b>corsConfigurationSource 보기 (클릭)</b></summary>

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    // 프론트엔드 주소 허용
    configuration.setAllowedOrigins(List.of("http://localhost:5173", "http://localhost:3000"));
    configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
    configuration.setAllowedHeaders(List.of("*"));
    configuration.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```
</details>

<br>

### 2. @EnableWebSecurity 와 PasswordEncoder
`SecurityConfig` 클래스를 보면 클래스 레벨에 `@EnableWebSecurity` 어노테이션이 붙어있고, 내부에 `passwordEncoder()` 빈 등록 로직이 있습니다.

**@EnableWebSecurity 란?**
<br>
이 어노테이션을 붙이면 스프링 부트가 제공하는 기본적인 웹 보안 설정을 우리가 직접 커스터마이징(`SecurityFilterChain` 빈 등록 등)할 수 있도록 스프링 시큐리티 필터 체인을 활성화해 줍니다. 

**PasswordEncoder 란?**
<br>
회원가입 시 사용자의 비밀번호를 DB에 그대로 저장하면 해킹 시 모든 비밀번호가 유출되는 심각한 문제가 생깁니다.
따라서 스프링 시큐리티에서 제공하는 `PasswordEncoder`(구체적으로는 `BCryptPasswordEncoder`)를 빈으로 등록하여, 비밀번호를 단방향 암호화(해싱)한 뒤 저장하도록 합니다. 
<br>
로그인 시에도 이 객체를 통해 사용자가 입력한 비밀번호와 DB의 암호화된 비밀번호가 일치하는지(`matches`) 검증하게 됩니다.

<details>
<summary><b>PasswordEncoder 빈 등록 코드 보기 (클릭)</b></summary>

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```
</details>

<br>
