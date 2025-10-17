#  SignBell 트러블슈팅 리포트 #3

> ## **"게임 종료 시 N+1 쿼리 문제로 인한 성능 저하"**
> (참가자 수만큼 반복적으로 DB 쿼리가 발생하여 게임 종료가 느려지는 문제)

---

### 1.  문제 상황 (Problem)

#### 시도 및 과정
- 게임 종료 시 모든 참가자의 점수를 계산하고 랭킹을 생성하는 `endGame()` 메서드를 구현했습니다.
- 캐시에서 점수를 조회하고, DB에서 User 정보를 가져와 GameHistory를 저장하는 로직을 작성했습니다.
- 각 참가자의 totalScore를 업데이트하고 랭킹 정보를 생성하여 브로드캐스트했습니다.

#### 긍정적 관찰
- 2~3명의 참가자로 테스트했을 때는 게임 종료가 빠르게 처리되었습니다.
- 점수 계산과 랭킹 생성이 정확하게 동작했습니다.

#### 핵심 문제 발생
- **4명의 참가자로 테스트했을 때**, 게임 종료 처리가 눈에 띄게 느려졌습니다.
- 로그를 확인한 결과, **참가자 수만큼 반복적으로 DB 쿼리가 발생**하는 것을 발견했습니다.
- 이는 전형적인 **N+1 쿼리 문제**였습니다.

<br>

**[관련 코드: 문제가 발생한 로직]**

```java
// QuizService.java - endGame() (개선 전)
@Transactional
public void endGame(Long roomId) {
    Map<Long, Integer> roundScores = roomState.getAllUserScores();
    
    // 점수 높은 순으로 정렬
    List<Map.Entry<Long, Integer>> sortedScores = roundScores.entrySet().stream()
            .sorted(Map.Entry.<Long, Integer>comparingByValue().reversed())
            .collect(Collectors.toList());
    
    List<RankingInfo> rankings = new ArrayList<>();
    List<GameHistory> historiesToSave = new ArrayList<>();
    
    // ❌ N+1 문제 발생 지점
    for (int i = 0; i < sortedScores.size(); i++) {
        Long userId = entry.getKey();
        Integer roundScore = entry.getValue();
        
        // ❌ 매번 DB 쿼리 발생 (N번)
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));
        
        // GameHistory 저장
        historiesToSave.add(GameHistory.builder()
                .participant(user)
                .score(roundScore)
                .build());
        
        // User totalScore 업데이트
        user.updateTotalScore(roundScore);
        
        // 랭킹 정보 생성
        rankings.add(RankingInfo.builder()
                .userId(user.getId())
                .nickname(user.getNickname())
                .profileImageUrl(user.getProfileImageUrl())
                .score(roundScore)
                .build());
    }
    
    // GameHistory 저장
    gameHistoryRepository.saveAll(historiesToSave);
}
```

**[실제 발생한 쿼리 로그]**

```sql
-- 1번 쿼리: GameRoom 조회
SELECT * FROM game_room WHERE game_room_id = 1;

-- 2번 쿼리: User 조회 (참가자 1)
SELECT * FROM user WHERE user_id = 123;

-- 3번 쿼리: User 조회 (참가자 2)
SELECT * FROM user WHERE user_id = 456;

-- 4번 쿼리: User 조회 (참가자 3)
SELECT * FROM user WHERE user_id = 789;

-- 5번 쿼리: User 조회 (참가자 4)
SELECT * FROM user WHERE user_id = 101;

-- 6번 쿼리: GameHistory 저장 (Bulk Insert)
INSERT INTO game_history (game_room_id, user_id, score, round) 
VALUES (1, 123, 250, 1), (1, 456, 200, 1), (1, 789, 150, 1), (1, 101, 100, 1);

-- 7~10번 쿼리: User totalScore 업데이트 (각각)
UPDATE user SET total_score = total_score + 250 WHERE user_id = 123;
UPDATE user SET total_score = total_score + 200 WHERE user_id = 456;
UPDATE user SET total_score = total_score + 150 WHERE user_id = 789;
UPDATE user SET total_score = total_score + 100 WHERE user_id = 101;

-- 총 10번의 쿼리 발생!
-- 참가자가 10명이면? 20번의 쿼리!
```

**[성능 측정 결과]**

```
참가자 2명: 게임 종료 처리 시간 ~50ms
참가자 4명: 게임 종료 처리 시간 ~150ms
참가자 10명 (예상): 게임 종료 처리 시간 ~400ms

→ 참가자 수에 비례하여 처리 시간 증가
```

---

### 2.  원인 분석 (Analysis)

#### 가설 1 (가장 유력한 원인): N+1 쿼리 문제
- **문제의 핵심**: 반복문 안에서 `userRepository.findById(userId)`를 호출하여 참가자 수만큼 쿼리가 발생했습니다.
- 4명의 참가자가 있으면 4번의 SELECT 쿼리가 개별적으로 실행되었습니다.
- 이는 전형적인 N+1 쿼리 문제로, 참가자 수가 증가할수록 성능이 선형적으로 저하됩니다.

#### 가설 2 (부차적인 원인): Dirty Checking으로 인한 개별 UPDATE
- User의 totalScore를 업데이트할 때, JPA의 Dirty Checking으로 인해 각 User마다 개별 UPDATE 쿼리가 발생했습니다.
- 이는 JPA의 정상적인 동작이지만, 대량의 업데이트 시 성능 문제를 야기할 수 있습니다.

#### 결론
현재 구조는 **"한 명씩 처리하는 방식"**으로 설계되어 있었습니다. 하지만 게임 종료는:
1. 모든 참가자의 정보를 한 번에 가져올 수 있음
2. GameHistory도 한 번에 저장할 수 있음
3. User 업데이트는 Dirty Checking으로 자동 처리됨

이러한 특성을 활용하여 **"일괄 처리 방식"**으로 전환하면 성능을 크게 개선할 수 있습니다.

<br>

**[N+1 문제 시각화]**

```
개선 전 (N+1 문제):
┌─────────────────────────────────────┐
│ for (userId in userIds) {           │
│   ┌─────────────────────────────┐   │
│   │ SELECT * FROM user          │   │ ← 쿼리 1
│   │ WHERE user_id = 123;        │   │
│   └─────────────────────────────┘   │
│   ┌─────────────────────────────┐   │
│   │ SELECT * FROM user          │   │ ← 쿼리 2
│   │ WHERE user_id = 456;        │   │
│   └─────────────────────────────┘   │
│   ┌─────────────────────────────┐   │
│   │ SELECT * FROM user          │   │ ← 쿼리 3
│   │ WHERE user_id = 789;        │   │
│   └─────────────────────────────┘   │
│   ┌─────────────────────────────┐   │
│   │ SELECT * FROM user          │   │ ← 쿼리 4
│   │ WHERE user_id = 101;        │   │
│   └─────────────────────────────┘   │
│ }                                   │
└─────────────────────────────────────┘
총 4번의 쿼리 (N번)

개선 후 (Bulk Query):
┌─────────────────────────────────────┐
│ ┌─────────────────────────────────┐ │
│ │ SELECT * FROM user              │ │ ← 쿼리 1번만!
│ │ WHERE user_id IN (123,456,789,101)│ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
총 1번의 쿼리
```

---

### 3.  해결 방안 (Solution)

#### 접근 방식 전환
- **"한 명씩 조회하는 방식"**에서 **"한 번에 모두 조회하는 방식"**으로 전환했습니다.
- JPA의 `findAllById()` 메서드를 활용하여 IN 쿼리로 일괄 조회하기로 결정했습니다.

#### 구체적인 해결책

**1. 참가자 ID 리스트 추출**

```java
@Transactional
public void endGame(Long roomId) {
    Map<Long, Integer> roundScores = roomState.getAllUserScores();
    
    // 점수 높은 순으로 정렬
    List<Map.Entry<Long, Integer>> sortedScores = roundScores.entrySet().stream()
            .sorted(Map.Entry.<Long, Integer>comparingByValue().reversed())
            .collect(Collectors.toList());
    
    // ✅ 1. 참가자들의 ID만 리스트로 추출
    List<Long> userIds = sortedScores.stream()
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
```

**2. 한 번의 쿼리로 모든 User 조회**

```java
    // ✅ 2. ID 리스트를 사용해 "단 한 번의 쿼리"로 모든 User 조회
    Map<Long, User> userMap = userRepository.findAllById(userIds).stream()
            .collect(Collectors.toMap(User::getId, user -> user));
    
    // 이제 userMap에서 O(1)로 User를 가져올 수 있음!
```

**3. Map에서 User 조회 (DB 쿼리 없음)**

```java
    List<RankingInfo> rankings = new ArrayList<>();
    List<GameHistory> historiesToSave = new ArrayList<>();
    
    int rank = 1;
    for (int i = 0; i < sortedScores.size(); i++) {
        Long userId = entry.getKey();
        Integer roundScore = entry.getValue();
        
        // ✅ 3. DB 조회 대신 Map에서 가져옴 (쿼리 발생 X)
        User user = userMap.get(userId);
        
        if (user == null) {
            log.info("게임 중 퇴장한 사용자 랭킹 제외 - userId: {}", userId);
            continue;
        }
        
        // GameHistory 객체 생성 (DB에 바로 저장하지 않음)
        historiesToSave.add(GameHistory.builder()
                .gameRoom(gameRoom)
                .participant(user)
                .score(roundScore)
                .round(completedRound)
                .build());
        
        // ✅ User totalScore 업데이트 (Dirty Checking으로 자동 처리)
        user.updateTotalScore(roundScore);
        
        // 랭킹 정보 생성
        rankings.add(RankingInfo.builder()
                .rank(rank)
                .userId(user.getId())
                .nickname(user.getNickname())
                .profileImageUrl(user.getProfileImageUrl())
                .score(roundScore)
                .build());
        
        rank++;
    }
```

**4. GameHistory 일괄 저장**

```java
    // ✅ 4. 준비된 GameHistory 리스트를 "한 번에" DB에 저장 (Bulk Insert)
    gameHistoryRepository.saveAll(historiesToSave);
    
    log.info("게임 종료 - roomId: {}, participants: {}", roomId, sortedScores.size());
}
```

#### 개선된 쿼리 로그

```sql
-- 1번 쿼리: GameRoom 조회
SELECT * FROM game_room WHERE game_room_id = 1;

-- 2번 쿼리: User 일괄 조회 (IN 쿼리)
SELECT * FROM user WHERE user_id IN (123, 456, 789, 101);

-- 3번 쿼리: GameHistory 일괄 저장 (Bulk Insert)
INSERT INTO game_history (game_room_id, user_id, score, round) 
VALUES (1, 123, 250, 1), (1, 456, 200, 1), (1, 789, 150, 1), (1, 101, 100, 1);

-- 4~7번 쿼리: User totalScore 업데이트 (Dirty Checking)
UPDATE user SET total_score = total_score + 250 WHERE user_id = 123;
UPDATE user SET total_score = total_score + 200 WHERE user_id = 456;
UPDATE user SET total_score = total_score + 150 WHERE user_id = 789;
UPDATE user SET total_score = total_score + 100 WHERE user_id = 101;

-- 총 7번의 쿼리 (개선 전: 10번)
-- 참가자가 10명이면? 13번의 쿼리 (개선 전: 20번)
```

#### 성능 개선 결과

```
개선 전:
- 참가자 2명: ~50ms (5번 쿼리)
- 참가자 4명: ~150ms (10번 쿼리)
- 참가자 10명: ~400ms (20번 쿼리)

개선 후:
- 참가자 2명: ~30ms (4번 쿼리) ✅ 40% 개선
- 참가자 4명: ~50ms (7번 쿼리) ✅ 67% 개선
- 참가자 10명: ~100ms (13번 쿼리) ✅ 75% 개선

→ 참가자 수가 많을수록 개선 효과가 큼!
```

---

### 4.  느낀 점 및 교훈 (Retrospect)

이번 트러블슈팅을 통해 **"작은 최적화가 큰 차이를 만든다"**는 것을 실감했습니다.

#### 기술적 교훈

1. **N+1 쿼리는 숨어있다**:
   - 2~3명으로 테스트할 때는 문제를 느끼지 못했습니다.
   - 하지만 참가자 수가 증가하면서 성능 저하가 명확해졌습니다.
   - **"지금은 괜찮다"가 아니라 "확장 가능한가"를 고민해야 합니다.**

2. **JPA의 findAllById() 활용**:
   - JPA는 이미 N+1 문제를 해결할 수 있는 도구를 제공합니다.
   - `findAllById()`는 자동으로 IN 쿼리를 생성하여 일괄 조회합니다.
   - 프레임워크가 제공하는 기능을 적극 활용해야 합니다.

3. **Bulk Insert의 중요성**:
   - `saveAll()`을 사용하면 여러 개의 INSERT를 한 번에 처리합니다.
   - 개별 `save()`를 반복하는 것보다 훨씬 효율적입니다.

4. **Map을 활용한 O(1) 조회**:
   - List를 순회하며 `findById()`를 호출하면 O(N) × O(1) = O(N)
   - Map으로 변환하면 O(1)로 조회 가능
   - 자료구조 선택이 성능에 큰 영향을 미칩니다.

#### 성능 최적화 관점

처음에는 "동작만 하면 된다"고 생각했습니다. 하지만:
- 2명일 때는 50ms
- 4명일 때는 150ms
- 10명일 때는 400ms

이런 식으로 증가하면, 100명이 동시에 게임을 종료하면 어떻게 될까요? 서버가 감당할 수 없을 것입니다.

**"지금 당장의 성능"이 아니라 "확장 가능한 성능"**을 고민해야 한다는 것을 배웠습니다.

#### 코드 리뷰의 중요성

사실 이 N+1 문제는 코드 리뷰 과정에서 발견되었습니다. 혼자서는 "잘 동작하네!"라고 생각했지만, 다른 개발자의 시각에서 보니 명확한 문제점이 보였습니다.

**"동작하는 코드"와 "좋은 코드"는 다릅니다.** 코드 리뷰를 통해 더 나은 코드를 작성할 수 있다는 것을 다시 한번 느꼈습니다.

#### 가장 큰 깨달음

**"성능 문제는 예방이 최선"**입니다.

이번에는 다행히 개발 단계에서 발견했지만, 만약 프로덕션에 배포된 후 발견했다면?
- 사용자들이 "게임이 느려요"라고 불평
- 급하게 핫픽스 배포
- 서비스 신뢰도 하락

이런 상황을 피하려면:
1. 개발 단계에서 성능을 고려
2. 다양한 규모로 테스트 (2명, 4명, 10명...)
3. 쿼리 로그를 확인하는 습관
4. 코드 리뷰에서 성능 이슈 체크

**"나중에 최적화하자"가 아니라 "처음부터 제대로 하자"**는 마음가짐이 중요하다는 것을 배웠습니다.

---

###  추가 개선 가능 사항

현재는 User의 totalScore 업데이트가 개별 UPDATE로 처리됩니다. 이것도 최적화할 수 있습니다:

```java
// 현재: Dirty Checking으로 개별 UPDATE (4번 쿼리)
user.updateTotalScore(roundScore);

// 개선 가능: Bulk UPDATE (1번 쿼리)
@Modifying
@Query("UPDATE User u SET u.totalScore = u.totalScore + :score WHERE u.id = :userId")
void updateTotalScore(@Param("userId") Long userId, @Param("score") Integer score);

// 또는 JPQL Batch Update
@Modifying
@Query("UPDATE User u SET u.totalScore = u.totalScore + " +
       "CASE u.id " +
       "WHEN :userId1 THEN :score1 " +
       "WHEN :userId2 THEN :score2 " +
       "END " +
       "WHERE u.id IN (:userIds)")
```

하지만 현재 구조에서는 Dirty Checking이 더 간단하고 안전하므로, 참가자 수가 수십 명 이상으로 증가하기 전까지는 현재 방식을 유지하기로 결정했습니다.

**"과도한 최적화는 독이 될 수 있다"**는 것도 함께 배웠습니다.
