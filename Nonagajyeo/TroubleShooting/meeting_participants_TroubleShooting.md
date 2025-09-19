## 🧯 트러블슈팅

-----

### **문제: 참여 승인/거절 기능의 다단계 복합 오류 (UI 렌더링, API 통신, DB 정합성)**

#### **문제 상황**

1.  **초기 증상 (UI)**: 모임 상세 페이지의 '참여자 목록' 탭에서 호스트 본인이 확정자 명단에 보이지 않았고, 신청 대기자의 신뢰도 점수가 렌더링되지 않았습니다.
2.  **기능 오류 (API)**: 호스트가 '승인' 또는 '거절' 버튼을 클릭했을 때, 프론트엔드 콘솔에는 `404 Not Found` 또는 `405 Method Not Allowed` 에러가 발생하며 서버와 통신이 실패했습니다.
3.  **데이터 불일치 (DB)**: API 통신 오류를 해결한 후, 승인/거절 요청은 성공하는 것처럼 보였으나 페이지를 새로고침(F5)하면 다시 원래의 '대기중' 상태로 되돌아가는 문제가 발생했습니다.

#### **원인 분석**

- **데이터 흐름 추적**:
    1.  **`userId` vs `participantId` 혼동**: 백엔드 API는 '참여 기록'의 고유 ID인 **`participantId`(Long 타입)** 를 파라미터로 요구했으나, 프론트엔드에서는 '사용자'의 고유 ID인 **`userId`(Long 타입)** 를 전송하고 있었습니다. 이로 인해 백엔드가 잘못된 대상을 처리하거나 대상을 찾지 못하는 문제가 발생했습니다.
    2.  **DTO 필드 누락**: 근본적으로, 백엔드가 프론트엔드로 참여자 목록 정보를 보낼 때 사용하는 `MeetingParticipantResponse` DTO에 `participantId` 필드가 누락되어, 프론트엔드에서 올바른 ID를 사용할 수 없었습니다.
    3.  **잘못된 `@Transactional` Import (핵심 원인)**: 백엔드 `ManageMeetingService`에서 `jakarta.transaction.Transactional` 어노테이션을 사용하고 있었습니다. Spring Boot 환경에서는 **`org.springframework.transaction.annotation.Transactional`** 을 사용해야 데이터베이스 변경 사항이 정상적으로 **커밋(commit)** 됩니다. 잘못된 어노테이션 사용으로 인해, 서비스 로직은 성공적으로 실행된 것처럼 보였으나 실제로는 모든 DB 변경이 조용히 **롤백(rollback)** 되고 있었습니다. 이것이 새로고침 시 상태가 원상 복구되는 현상의 직접적인 원인이었습니다.
    4.  **React `key` Prop 누락**: `hostInfo` DTO에는 `userId`가 없었기 때문에, 프론트엔드에서 수동으로 생성한 호스트 객체의 `key` prop이 `undefined`가 되었습니다. 이로 인해 React는 리스트의 변경사항을 정확히 감지하지 못하고, **낙관적 업데이트** 후에도 화면을 제대로 리렌더링하지 못하는 2차 문제를 유발했습니다.

#### **해결 방법**

-   백엔드와 프론트엔드에 걸쳐 **데이터의 흐름을 일치시키고, 올바른 기술 스택을 사용하도록 코드를 전면 수정**했습니다.

-   **수정 전 코드 (`ManageMeetingService.java`)**
    ```java
    import jakarta.transaction.Transactional; // 🚨 잘못된 import 사용으로 DB 변경이 롤백됨

    @Service
    @Transactional
    @RequiredArgsConstructor
    public class ManageMeetingService {
        // ...
        public void approveParticipant(Long meetingId, Long participantId, String hostProviderId) {
            // ... 내부 로직은 실행되나, 메서드 종료 후 모든 변경이 취소됨
        }
    }
    ```

-   **수정 후 코드 (백엔드)**
    -   **`MeetingParticipantResponse.java` 수정**:
        -   `participantId` 필드를 추가하여, 프론트엔드가 '참여 기록'의 고유 ID를 받을 수 있도록 수정했습니다.
        -   신뢰도 점수 필드명을 `trusterScore`에서 `trustScore`로 변경하여 다른 DTO와의 일관성을 확보했습니다.

    -   **`ManageMeetingService.java` 수정**:
        -   `import` 문을 **`org.springframework.transaction.annotation.Transactional`**로 변경하여 데이터베이스 트랜잭션이 정상적으로 커밋되도록 수정했습니다.
        -   '참여 거절' 로직을 DB에서 `DELETE`하는 대신, `applicationStatus`를 `REJECTED`로 `UPDATE`하는 방식으로 변경하여 데이터 기록을 유지하도록 개선했습니다.

    -   **`MeetingController.java` 수정**:
        -   `@PostMapping`을 사용하여 `.../participants/{participantId}/reject` API 엔드포인트를 추가했습니다.

-   **수정 후 코드 (프론트엔드)**
    -   **`participantApi.js` 수정**:
        -   `rejectParticipant` 함수의 API 호출 방식을 백엔드와 일치하는 `POST`로 변경했습니다.
    -   **`MeetingDetailPage.jsx` 수정**:
        -   `handleApprove`, `handleReject` 함수가 `userId` 대신 `participantId`를 파라미터로 사용하도록 수정했습니다.
        -   '낙관적 업데이트' 로직을 구현하여, API 성공 응답 시 서버를 기다리지 않고 즉시 UI를 변경함으로써 사용자 경험을 향상시켰습니다.
        -   호스트 객체 생성 시, `key` prop으로 사용할 `userId` 필드에 고유값인 `nickname`을 할당하여 React의 리스트 렌더링 오류를 해결했습니다.

-----

### **추후 개선에 대한 논의**

> 이번 트러블슈팅은 프론트엔드의 '낙관적 업데이트'와 백엔드의 '트랜잭션 관리' 사이의 상호작용이 얼마나 중요한지를 보여주는 대표적인 사례입니다.
>
> 프론트엔드에서 API 호출이 성공했다는 응답을 받는 것과, 백엔드에서 데이터베이스 트랜잭션이 최종적으로 **커밋(commit)되는 것**은 별개의 과정임을 명확히 인지해야 합니다. 만약 데이터 정합성이 매우 중요한 기능이라면, 낙관적 업데이트 후에도 일정 시간차를 두고 서버로부터 다시 데이터를 조회하여 상태를 한번 더 동기화하는 '재검증(re-validation)' 로직을 추가하는 것을 고려해볼 수 있습니다.
>
> 하지만 현재의 '참여 승인/거절' 기능은 낙관적 업데이트만으로도 충분히 안정적이며, 사용자에게 즉각적인 피드백을 제공하는 현재의 방식이 더 나은 사용자 경험을 제공한다고 판단됩니다. 또한, **프론트엔드와 백엔드 간의 DTO 명세(데이터 계약)를 명확히 정의하고 공유하는 문화**가 초기 개발 단계의 오류를 크게 줄일 수 있다는 교훈을 얻었습니다.
