## 예외를 다루는 Best Practice가 있을까?
자바의 예외에 대한 포스팅은 굳이 이 글이 아니고도 많은 블로그에서 찾아볼 수 있으므로 생략하겠다. 굳이 언급하자면 `Exception`을 상속 받으면 체크 예외로 지금 예외를 잡거나 던져야 한다. `RuntimeException`을 상속 받으면 컴파일러가 체크하지 않는 언체크 예외이다. 스프링에서는 하나의 트랜잭션에서 `RuntimeException`이 터지면 롤백되도록 설계되었다. 이 정도는 다 아는 내용이라고 생각하고 더는 생략하겠다.

오늘 이 글에서 하고 싶은 말은 **예외를 다루는 Best Practice**가 있을지 고민한 글이다.
그 중에서도,
> 커스텀 예외는 언제 만들어야 할까?

에 대해 글로 녹여봤다.

글을 쓰게된 것은 사내 프로젝트를 진행하면서 동료분과 있었던 일 때문이다. 나는 런타임 예외를 상속 받는 커스텀 예외를 만들기를 원했고, 동료 분은 굳이 만들지 말자고 하셨다. 동료분은 `IllegalArgumentException`으로도 충분하다고 하셨기 때문이다.  

하지만 내 생각은 달랐다. 나는 예외에도 이름이 있기를 원했다. 가독성 때문이었다. 예를 들어 JPA Repository에서 엔티티를 findById로 가져올 때 Optional로 감싸서 가져오는데, 이때 필연적으로 `orElseThrow` 등을 하여 가져온다. 이때 가독성 측면에서 `IllegalArgumentException`보다, `MemberNotFoundException`와 같은 이름이 있는 예외를 던지는 것이 좋다고 생각했기 때문이다.

```java
memberRepository.findById(id).orElseThrow(() -> new IllegalArgumentException("회원을 찾을 수 없습니다."));

memberRepository.findById(id).orElseThrow(() -> new MemberNotFoundException("회원을 찾을 수 없습니다."));
```

하지만 동료 분은 이런 말씀을 하셨다.
회원을 찾을 수 없는 예외는 결국 id가 잘못된 값이기 때문에 발생한 예외 아니냐. 그래서 `IllegalArgumentException`으로 왠만한 건 다 커버가 가능할 거 같다고 하셨다. 아직 우리 개발 팀의 인원이 많지 않은 상황이며, 예외 또한 유지 보수와 관리의 대상임을 감안하여 동료 분의 말을 따랐던 기억이 있다.

그러나 내 생각은 이러한 의견에는 정답이 없다고 생각한다. 그러던 중 스택 오버플로우에 나와 같은 고민을 하는 사람이 있어 공유해보고자 한다.

## 부가적인 정보가 없으면 커스텀 예외를 만들지 말자

아래의 코드를 보자. (아님 위의 내 코드 `MemberNotFoundException`를 보자.) 문제점이 무엇일까?
```java
public class DuplicateUsernameException extends RuntimeException {}
```
이 코드의 문제점은 **예외의 이름 말고는 유용한 정보를 제공하지 않는 것이다.** 우리는 Java의 예외 클래스가 여느 자바 클래스와 비슷하다는 것을 잊으면 안 된다. 즉, 커스텀 예외를 만들면 예외에 대한 보다 유용한 정보를 제공할 수 있는 메소드를 만들 수 있다. `DuplicateUsernameException`에 다음과 같은 유용한 메소드를 추가해보자.

```java
public class DuplicateUsernameException extends RuntimeException {
    public DuplicateUsernameException(String username){....}
    public String requestedUsername(){...}
    public String[] availableNames(){...}
}
```
새로운 버전에서는 두 가지 유용한 메소드를 제공한다. `requestedUsername()`는 요청된 이름을 반환하며 
`availableNames()`는 요청된 이름과 유사한 사용 가능한 이름들을 반환한다. 비로소 이 메소드들을 통해 사용 가능한 회원 이름과 사용 불가능한 회원 이름이 무엇인지 클라이언트에게 알려줄 수 있게 되었다. 

하지만 만약 **추가적인 정보가 없다면, 그냥 표준 예외를 던지자**.
```java
throw new RuntimeException("이미 사용 중인 회원 이름입니다.");
```

**참조**
> <a href='https://stackoverflow.com/questions/22698584/when-should-we-create-our-own-java-exception-classes' target='_blank'>When should we create our own Java exception classes?</a>

## 자바에서 제공하는 표준 예외들
다음은 자바에서 제공하는 표준 예외들이다. 런타임 예외(언체크 예외)를 상속 받는 예외들과 쓰임새를 정리하였다.
- `IllegalArgumentException` : 메서드에 전달된 인자가 유효하지 않을 때 발생합니다. 주로 메서드에 전달된 값이 허용 범위를 벗어나거나 예상하지 못한 형식일 때 발생합니다.
- `IllegalStateException` : 메서드가 특정 상태에서 호출되지 않았을 때 발생합니다. 예를 들어, 초기화되지 않은 객체에 대한 메서드 호출이 있을 경우 발생할 수 있습니다.

비즈니스 로직에서 사용자의 잘못된 요청으로 발생할 수 있는 대표적인 런타임 예외는 위 두 개로 대부분의 상황을 커버할 수 있다 생각하여 두 개만 기재하였다.

## 번외 - 예외에 Http 응답 코드를 기재하는 것은 안티패턴일까?
가령 이런 경우다.
1. 비즈니스 코드에서 생성자와 같은 메소드에 직접 Http Status Code를 기입
```java
void bizLogic() {
	...//생략
	if(isConflict(name)) {
		throw new ConflictException(409, "중복되는 이름입니다.");
	}
}
```
2. 이후 `ControllerAdvice`에서 처리
```java
@ExceptionHandler(ConflictException.class)
public ResponseEntity<?> handleCustomException(ConflictException e) {
  return new ResponseEntity<>(e.getMessage, HttpStatus.valueOf(e.getStatus()));
}
```

위 코드에서는 예외의 생성자에서 `Http 응답 코드`를 초기화한다. 이러한 코드는 앞으로 **무슨 행동이 일어날지 기대하게 만든다.** 가령 〈이름이 중복되면, `ConflictException`이 터지는데 이때 `Http 응답 코드는 409`번이 나가겠구나〉 하는 기대(예측) 말이다. 이러한 ‘기대’는, 논리적으로 서비스 영역과 프레젠테이션 영역의 강결합을 부른다고 생각한다.

그리고 예외에 대한 처리는 `@ControllerAdvice`와 같은 어노테이션을 통해 하나의 클래스에서 관리하는 것이 유지보수 측면에서 좋다. 이러한 스프링의 설계에 거슬러서 서비스 코드에서 Http 응답 코드를 정하면 안 된다는 게 내 생각이다.