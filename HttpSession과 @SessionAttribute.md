# 세션 사용배경

* **배경**: 쿠키를 통하여 로그인 또는 stateLess상태의 서버에서 해당 유저정보를 계속 받는 등의 코드는 심각한 보안문제가 있습니다.<br>

* **문제**: 
<br>1. 쿠키 값은 임의로 변경할 수 있다.
<br>2. 쿠키에 보관된 정보는 훔쳐갈 수 있다. 
<br>3. 해커가 쿠키를 한번 훔쳐가면 평생 사용할 수 있다. 등등 <br>
<br>쿠키에 중요한 정보를 보관하는 방법은 여러가지 보안 이슈가 있습니다. 
따라서 이 문제를 해결하려면 결국 중요한 정보를 모두 서버에 저장해야 합니다.<br>

* **질문**: 이때 클라이언트와 서버는 추정 불가능한 임의의 식별자 값으로 연결해야 합니다.<br>
이렇게 서버에 중요한 정보를 보관하고 연결을 유지하는 방법을 세션이라고 하는데, 이 세션이 동작하는 과정이 무엇일까요?.

<br>

# 동작과정

사용자가 아이디와 비밀번호 정보를 서버에 전달하면 
서버에서는 해당 사용자가 맞는지 확인 후 추정 불가능한 세션 ID(UUID)를 생성합니다.

>세션ID = zz0101xx-bab9-4b92-9b32-dadb280f4b61<br>
httpSession으로 생성 시 JSESSIONID=5B78E23B513F50164D6FDD8C97B0AD05

<br>

이렇게 생성된 세션ID와 세션에 보관할 유저정보를 서버의 세션저장소에 key value형태(map)로 저장하게됩니다.

<br>

서버는 클라이언트에게 해당 세션ID만을 담은 
쿠키를 전달하게됩니다.
그리고 클라이언트는 쿠키저장소에 해당 쿠키를 보관합니다.

<br>

로그인 이후에는 클라이언트가 서버에 요청을 보낼때마다 항상 세션ID가 담긴 쿠키를 전달하게됩니다.

<br>

그리고 서버는 해당 쿠키에서 세션ID를 꺼내어 
세션저장소를 조회하며 로그인시 보관한 세션정보를 사용하게 됩니다.

<br>

# 직접 Session 구현 예시
* 세션은 아래와같은 3가지 기능으로 구현이 되며 동작합니다.
<br> 1. 세션 생성 
<br>2. 세션 조회 
<br>3. 세션 만료

<br>

```java
//생성
public static final String SESSION_COOKIE_NAME = "mySessionId"; // 쿠키이름 상수
private Map<String, Object> sessionStore = new ConcurrentHashMap<>(); // 세션 저장소는 key/value형태인 map으로 관리되며 동시성을 고려하여 Concurrent로 만들어짐

//생성
public void createSession(Object value, HttpServletResponse response) {
    //세션 id를 생성하고, 값을 세션에 저장
    String sessionId = UUID.randomUUID().toString();
        sessionStore.put(sessionId, value);
     //쿠키 생성
    Cookie mySessionCookie = new Cookie(SESSION_COOKIE_NAME, sessionId);
    response.addCookie(mySessionCookie);
}

```

<br>

```java
//조회
public Object getSession(HttpServletRequest request) {
Cookie sessionCookie = findCookie(request, SESSION_COOKIE_NAME);
    if (sessionCookie == null) {
        return null;
        }
    return sessionStore.get(sessionCookie.getValue());
}

```

<br>

```java
//만료
public void expire(HttpServletRequest request) {
    Cookie sessionCookie = findCookie(request, SESSION_COOKIE_NAME);
    if (sessionCookie != null) {
        sessionStore.remove(sessionCookie.getValue());
        }
    }
    
    private Cookie findCookie(HttpServletRequest request, String cookieName) {
        if (request.getCookies() == null) {
            return null;
        }
        
        return Arrays.stream(request.getCookies())
                .filter(cookie -> cookie.getName().equals(cookieName))
                .findAny()
                .orElse(null);
    }
```
<br>

# HttpSession사용 예시 
* 자바에서 제공하는 라이브러리로(jakarta.servlet.
httpHttpSession) 위의 직접 구현한 세션과 동일하게 동작합니다.

<br>

* **세션 생성 예시**
```java
    // HttpSession에 데이터를 보관하고 조회할 때, 같은 이름이중복 되어 사용되므로 상수를 정의
    // 이때 request는 HttpServletRequest request 
    
    //세션이 있으면 있는 세션 반환, 없으면 신규 세션 생성
    //만약 getSession(false)이면 없으면 신규 생성X하고 null반환
    HttpSession session = request.getSession();

    //세션에 로그인 회원 정보 보관
    session.setAttribute(SessionConst.LOGIN_MEMBER, loginMember);

```
<br>

* **세션 삭제 예시**
```java
//세션을 삭제한다.
HttpSession session = request.getSession(false);
if (session != null) {
        session.invalidate(); //세션을 제거.
    }

```

* 이렇듯 HttpSession은 세션을 만들고 삭제하는 등의 기능을 지원합니다.

>세션을 삭제할때 오해하지 말아야하는것이 회원가입을 하여 회원이 저장되어있는 저장소는 따로있는것이고 여기서 map형태로 관리되는 세션저장소에서 제거한다고하여 저장된 회원정보가 사라지는 것이 아닙니다.

<br>

# @SessionAttribute 사용 예시
* 스프링이 제공하는 것으로(org.springframework.web.bind.annotation.SessionAttribute;)<br>세션을 더 편리하게 사용할 수 있도록 지원합니다.그리고 보통 이미 로그인된 사용자를 찾을 떄 사용합니다.<br>
>@SessionAttribute는 세션을 생성하지 않습니다

<br>

# 사용 예시

```java
@GetMapping("/")
public String home(@SessionAttribute(name = SessionConst.LOGIN_MEMBER, required = false) Member loginMember, Model model) {
    //세션에 회원 데이터가 없으면 home
    if (loginMember == null) {
        return "home";
    }

    //세션이 유지되면 로그인으로 이동
    model.addAttribute("member", loginMember);
    return "loginHome";
```
<br>

* 이렇듯 보통 파라미터로 받아 로그인 유무를 확인하는데 쓰입니다.
* 간단하게 홈화면을 요청(get)했을때 로그인 여부를 확인하여 유지가 되어있으면 해당 유저정보를 resource의 loginHome.html로 넘기고 로그인이 되어있지 않으면 일반 home.html로 넘기는 기능이다 
* required = false 이게 false가 되어있으면 로그인이 되어있지 않아도 오류가 나지 않음

<br>

# 세션 타임아웃

* **상황**: 
세션은 사용자가 로그아웃을 직접 호출해서 session.invalidate()
가 호출 되는 경우에 삭제됩니다.<br> 그런데 로그아웃을 선택하지 않고, 그냥 웹 브라우저를 종료하는 경우 (HTTP가 비 연결성)서버 입장에서는 해당 사용자가 웹 브라우저를 종료한 것인지 아닌지를 인식할 수 없습니다. 

<br>

* **문제**: 만약 로그아웃을 하지않고 브라우저를 종료하는 식의 종료를 하여 세션을 그대로 놔두게된다면?
<br>1. 세션과 관련된 쿠키를 탈취 당했을 경우 해당 쿠키로 악의적인 요청을 할 수 있습니다. 
<br>2. 세션은 기본적으로 메모리에 생성되는데 많은 사용자들의 세션 생성하면 그만큼의 메모리에 사용자들의 세션이 쌓이게 되는것입니다.

<br>

* **질문**:
그렇다면 서버에서 세션 데이터를 언제 삭제해야 하는지 다르게 말하면 언제까지 살려두어야할까요? 그리고 어떻게 설정을 해야할까요?

<br>

# 세션 타임아웃 예시

* 스프링부트로 설정 할 수있습니다.<br>(application.propertiesserver.servlet.session.timeout=60)
> 60초, 기본은 1800(30분) (글로벌 설정은 분 단위로 설정해야합니다.  60(1분), 120(2분) ...)

<br>

* 코드로도 설정이 가능합니다.
```java
session.setMaxInactiveInterval(1800); //1800초
```
<br>

* 해당 세션의 HTTP 요청이 있으면 최근 요청 시간 기준으로 설정한 시간으로 다시 초기화 됩니다.<br>
session.getLastAccessedTime(): 이 메서드로 최근 세션 접근 시간을 확인 할 수있습니다.

<br>

# 간단 질문
```java
HttpSession.getAttribute("user")
```

<br>

* 위의 상황에서 사용자A가 접속해도 "user"를 Key로 값을 가져오고, 사용자B가 접속해도 "user"를 Key로 가져옵니다.<br> 어떻게 같은 Key를 쓰는데 A와 B를 구분해서 값을 가져올까요?

<br>

>1단계 - 우선 쿠키에서 jsessionid를 꺼내어 세션저장소(jsessionid/해당회원 모든 정보(map형태))에서 jsessionid를 key로 해당유저의 모든정보가 담긴 value를 꺼내옵니다.<br>
<br>
2단계 - 꺼내온 value는 유저정보저장소(user/ 유저정보)에서 상수user라는 Key로 유저정보인 value를 꺼내올수 있습니다.<br>따라서 같은 key를 씀에도 불구하고 유저a와 유저b를 구분하여 값을 가져오게됩니다.
<br>
<br>추가로 만약 user말고도 user나이를 따로 빼어 age라는 상수로 나이값을 저장한다는 가정하에 유저정보 저장소는 아래와 같은 구조가됩니다.
<br>[user/유저정보]
<br>[age/유저나이]
<br>따라서 getAttribute("age")를 하면 해당유저 나이가 나오게되는것입니다.
 
 <br>

# 정리 및 교훈 

* 서블릿의 HttpSession이 제공하는 타임아웃 기능 덕분에 세션을 안전하고 편리하게 사용할 수 있으며.<br> 실무에서 주의
할 점은 세션에는 최소한의 데이터만 보관해야 한다는 것이고, (보관한 데이터 용량 * 사용자 수)로 세션의 메모리 사용량이 급격하게 늘어나서 장애로 이어질 수 있으니 적절히 타임아웃 시간을 설정해야 합니다.

* 로그인을 하였는지 확인만 하는 상황에서는 **@SessionAttribute**를 사용하는것이 적절해보이며, 로그인을 하여 세션을 생성하거나 로그아웃을 하여 삭제하는 경우 **HttpSession**을 사용하는것이 적절해보입니다.

