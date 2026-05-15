# DTO와 Record

* DTO란 Data Transfer Object (데이터 전송 객체)
* Record란 Java 14에서 도입된 불변 데이터 캐리어(정식 16버전)

<br>

# DTO란?

클라이언트와 서버, 또는 계층(Layer)간에 데이터를 전송하기 위한 객체입니다.

즉, 로직을 담지 않고 순수하게 데이터만을 담아서 전달하는 역할을 합니다.

예를 들어 API 요청/응답, 서버내에서 순수하게 데이터를 전달, 데이터베이스 조회 결과를 클라이언트에게 전달할 때 사용됩니다.

<br>

## DTO의 표현 방식

```java
public class UserDTO {

    private String name;

    private String email;

    private int age;

    public UserDTO() {
    }

    public UserDTO(String name, String email, int age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "UserDTO{" +
                "name='" + name + '\'' +
                ", email='" + email + '\'' +
                ", age=" + age +
                '}';
    }

    .
    .
    .
}
```

<br>

* 위와 같이 필드, Getter/Setter, 생성자, toString 등 모두 직접 작성해야합니다.


```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@ToString

public class UserDTO {
    private String name;
    private String email;
    private int age;
}
```
>위와같이 Lombok을 사용하면 더욱 편리하게 표현 가능합니다.

<br>

# Record란?

Java 16부터 정식 도입된 기능으로, 불변(Immutable) 데이터를 담기 위한 특별한 클래스입니다.

DTO와 비슷하게 데이터 전달 목적으로 사용되지만, 자동으로 많은 메서드를 생성해줍니다.

간결한 문법으로 getter, equals(), hashCode(), toString() 등이 자동으로 만들어집니다.

<br>

## Record의 표현 방식

```java
public record UserRecord(String name, String email, int age) {}
```

<br>

* 단 한 줄로 DTO를 정의할 수 있습니다.
* Record는 기본적으로 모든 필드가 **final**이며 private입니다.
* 자동으로 생성되는 것들:
  - 생성자 (모든 필드를 받는 생성자)
  - Getter (필드명 그대로: name(), email(), age())
  - equals()
  - hashCode()
  - toString()

<br>

# DTO 와 Record 차이점

### DTO (Lombok 없이)
```java
public class UserDTO {
    private String name;
    private String email;
    private int age;

    public UserDTO() {
    }

    public UserDTO(String name, String email, int age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "UserDTO{" +
                "name='" + name + '\'' +
                ", email='" + email + '\'' +
                ", age=" + age +
                '}';
    }
}
```

### DTO (Lombok 사용시)
```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@ToString
public class UserDTO {
    private String name;
    private String email;
    private int age;
}
```

### Record
```java
public record UserRecord(String name, String email, int age) {}
```

<br>

## 사용 방법의 차이

### DTO 사용
```java
// 기본 생성자 사용
UserDTO user1 = new UserDTO();
user1.setName("test");
user1.setEmail("test@example.com");
user1.setAge(20);

// 생성자로 생성
UserDTO user2 = new UserDTO("test", "test@example.com", 30);

String tempName = user2.getName(); 
String tempEmail = user2.getEmail(); 

user2.setAge(31);
```

### Record 사용
```java
// 생성자로 생성 (기본 생성자 없음)
UserRecord user1 = new UserRecord("test", "test@example.com", 20);

String tempName = user1.name(); 
String tempEmail = user1.email(); 

// 불변이므로 수정 불가능
// user1.setAge(31); <- 불가능, Setter가 없음

// 다른 값으로 새 객체 생성
UserRecord user2 = new UserRecord(user1.name(), user1.email(), 30);
```

<br>

# DTO의 장단점

## 장점

* **유연성**: Getter/Setter를 통해 필드를 언제든지 수정할 수 있습니다.
* **호환성**: 오래된 자바 버전에서도 사용 가능합니다.
* **관례**: 많은 프레임워크(Spring, JPA 등)가 Getter/Setter를 기반으로 작동합니다.
* **부분 생성**: 기본 생성자로 객체를 생성 후 필요한 필드만 커스텀하여 설정 가능합니다.

>현재 자바 17버전이 약61%정도로 사용되고 있으며 레거시의 경우 11, 8버전이 30~35% 정도로 사용되고 있습니다  
## 단점

* **보일러플레이트 코드**: Getter/Setter, toString, equals 등을 모두 작성해야 합니다.
* **불변성 미보장**: Setter를 통해 언제든지 값이 변경될 수 있어 예상치 못한 버그가 발생할 수 있습니다.
* **가독성 저하**: 코드가 길어져 가독성이 떨어집니다.

>보일러플레이트 코드: 컴퓨터 프로그래밍에서 수정하지 않거나 최소한만 수정하여 여러 곳에서 반복적으로 재사용하는 표준화된 코드 형태이며,
<br>
쉽게 말해 반드시 적어야하는 필수 준비 코드 정도이며 이를 해결하기 위해 Lombok, record를 경우에따라 사용합니다.

<br>

# Record의 장단점

## 장점

* **간결함**: 한 줄의 코드로 불변 데이터 객체를 정의할 수 있습니다.
* **불변성**: 모든 필드가 final이므로 생성 후 변경이 불가능하여 데이터 무결성이 보장되며, 동시에 사용자가 값을 변경하는 실수를 방지합니다.
* **자동 생성**: equals(), hashCode(), toString() 등이 자동으로 생성됩니다.
* **명확성**: Record임을 선언함으로써 "이것은 데이터 전달 객체"임이 명확합니다.

## 단점

* **자바 버전 제한**: Java 16 이상에서만 사용 가능합니다.
* **수정 불가**: 생성 후 필드를 변경할 수 없어 값 수정이 필요하면 새 객체를 만들어야 합니다.
* **기본 생성자 부재**: 기본 생성자(파라미터 없는 생성자)가 없습니다.

<br>

# "언제" 그리고 "어떻게" 쓰이는가?

## DTO를 사용하는 경우

### 1. 값 수정이 필요한 경우
```java
// API 요청 처리 중 데이터 변환이 필요할 때
@PostMapping("/users")
public ResponseEntity<UserDTO> createUser(UserDTO dto) {
    dto.setName(dto.getName().trim()); // 공백 제거
    userService.save(dto);
    return ResponseEntity.ok(dto);
}
```

### 2. 기본 생성자가 필요한 경우
```java
// Spring Data JPA와 같은 기본 생성자가 꼭 있어야 하는 경우
@Entity
public class User {
    // Entity는 보통 DTO로 변환되어 전달
}

@Getter
@Setter
@NoArgsConstructor
public class UserDTO { // 기본 생성자 필요
    private String name;
    private String email;
}
```

### 3. 레거시 프로젝트인 경우
```java
// 오래된 자바 버전을 사용 중일 때
public class UserDTO { // Java 8 이상만으로도 사용 가능
    private String name;
    // ...
}
```

<br>

## Record를 사용하는 경우

### 1. 데이터가 변경되지 않는 경우
```java
// API 응답으로 조회된 사용자 정보 (읽기만 함)
public record UserResponse(String name, String email, int age) {
}

@GetMapping("/users/{id}")
public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
    UserResponse response = userService.findUserAsRecord(id);
    return ResponseEntity.ok(response); // 응답 후 수정 없음
}
```

### 2. 불변성이 중요한 경우
```java
// 스레드 안전이 필요한 경우
public record ImmutableUserData(String name, String email, int age) {
}

// 여러 스레드에서 동시에 접근해도 안전
Map<Long, ImmutableUserData> cache = new ConcurrentHashMap<>();
```
>Concurrent가 붙으면 동시성을 제어하는 기능이 붙는다고 생각하면 편합니다.
### 3. 데이터 구조가 단순한 경우
```java
// 응답 DTO가 단순할 때
public record LoginResponse(String token, long expiresAt) {
}

public record ErrorResponse(int code, String message) {
}
```

### 4. 모던 프로젝트에서
```java
// Java 16버전 이상 프로젝트에서 신규 개발할 때
public record CreateUserRequest(String name, String email, int age) {
}

@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(CreateUserRequest request) {
    // request는 불변이므로 안전함
    return ResponseEntity.ok(userService.create(request));
}
```

<br>

# 정리

| 항목 | DTO | Record |
|------|-----|--------|
| **자바 버전** | 모든 버전 | Java 16+ |
| **수정 가능** | 가능 (Setter) | 불가능 (final) |
| **기본 생성자** | 있음 | 없음 |
| **코드량** | 많음(Lombok으로 해결 가능) | 매우 적음 |
| **불변성** | 미보장 | 보장 |
| **용도** | 유연한 데이터 전달 | 불변 데이터 전달 |
| **프레임워크 호환성** | 우수 | 상대적으로 낮음 |

<br>

**결론**: 
* **DTO**: 로직 내에서 값 수정이 필요하거나, 기본 생성자가 필요하거나, 레거시 환경일때 사용합니다.
* **Record**: 데이터가 불변이며, 신규 개발 환경에서 간결한 코드를 원할 때 사용