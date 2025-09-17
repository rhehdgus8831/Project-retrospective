## 🧯 트러블슈팅

-----

### **문제: 모임 목록 조회 시 N+1 쿼리 문제로 인한 성능 저하**

#### **문제 상황**

1.  메인 페이지에서 전체 모임 목록을 조회하는 API(`GET /api/meetings/search`)를 구현하고 테스트하는 과정에서, 모임(Meeting)의 개수(N)만큼 추가적인 쿼리가 발생하는 **N+1 문제**를 발견했습니다.
2.  모임이 8개일 경우, 총 9개의 쿼리(모임 목록 조회 1번 + 각 모임의 마트 정보 조회 8번)가 서버 로그에 기록되었습니다.
3.  이로 인해 데이터베이스에 불필요한 부하가 발생하고, API 응답 시간이 지연되는 성능 문제가 확인되었습니다.

#### **원인 분석**

  - **데이터 흐름 및 원인 추적**:

    1.  `FindMeetingService`에서 `meetingsRepository.findAll()`을 호출하여 `Meeting` 엔티티 목록을 조회합니다.
    2.  `Meeting` 엔티티와 `Mart` 엔티티는 **다대일(@ManyToOne) 관계**로 매핑되어 있으며, 기본 **지연 로딩(Lazy Loading)** 전략으로 설정되어 있습니다.
    3.  `MeetingSimpleResponse` DTO로 변환하는 과정에서, 각 `Meeting` 엔티티의 `getMart().getMartName()` 메서드를 호출합니다.
    4.  **핵심 원인**: 지연 로딩으로 인해, `getMart()`가 호출되는 시점에 `Mart` 엔티티가 아직 로드되지 않았으므로, JPA는 각 `Meeting` 객체에 대해 `Mart` 정보를 가져오기 위한 **별도의 SELECT 쿼리를 실행**합니다. 이로 인해 모임 목록 조회 쿼리 1번에 더해, N개의 모임에 대한 N개의 마트 정보 조회 쿼리가 추가로 발생했습니다.

    <!-- end list -->

    ```
    -- 쿼리 로그 예시
    Hibernate: select m1_0.meeting_id, ... from meetings m1_0 -- (1) 모임 전체 조회 쿼리 (1번)
    Hibernate: select m1_0.mart_id, ... from marts m1_0 where m1_0.mart_id=? -- (2) 첫 번째 모임의 마트 정보 조회
    Hibernate: select m1_0.mart_id, ... from marts m1_0 where m1_0.mart_id=? -- (3) 두 번째 모임의 마트 정보 조회
    ... (N번 반복) ...
    ```

#### **해결 방법**

  - `MeetingsRepository`에 JPQL과 \*\*`JOIN FETCH`\*\*를 사용하여, 모임 목록을 조회할 때 연관된 `Mart` 엔티티 정보까지 **한 번의 쿼리로 함께** 불러오도록 수정했습니다. `JOIN FETCH`는 지연 로딩으로 설정된 연관 엔티티를 즉시 함께 조회하여 N+1 문제를 근본적으로 해결합니다.

  - **수정 전 코드 (`MeetingsRepository.java`)**

    ```java
    // N+1 문제 발생 코드
    // findAll()은 연관된 Mart 엔티티를 함께 조회하지 않음
    List<Meeting> findAll();
    ```

  - **수정 후 코드 (`MeetingsRepository.java`)**

    ```java
    /**
     * 모든 모임을 생성 시간(createdAt) 내림차순으로 정렬하여 조회합니다. (최신순)
     * "m.mart"를 함께 조회(JOIN FETCH)하여 N+1 문제를 방지하고
     * Mart 정보가 누락되지 않도록 보장합니다.
     */
    @Query("SELECT m FROM Meeting m JOIN FETCH m.mart ORDER BY m.createdAt DESC")
    List<Meeting> findAllWithMartByOrderByCreatedAtDesc();
    ```

  - **서비스 레이어 수정 (`FindMeetingService.java`)**

      - 기존의 `meetingsRepository.findAll()` 호출 부분을 새로 추가한 `findAllWithMartByOrderByCreatedAtDesc()` 메서드 호출로 변경했습니다.

    <!-- end list -->

    ```java
    // FindMeetingService.java
    public List<MeetingSimpleResponse> searchMeetings(String keyword) {
        List<Meeting> meetings;

        if (keyword != null && !keyword.isBlank()) {
            meetings = meetingsRepository.findByKeywordAndRecruiting(keyword);
        } else {
            // ✅ N+1 문제가 해결된 메서드 호출
            meetings = meetingsRepository.findAllWithMartByOrderByCreatedAtDesc();
        }

        return meetings.stream()
                .map(MeetingSimpleResponse::new)
                .collect(Collectors.toList());
    }
    ```

-----

### **결과 및 교훈**

> 이번 트러블슈팅을 통해 JPA의 **지연 로딩**과 **N+1 문제**에 대해 깊이 이해할 수 있었습니다. 처음에는 기능이 정상적으로 동작하는 것처럼 보였지만, 쿼리 로그를 꼼꼼히 확인하면서 숨어있는 성능 병목 지점을 발견했습니다. `JOIN FETCH`를 적용하여 단 한 번의 쿼리로 모든 데이터를 가져오도록 최적화하는 과정은 매우 의미 있는 경험이었습니다.
>
> 특히 \*\*즉시 로딩(Eager Loading)\*\*은 당장의 N+1 문제를 해결할 수는 있지만, 연관된 모든 데이터를 항상 함께 불러오기 때문에 불필요한 데이터 로딩으로 이어질 수 있다는 점을 배웠습니다. 반면, \*\*`JOIN FETCH`\*\*는 필요한 시점에, 필요한 연관 데이터만 명시적으로 함께 조회할 수 있어 성능과 유연성 측면에서 더 나은 해결책이라는 것을 깨달았습니다. 앞으로는 엔티티를 설계하고 쿼리를 작성할 때 데이터 접근 패턴을 신중하게 고려하여, 숨어있는 성능 문제까지 예측하고 방어하는 개발자가 되어야겠다고 다짐했습니다.
