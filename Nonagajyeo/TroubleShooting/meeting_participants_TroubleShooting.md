

#  모임 참여 신청 기능 트러블슈팅

**작성자**: 고동현  
**작성일**: 2025-09-19  
**기능**: 모임 상세 페이지 내 참여자 목록 관리 (호스트의 참여 승인/거절)

## 1. 개요

모임 참여 신청 관리 기능 개발 중, 여러 단계에 걸쳐 복합적인 문제가 발생했습니다. 이 문서는 각 문제의 증상, 원인 분석 과정, 그리고 최종 해결 방법을 기록하여 유사한 문제 발생 시 빠르고 효과적인 해결을 돕기 위해 작성되었습니다. 문제는 크게 **UI 렌더링**, **API 통신**, **데이터 정합성** 세 가지 영역에서 발생했습니다.

---

## 2. 주요 문제 및 해결 과정

### 문제 1: 초기 렌더링 오류

#### 증상 A: 호스트가 '확정된 참여자' 목록에 표시되지 않음
- **현상**: 모임 상세 페이지에 진입했을 때, 호스트 본인이 확정 참여자 목록에 보이지 않고 비어있는 문제.
- **원인 분석**:
    1.  백엔드 API(`GET /api/meetings/{id}`)는 '모임 정보'와 '참여자 목록'을 별개의 데이터 구조로 전달했습니다.
    2.  '모임 정보' 객체 안에는 `hostInfo`가, '참여자 목록' 객체 안에는 `participants.approved` 배열이 있었습니다.
    3.  프론트엔드(`ParticipantsTab.jsx`)는 `participants.approved` 배열만 렌더링하고 있어, `hostInfo`가 누락되었습니다.
- **해결**:
    -   `MeetingDetailPage.jsx`의 렌더링 직전 단계에서, `hostInfo` 객체를 참여자 객체와 동일한 형태로 가공하여 `participants.approved` 배열의 맨 앞에 수동으로 추가했습니다. 이를 통해 렌더링 컴포넌트는 데이터 구조를 신경 쓸 필요 없이 받은 목록만 그대로 그리도록 역할을 유지했습니다.

#### 증상 B: 신뢰도 점수가 호스트에게는 보이지만, 다른 참여자에게는 보이지 않음
- **현상**: '신청 대기자' 목록에 있는 사용자의 신뢰도 점수 값이 렌더링되지 않는 문제.
- **원인 분석**:
    1.  백엔드의 두 DTO에서 사용하는 필드 이름이 서로 달랐습니다.
        -   `MeetingDetailResponse.java`의 `HostInfo` 클래스: `trustScore`
        -   `MeetingParticipantResponse.java`: `trusterScore`
    2.  프론트엔드 `ParticipantsTab.jsx`에서는 `participant.TrusterScore` (대문자 T)를 사용하고 있어 데이터 바인딩에 실패했습니다.
- **해결**:
    -   **백엔드**: `MeetingParticipantResponse.java`의 필드명을 `trusterScore`에서 `trustScore`로 변경하여 모든 DTO에서 필드 이름을 통일했습니다.
    -   **프론트엔드**: `ParticipantsTab.jsx`와 `MeetingDetailPage.jsx`에서 신뢰도 점수를 참조하는 모든 코드를 `participant.trustScore` (소문자 t)로 통일하여 문제를 해결했습니다.

---

### 문제 2: 참여 승인/거절 기능 API 통신 오류

#### 증상: 승인/거절 버튼 클릭 시 `40x` 에러 발생
- **현상**: 참여 승인/거절 버튼 클릭 시, 브라우저 콘솔에 `404 Not Found`, `405 Method Not Allowed` 등 API 통신 실패 에러가 발생.
- **원인 분석**:
    1.  **`userId` vs `participantId` 혼동**: 백엔드 API는 '참여 기록'의 고유 ID인 `participantId`를 요구했지만, 프론트엔드에서는 '사용자'의 고유 ID인 `userId`를 보내고 있었습니다. 이로 인해 백엔드가 잘못된 대상을 처리하거나 대상을 찾지 못했습니다.
    2.  **DTO 필드 누락**: 근본적으로, 백엔드가 참여자 목록을 보낼 때 사용하는 `MeetingParticipantResponse` DTO에 `participantId` 필드가 누락되어 프론트엔드에서 사용할 수 없었습니다.
    3.  **API 엔드포인트 부재**: '거절' 기능의 경우, 프론트엔드는 `DELETE /.../reject`를 호출했지만 백엔드 `MeetingController`에는 해당 요청을 처리할 API 자체가 정의되지 않았습니다.
- **해결 (Full-Stack)**:
    -   **백엔드**:
        1.  `MeetingParticipantResponse.java` DTO에 `participantId` 필드를 추가하여, 참여 기록 ID를 프론트엔드로 전달하도록 수정했습니다.
        2.  `MeetingController.java`에 `@PostMapping`을 사용하여 `/reject` 엔드포인트를 새로 추가했습니다.
    -   **프론트엔드**:
        1.  `handleApprove`, `handleReject` 함수에서 `userId` 대신, DTO 수정을 통해 새로 받게 된 `participantId`를 사용하여 API를 호출하도록 수정했습니다.
        2.  `participantApi.js`에서 거절 API 호출 방식을 백엔드 컨트롤러와 일치하는 `POST`로 변경했습니다.

---

### 문제 3: 데이터 정합성 및 상태 관리 오류

#### 증상 A: 승인/거절 후 UI가 업데이트되지 않고 React `key` 경고 발생
- **현상**: API 호출은 성공하는 것처럼 보이나, 대기자가 사라지거나 확정자 목록으로 이동하지 않는 문제. 브라우저 콘솔에 `Each child in a list should have a unique "key" prop` 경고가 지속적으로 발생.
- **원인 분석**:
    1.  **`key` 값 누락**: `MeetingDetailPage.jsx`에서 `hostInfo`를 기반으로 수동 생성한 `hostAsParticipant` 객체에 `key`로 사용할 `userId` 필드가 누락되었습니다. (`hostInfo` DTO에 `id` 관련 필드가 없었음)
    2.  **React 렌더링 오류**: 리스트의 첫 항목인 호스트의 `key`가 `undefined`가 되자, React는 리스트의 변경사항을 정확히 감지하고 DOM을 업데이트하는 데 실패했습니다. 이는 `key` prop이 React의 리스트 렌더링 알고리즘에 얼마나 중요한지를 보여줍니다.
- **해결**:
    -   `hostInfo` 객체에는 고유 ID가 없었지만, `nickname`은 고유하다는 점을 이용하여 `hostAsParticipant` 객체를 생성할 때 `userId: meeting.hostInfo.nickname` 코드를 추가했습니다. 이를 통해 호스트 객체에 안정적인 `key` 값을 부여하여 React의 렌더링 오류와 `key` 경고를 동시에 해결했습니다.

#### 증상 B: 새로고침(F5) 시 승인/거절 상태가 원상 복구됨
- **현상**: '낙관적 업데이트(Optimistic Update)'를 통해 프론트엔드 UI는 즉시 변경되지만, 페이지를 새로고침하면 다시 승인 전 상태로 되돌아가는 최종 문제.
- **원인 분석**:
    1.  **트랜잭션 롤백**: 백엔드 `ManageMeetingService`의 `approveParticipant` 메서드가 프론트엔드에는 성공 응답을 보냈으나, 메서드 종료 후 데이터베이스에 변경사항을 저장(commit)하는 과정에서 숨겨진 에러가 발생하여 모든 작업이 취소(rollback)되고 있었습니다.
    2.  **잘못된 `@Transactional` Import**: 가장 핵심적인 원인으로, `jakarta.transaction.Transactional`을 import하여 사용하고 있었습니다. Spring Boot 환경에서는 `org.springframework.transaction.annotation.Transactional`을 사용해야 트랜잭션이 정상적으로 관리됩니다.
- **해결 (Backend)**:
    -   `ManageMeetingService.java`에서 `import` 구문을 `org.springframework.transaction.annotation.Transactional`로 수정했습니다.
    -   부가적으로, 트랜잭션 내에서 오래된 데이터를 조회하던 참여자 수 계산 로직을 수정하여 데이터 정합성을 높였습니다.

---

## 3. 결론 및 교훈

이번 트러블슈팅은 프론트엔드와 백엔드가 복합적으로 얽힌 문제를 단계적으로 해결해 나가는 전형적인 과정이었습니다. 주요 교훈은 다음과 같습니다.

1.  **데이터의 흐름을 의심하라**: UI 문제는 데이터의 문제일 가능성이 높습니다. 프론트엔드가 받는 데이터의 구조(`console.log`)와 백엔드가 보내주는 데이터의 설계(DTO)를 가장 먼저 확인하는 것이 중요합니다.
2.  **`key`는 단순한 경고가 아니다**: React의 `key` prop 경고는 렌더링 성능 저하뿐만 아니라, 상태 업데이트가 실패하는 심각한 버그의 원인이 될 수 있으므로 반드시 해결해야 합니다.
3.  **프론트엔드의 '성공'이 진짜 '성공'은 아니다**: API 호출 성공과 실제 데이터베이스 저장은 별개의 과정일 수 있습니다. '낙관적 업데이트' 후 새로고침을 통해 데이터가 영속적으로 저장되었는지 반드시 검증해야 합니다.
4.  **Full-Stack 관점의 디버깅**: 이번 문제 해결 과정은 프론트엔드의 React 코드부터 백엔드의 Controller, Service, DTO, 그리고 `@Transactional` 어노테이션에 이르기까지 전 영역에 걸쳐있었습니다. 복잡한 문제는 한쪽만 봐서는 해결할 수 없다는 것을 보여주는 좋은 사례입니다.
