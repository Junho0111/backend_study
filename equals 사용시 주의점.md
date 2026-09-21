# NPE(NullPointerException) 방지를 위한 equals() 작성법

자바에서 문자열(String)을 비교할 때, 변수나 상수의 위치를 어떻게 두느냐에 따라 `NullPointerException`(NPE)을 방지할 수 있습니다.

## 1. 문제 상황 (NPE 발생 위험)
```java
String inputCategory = null; // 사용자가 실수로 값을 안 보냈거나 DB에 Null로 들어간 경우

// 안좋은 예시
if (inputCategory.equals("ATTRACTION")) { 
    // NPE 발생 => 애플리케이션이 에러로 멈춤
}
```
위 코드는 `inputCategory`가 `null`일 때, `null` 객체의 `.equals()` 메서드를 호출하려고 하므로 즉시 NPE가 발생하여 서버가 중단되거나 500 에러를 뱉게 됩니다.

<br>

## 2. 해결 방법 (상수/리터럴을 앞으로)
```java
String inputCategory = null;

// 좋은 예시
if ("ATTRACTION".equals(inputCategory)) {
    // 그냥 false를 반환하고 안전하게 넘어감
}
```
**ATTRACTION**이라는 문자열 상수(리터럴)는 `null`이 아님이 보장됩니다.
따라서 상수 객체를 앞에 두고 `.equals()`를 호출하면, 괄호 안에 들어오는 변수가 `null`이더라도 자바의 `String.equals()` 로직에 의해 자연스럽게 `false`를 반환하고 NPE가 발생하지 않습니다.
