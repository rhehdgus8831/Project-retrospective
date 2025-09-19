## 🧯 트러블슈팅

-----

### **문제: JWT 인증 시 `providerId`와 `userId` 타입 불일치로 인한 인증 실패**

#### **문제 상황**

1.  백엔드에 JWT와 Spring Security를 이용한 인증 시스템을 구현했습니다.
2.  Postman을 사용해 유효한 `ACCESS_TOKEN`을 Bearer 토큰으로 `Authorization` 헤더에 담아 API(`POST /api/meetings`)를 호출했습니다.
3.  예상 결과는 `201 CREATED`였지만, 실제로는 `401 Unauthorized` 또는 커스텀 에러 응답(`AUTHENTICATION_NOT_FOUND`)이 반환되었습니다.
4.  백엔드 서버 로그에는 `BusinessException: 인증 정보를 찾을 수 없습니다.` 라는 경고가 기록되어, 토큰 자체는 유효하지만 서버가 사용자를 식별하지 못하는 것으로 추정되었습니다.

#### **원인 분석**

  - **데이터 흐름 추적**:
    1.  `JwtAuthenticationFilter`는 토큰을 성공적으로 검증하고, 토큰의 `subject`에 저장된 \*\*`providerId`(String 타입, 예: "123456789")\*\*를 `SecurityContext`의 `Principal`로 설정합니다.
    2.  `MeetingController`의 API 메서드는 `@AuthenticationPrincipal Long currentUserId` 어노테이션을 통해 `Principal` 값을 주입받으려고 시도합니다.
    3.  **핵심 원인**: Spring Security는 `Principal`에 저장된 \*\*`String` 타입의 `providerId`\*\*를 \*\*`Long` 타입의 `currentUserId`\*\*로 자동 형변환할 수 없습니다. 결과적으로 `currentUserId` 파라미터에는 `null`이 주입됩니다.
    4.  이 `null` 값이 서비스 레이어로 전달되자, 서비스는 사용자가 없는 것으로 간주하고 `AUTHENTICATION_NOT_FOUND` 예외를 발생시켰습니다.

#### **해결 방법**

  - 인증 과정 전반에 걸쳐 사용자 식별자의 데이터 타입을 **`String` 타입인 `providerId`로 통일**했습니다. 이를 통해 컨트롤러가 `Principal` 값을 올바르게 주입받고, 서비스가 해당 값으로 사용자를 정확히 조회할 수 있도록 수정했습니다.

  - **수정 전 코드 (`MeetingController.java`)**

    ```java
    @PostMapping
    public ResponseEntity<ApiResponse<MeetingCreateResponse>> createMeeting(
            @Valid @RequestBody MeetingCreateRequest request,
            @AuthenticationPrincipal Long currentUserId) { // 🚨 String -> Long 형변환 실패로 null 주입
        
        Meeting newMeeting = createMeetingService.createMeeting(request, currentUserId);
        // ...
    }
    ```

  - **수정 후 코드 (`MeetingController.java`)**

    ```java
    @PostMapping
    public ResponseEntity<ApiResponse<MeetingCreateResponse>> createMeeting(
            @Valid @RequestBody MeetingCreateRequest request,
            @AuthenticationPrincipal String providerId) { // ✅ String 타입으로 올바르게 주입받음
        
        Meeting newMeeting = createMeetingService.createMeeting(request, providerId);
        // ...
    }
    ```

  - **서비스 레이어 수정 (`CreateMeetingService.java`)**

      - 컨트롤러의 변경에 맞춰, 서비스 메서드도 `String providerId`를 파라미터로 받도록 수정했습니다.
      - 사용자 조회 로직을 `usersRepository.findById(userId)`에서 \*\*`usersRepository.findByProviderId(providerId)`\*\*로 변경하여, `providerId`를 통해 `User` 엔티티를 조회하도록 수정했습니다.

    <!-- end list -->

    ```java
    // CreateMeetingService.java
    public Meeting createMeeting(MeetingCreateRequest request, String providerId) {
        // ...
        User hostUser = usersRepository.findByProviderId(providerId) // ✅ providerId로 사용자 조회
                .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));
        // ...
    }
    ```

-----

### **추후 개선에 대한 논의**

> 현재 `providerId`를 직접 `Principal`로 사용하는 방식은 단일 소셜 로그인(카카오) 환경에서는 직관적이고 효율적입니다.
>
> 하지만 추후 구글, 네이버 등 **다수의 SSO(Single Sign-On) 공급자를 도입할 경우**, 각 공급자마다 `providerId`의 형식과 정책이 다를 수 있습니다. 이 경우, 컨트롤러와 서비스가 특정 공급자의 `providerId`에 종속되는 것은 확장성에 불리할 수 있습니다.
>
> **장기적인 관점에서는** `JwtAuthenticationFilter` 단에서 `providerId`로 `User` 엔티티를 조회한 뒤, 우리 시스템의 고유 식별자인 **`userId(Long)`를 `Principal`로 설정**하는 방식을 고려해볼 수 있습니다. 이렇게 하면 서비스 로직 전체가 공급자에 관계없이 일관된 `userId`를 기준으로 동작하게 되어 더욱 유연하고 확장성 있는 구조가 될 것입니다.
>
> 현 단계에서는 `providerId`를 사용하는 현재의 해결책이 최선이며, 위 내용은 추후 인증 시스템 확장 시 고려할 아키텍처 개선 사항으로 기록해 둡니다.
