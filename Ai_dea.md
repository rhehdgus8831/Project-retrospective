
#  Ai-dea 팀 프로젝트 회고록

## 📌 프로젝트 개요
- **프로젝트명**: [Vibe_Fiction](https://github.com/Vibe-Fiction/AI-dea)
- **기간**: 2025년 08월 04일 ~ 2025년 08월 21일
- **주제**: Relai는 AI와 사용자가 함께 소설을 창작하고, 릴레이 방식으로 확장해 나가는 협업형 플랫폼입니다.
  사용자는 인물 관계와 세계관을 직접 설계하거나, AI의 제안을 바탕으로 새로운 흐름을 만들어갈 수 있습니다.

---

## 🛠 사용 기술 스택

## 기술 스택

| 구분            | 기술 및 버전                                                                           |
| ------------- |-----------------------------------------------------------------------------------|
| **언어**        | Java 17                                                                           |
| **프레임워크**     | Spring Boot **3.5.4**                                                             |
| **프론트엔드**     | HTML5, CSS3, JavaScript (ES6+), Thymeleaf                                         |
| **보안/인증**     | Spring Security, JWT (JJWT **0.12.3**)                                            |
| **AI 연동**     | Google Gemini API                                                                 |
| **데이터베이스**    | MariaDB (JDBC 드라이버), Spring Data JPA, Hibernate ORM, QueryDSL **5.0.0 (Jakarta)** |
| **빌드/의존성 관리** | Gradle (Spring Dependency Management Plugin **1.1.7**)                            |
| **테스트 프레임워크** | JUnit 5, Mockito, H2 Database                                                    |
| **개발 편의 도구**  | Lombok, Spring Boot DevTools                                                      |
| **로깅**        | Spring Boot Logging, Hibernate SQL Debug/Trace                                    |
| **파일 업로드**    | 로컬 파일 업로드 (`~/aidea/uploads/`)                                             |
| **협업 도구**     | Git, GitHub, Discord                                                              |
| **개발 환경**     | IntelliJ IDEA, Windows 11                                                         |
| **테스트 환경**    | Chrome, Postman                                                                   |

          
  

---


| 역할 | 이름 | 담당 |
|------|------|------|
| 팀원 | **고동현** |사용자 인증 시스템 설계 및 백엔드 개발<br><br>- JWT 기반 인증/인가 시스템 구축<br> - Spring Security 설정 및 최적화<br>      • 인증/비인증 경로 분리를 통한 보안 정책 수립 및 적용<br>      • 회원가입 및 로그인 핵심 API 개발<br>      • 실시간 아이디/이메일/닉네임 중복 체크 API 개발을 통한 UX 개선<br>      • 로그인 응답에 사용자 권한(Role) 정보를 포함하도록 DTO 및 서비스 로직 개선<br>     - 백엔드 API와 프론트엔드 UI를 Fetch API를 사용하여 비동기 통신으로 연동 |
| 팀장 | 송민재 |  |
| 팀원 | 백승현 |  |
| 팀원 | 왕택준 |  |

---

# Vibe Fiction 개발 트러블슈팅 및 회고록

## 🧯 트러블슈팅

-----

### **문제 1: Spring Security로 인한 `403 Forbidden` 에러**

#### **문제 상황**

1.  `@PostMapping`으로 API 엔드포인트(`/api/auth/signup`, `/api/auth/login`)를 정상적으로 구현했습니다.
2.  Postman으로 해당 API를 호출했을 때, 컨트롤러 로직이 실행되기도 전에 `403 Forbidden` 에러가 발생했습니다.
3.  서버 로그에는 특별한 예외가 기록되지 않아 초기 원인 파악이 어려웠습니다.

#### **원인 분석**

  - Spring Security 의존성을 추가하면, 기본적으로 **모든 요청에 대해 인증을 요구**하도록 자동 설정됩니다. `permitAll()`로 명시적으로 허용하지 않은 모든 경로는 인증되지 않은 접근으로 간주되어 `403` 에러를 반환합니다.

#### **해결 방법**

  - `SecurityConfig` 설정 파일 내 `SecurityFilterChain` 빈에서, `authorizeHttpRequests`를 사용하여 인증 없이 접근해야 하는 API 경로들을 명시적으로 허용했습니다.

  - **수정 전 코드 (`SecurityConfig.java`)**

    ```java
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // ...
            .authorizeHttpRequests(authorize -> authorize
                .anyRequest().authenticated() // 모든 요청에 인증을 요구하는 기본 설정만 있었음
            );
        return http.build();
    }
    ```

  - **수정 후 코드 (`SecurityConfig.java`)**

    ```java
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // ...
            .authorizeHttpRequests(authorize -> authorize
                // 회원가입/로그인 등 인증 없이 호출해야 하는 API 경로들을 명시적으로 허용
                .requestMatchers("/api/auth/**", "/api/novels/**").permitAll() 
                .anyRequest().authenticated() // 그 외의 요청만 인증을 요구
            );
        return http.build();
    }
    ```

-----

### **문제 2: `BusinessException` 발생 시 `GlobalExceptionHandler` 비정상 동작**

#### **문제 상황**

1.  서비스 로직에서 중복된 아이디로 회원가입 시 `BusinessException`을 정상적으로 `throw`했습니다.
2.  서버 콘솔 로그에는 해당 `BusinessException`이 발생했다는 기록이 남았습니다.
3.  하지만 Postman에는 `@ExceptionHandler(BusinessException.class)`에서 정의한 커스텀 에러 응답(`ErrorResponse`)이 아닌, **스프링 부트의 기본 500 서버 에러**가 반환되었습니다.

#### **원인 분석**

  - 서비스 로직에서 `throw new BusinessException("에러 메시지")`와 같이 **문자열만으로 예외를 생성**했습니다. 이 경우 `BusinessException` 객체 내의 `ErrorCode` 필드는 `null` 상태가 됩니다.
  - `GlobalExceptionHandler`의 핸들러 메서드 내부에서 `exception.getErrorCode().getStatus()`와 같이 `null`인 `ErrorCode` 객체의 메서드를 호출하려다 **`NullPointerException`이 발생**했습니다.
  - **예외 처리기 자체에서 새로운 예외가 발생**했기 때문에, 정상적인 커스텀 응답을 생성하지 못하고 서버 기본 에러로 처리되었습니다.

#### **해결 방법**

  - 서비스 로직에서 `BusinessException`을 생성할 때, 문자열이 아닌 **`ErrorCode` Enum을 사용**하여 생성하도록 코드를 수정했습니다.

  - **수정 전 코드 (`SignUpServiceKO.java`)**

    ```java
    // ...
    if (usersRepository.existsByLoginId(requestDto.getLoginId())) {
        throw new BusinessException("이미 사용 중인 아이디입니다.");
    }
    // ...
    ```

  - **수정 후 코드 (`SignUpServiceKO.java`)**

    ```java
    // ...
    if (usersRepository.existsByLoginId(requestDto.getLoginId())) {
        throw new BusinessException(ErrorCode.DUPLICATE_USERNAME);
    }
    // ...
    ```

-----

### **문제 3: JSON 파싱 오류와 `@Valid` 유효성 검증 오류의 응답 형식 불일치**

#### **문제 상황**

1.  `@Valid`를 통한 유효성 검증 실패(예: 닉네임 길이 위반) 시에는 `validationErrors` 목록을 포함한 일관된 JSON 형식으로 에러가 반환되었습니다.
2.  날짜 필드에 잘못된 형식의 문자열(예: `"2000-1-1"`)을 보내 JSON 파싱 자체가 실패하는 경우, **스프링 부트 기본 에러 페이지만 노출**되고 응답 형식이 일관되지 않았습니다.

#### **원인 분석**

  - JSON 파싱 오류(`HttpMessageNotReadableException`)는 **DTO 객체가 생성되기 전, 요청 본문을 읽는 단계**에서 발생합니다.
  - `@Valid` 유효성 검증 오류(`MethodArgumentNotValidException`)는 **DTO 객체가 성공적으로 생성된 후, 그 내용물을 검사하는 단계**에서 발생합니다.
  - 두 예외의 발생 시점과 타입이 근본적으로 달라, 기존의 `MethodArgumentNotValidException` 핸들러가 파싱 오류를 처리할 수 없었습니다.

#### **해결 방법**

  - `GlobalExceptionHandler`에 `HttpMessageNotReadableException`을 처리하는 **별도의 핸들러 메서드를 추가**했습니다.

  - 이 핸들러 내부에서, 단일 파싱 에러 정보를 예외의 원인(`cause`)에서 직접 추출하여 **`@Valid` 에러와 동일한 `validationErrors` 리스트 형식으로 수동으로 가공**한 뒤 반환하도록 구현하여 응답 형식의 일관성을 확보했습니다.

  - **수정 전 코드 (`GlobalExceptionHandler.java`)**

    ```java
    // HttpMessageNotReadableException을 처리하는 핸들러가 존재하지 않았음
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(...) {
        // @Valid 관련 예외만 처리
    }
    ```

  - **수정 후 코드 (`GlobalExceptionHandler.java`)**

    ```java
    // 기존 @Valid 핸들러는 유지
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(...) {
        // ...
    }

    // JSON 파싱 에러를 처리하는 핸들러 추가
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ErrorResponse> handleHttpMessageNotReadableException(...) {
        // 단일 에러를 추출하여 validationErrors 리스트 형식으로 가공하는 로직
        // ...
    }
    ```

-----

## 📚 배운 점 / 느낀 점

  - **Spring Security의 동작 원리**: 의존성 추가만으로도 모든 요청이 기본적으로 차단되며, `SecurityConfig`를 통해 명시적으로 접근 규칙을 정의해야 한다는 것을 배웠습니다. 보안은 '기본적으로 차단하고, 예외적으로 허용'하는 원칙을 따른다는 것을 체감했습니다.

  - **예외 처리 로직의 견고성**: 예외를 단순히 `throw`하는 것에서 그치지 않고, 예외를 처리하는 `@ExceptionHandler` 내부에서 또 다른 예외(e.g., `NullPointerException`)가 발생할 수 있다는 점을 깨달았습니다. 예외를 던지는 쪽과 받는 쪽의 계약(e.g., `ErrorCode`를 넘겨주기)을 명확히 하는 것이 얼마나 중요한지 느꼈습니다.

  - **스프링의 요청 처리 생명주기**: 사용자의 요청이 컨트롤러에 도달하기까지 여러 단계를 거친다는 것을 이해하게 되었습니다. 특히, 요청 본문을 객체로 변환하는 '파싱/바인딩' 단계와 객체의 내용물을 검증하는 `@Valid` 단계가 명확히 분리되어 있다는 점을 학습했습니다. 이 생명주기를 이해하는 것이 정확한 예외 처리에 필수적임을 느꼈습니다.

-----

## 🚀 다음 프로젝트에 반영할 점

 - **인증/인가 로직의 선제적 구현**: API 엔드포인트를 구현하기 전에, `SecurityConfig`에 전체적인 인증/인가 규칙의 골격을 먼저 잡겠습니다. 어떤 경로는 인증 없이 허용하고, 어떤 경로는 특정 권한이 필요한지 미리 정의함으로써, 개발 과정에서 발생할 수 있는 `401/403` 에러를 최소화하고 보안 설정을 누락하는 실수를 방지할 것입니다.

 - **JPA/DB 상호작용에 대한 깊은 이해**: 단순히 JPA 메서드를 사용하는 것을 넘어, `@Entity`의 생명주기나 외래 키 제약 조건과 같은 DB 레벨의 규칙이 애플리케이션에 미치는 영향을 더 깊이 고려하겠습니다. 특히 테스트 데이터 초기화나 데이터 삭제 로직 구현 시, 연관 관계에 있는 테이블까지 고려하여 데이터 무결성을 해치지 않는 안전한 방법을 우선적으로 적용할 것입니다.

  
 
