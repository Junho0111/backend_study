# IOC, DIP란?

* IOC란 제어의 역전(Inversion of Control)
* DIP란 의존 역전 원칙(Dependency Inversion Principle)

<br>

# 문제
손님은 커스텀 햄버거가 먹고싶어 햄버거를 주문했는데 돌아온건 이미 정해진재료의 햄버거입니다

이렇게 햄버거가 직접 객체를 생성하여 코드를 제어하는 것을 제어권이 내부에 있다고 하는데 이렇게되면 손님이 원하는데로 햄버거를 만들수없으며 손님은 이미 만들어진 햄버거만을 먹어야하는 문제가 생깁니다.

<br>

손님이 원하는데로 햄버거를 커스텀하려면 어떻게해야할까요??

```java
public class CustomHamburger {
    BeefPatty beefPatty; // 소고기 패티
    CheddarCheese cheddarCheese; // 체다 치즈
    Bread bread; // 빵

    public Hamburger() {
        this.beefPatty = new BeefPatty();
        this.cheddarCheese = new CheddarCheese();
        this.bread = new Bread();
    }
}
```
<br>

# 해결 과정

* Ioc적용 제어권 외부로 넘김
```java
public class CustomHamburger {
    BeefPatty beefPatty; // 소고기 패티
    CheddarCheese cheddarCheese; // 체다 치즈
    Bread bread; // 빵

    public Hamburger(BeefPatty beefPatty, CheddarCheese cheddarCheese, Bread bread) {
        this.beefPatty = beefPatty;
        this.cheddarCheese = cheddarCheese;
        this.bread = bread;
    }
}
```

<br>

* Dip적용 고수준모듈과 저수준 모듈 모두 추상화에 의존
```java
public class CustomHamburger {
    Patty patty; // 패티 인터페이스
    Cheese cheese; // 치즈 인터페이스
    Bread bread; // 빵 인터페이스

    public Hamburger(Patty patty, Cheese cheese, Bread bread) {
        this.patty = patty;
        this.cheese = cheese;
        this.bread = bread;
    }
}
```
<br>
<br>

# 해결

생성자에 직접 재료를 받아 만들게 하였는데, 이렇게 밖에서 값을 받아 만들어지는것을 제어권이 외부에 있다라고 하여 IOC 제어의 역전이라고합니다. 또한

인터페이스에 의존한 덕분에 이제 손님이 원하는대로 햄버거를 만들 수 있습니다
<br> 즉 상위 레벨의 모듈은 하위 레벨의 모듈에 의존하지 않아야하며, 
둘다 추상화에 의존해야합니다. 햄버거도 **치즈**라는 인터페이스에 의존하고 구체적인 치즈 또한 치즈라는 인터페이스 덕분에 어떤 치즈가 들어와도 문제가 없습니다 

<br> 그리고 이를 DIP 의존 역적의 원칙이라고 합니다.
<br> **햄버거 -> 각각의 재료 인터페이스 <- 각각의 구체적 재료** (둘다 추상화에 의존함)


<br>결과적으로 IOC원칙과 DIP원칙을 적용하여 문제를 해결하게되었습니다. 

```java
public class CustomHamburger {
    Patty patty; // 패티 인터페이스
    Cheese cheese; // 치즈 인터페이스
    Bread bread; // 빵 인터페이스

    public Hamburger(Patty patty, Cheese cheese, Bread bread) {
        this.patty = patty;
        this.cheese = cheese;
        this.bread = bread;
    }
}
```

<br>
<br>

# DI란?
* DI란 의존관계 주입(Dependency Injection)
<br>1. 생성자 주입 
<br>2. Setter 주입
<br>3. Interface 주입

<br>

위와 같은 방법들이 있으며, 덕분에 의존하는 대상에 변화가 생기면 의존받는 대상도 그에따라 문제없이 변할 수 있으며 변화로부터 자유로워지는 디자인 패턴입니다.


<br>
<br>

# Spring DI

* 스프링은 빈으로 등록이 되면 스프링이 자동으로 인스턴스를 생성하며 이때 필요한 의존성도 자동으로 주입해줍니다.

```java
@Controller
public class Controller {
    private final Service service;

    public controller(Service service) {
        this.service = service;
    }
}
```

* 이 경우에 어떻게 의존성을 주입해줘야할까요?

<br>

## 필드 주입

```java
@Controller
public class Controller {

    @Autowired
    private final Service service;
}
```

* 이렇게 주입받을곳에 **@Autowired**를 붙여서 스프링에게 알려주면 스프링이 자동으로 주입해주게됩니다.(단 주입받는 service도 빈으로 등록되어있어야함.)

* 하지만 이는 의존성이 프레임워크에 강하게 의존되어 최근에는 사용되지 않습니다

<br>

## 생성자 주입
```java
@Controller
public class Controller {
    private final Service service;

    public controller(Service service) {
        this.service = service;
    }
}
```

* 스프링이 최근에 추천하는 방식으로써 객체를 최초에 생성하는 시점에 스프링이 의존성을 자동으로 주입해줍니다(단 주입받는 service는 빈으로 등록이 되어있어야합니다)

* 추가로 스프링 4.3부터는 위의 코드처럼 생성자가 1개만 있을시 @Autowired를 생략할 수 있습니다.

<br>

## 번외(Lombok)

```java
@Controller
@RequiredArgsConstructor // final이 붙은 필드에 한하여 생성자를 자동으로 만들어줌
public class Controller {
    private final Service Service; 
}
```

* 최근에는 Lombok을 활용하여 더욱 간결하게 코드를 가져갑니다.
<br>(단 주입받는 service는 빈으로 등록이 되어있어야합니다)

<br>

# 생성자가 여러개일때는?
* 의존성을 자동으로 주입하는데 사용할 생성자에 @Autowired를 붙여줍니다

* @Autowired가 여러개 있을경우에는 가장 많은 의존성을 주입할 수 있는 생성자를 사용합니다.

* @Autowired가 붙은 모든 생성자가 사용 불가능한 경우 혹은 어떤 생성자에도 @Autowired가 붙지 않았을 경우에는 기본 생성자를 호출합니다

* 위 상황에서 기본 생성자까지 없을 경우에는 컴파일 에러가 납니다.

* 우선순위 (생성자 -> 필드 순)

<br>

# 주입 받는 것의 타입은 같고 그 개수가 여러개일때는?

## @Qualifier

```java
@Service
@Qualifier("mainService")
public class ServiceA implemnet Service {
    ...
}
```

```java
@Controller
public class Controller {
    private final Service service;

    public controller(@Qualifier("mainService") Service service) {
        this.service = service;
    }
}
```

* 이렇게 해주면 서비스가 여러개여도 @Qualifier("mainService")가 붙은 서비스를 우선적으로 가져옵니다.

## @Primary

```java
@Service
@Primary
public class ServiceA implemnet Service {
    ...
}
```

```java
@Controller
public class Controller {
    private final Service service;

    public controller(Service service) {
        this.service = service;
    }
}
```

* 이렇게 @Primary를 붙여주면 그 해당 빈은 해당 타입으로 의존성을 검색할떄 우선적으로 주입이 됩니다.

<br>
<br>

# 만약 둘다 붙어있으면 누가 우선인가?

* 결론 부터말하면 @Qualifier가 붙은것이 우선적이며 이는 기본적으로 "이게 우선이야"라고 정한것보다 사용자가 직접 정의한것에 우선순위를 먼저 주는것입니다

<br>

## 정리

타입 -> @Qualifier -> @Primary -> 변수명

* (변수명으로 우선순위를 정하면 나중에 문제가 햇갈려서 문제 생길 수 있음)