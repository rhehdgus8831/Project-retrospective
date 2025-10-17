# 🧯 SignBell 트러블슈팅

> ## **"게임 중 참가자 연결 해제 시 게임 진행 중단 및 데이터 불일치 문제"**
> (참가자가 게임 중 창을 닫으면 게임이 멈추고 랭킹에 오류가 발생하는 문제)

---

### 1.  문제 상황 (Problem)

#### 시도 및 과정
- 게임 진행 중 참가자의 WebSocket 연결 해제를 감지하는 `WebSocketEventsListener`를 구현했습니다.
- 연결 해제 시 자동으로 방에서 퇴장 처리하는 `GameRoomLeaveService`를 구현했습니다.
- 게임 종료 시 캐시에서 점수를 조회하여 랭킹을 생성하고 DB에 저장하는 로직을 구현했습니다.

#### 긍정적 관찰
- 대기실에서 참가자가 나가는 경우, 정상적으로 퇴장 처리되고 다른 참가자들에게 알림이 전송되었습니다.
- 게임이 정상적으로 종료되면 점수가 정확하게 계산되고 랭킹이 올바르게 표시되었습니다.

#### 핵심 문제 발생
1. **게임 중 참가자가 창을 닫으면**:
   - 캐시에는 해당 참가자의 점수가 남아있음
   - DB에서는 `GameParticipant`가 삭제됨
   - 게임 종료 시 캐시의 userId로 User를 조회하면 실패
   - 랭킹 생성 중 `continue`로 건너뛰면서 순위가 1, 2, 4등으로 건너뜀

2. **도전 차례인 사람이 나가면**:
   - `currentChallenger`가 나간 사람의 userId로 남아있음
   - 아무도 답을 제출할 수 없어 게임이 멈춤
   - 타임아웃도 없어서 영구적으로 대기 상태

<br>

**[관련 코드: 문제가 발생한 로직]**

```java
// QuizService.java - endGame()
@Transactional
public void endGame(Long roomId) {
    Map<Long, Integer> roundScores = roomState.getAllUserScores();
    
    // 점수 높은 순으로 정렬
    List<Map.Entry<Long, Integer>> sortedScores = roundScores.entrySet().stream()
            .sorted(Map.Entry.<Long, Integer>comparingByValue().reversed())
            .collect(Collectors.toList());
    
    // User 조회
    List<Long> userIds = sortedScores.stream()
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
    
    Map<Long, User> userMap = userRepository.findAllById(userIds).stream()
            .collect(Collectors.toMap(User::getId, user -> user));
    
    // ❌ 문제: 퇴장한 사람은 User를 못 찾음
    for (int i = 0; i < sortedScores.size(); i++) {
        Long userId = entry.getKey();
        User user = userMap.get(userId);
        
        if (user == null) {
            log.warn("유저를 찾을 수 없습니다: {}", userId);
            continue;  // ❌ 순위가 1, 2, 4등으로 건너뜀
        }
        
        rankings.add(RankingInfo.builder()
                .rank(i + 1)  // ❌ 잘못된 순위
                ...
        );
    }
}
```

**[문제 상황 로그]**

```bash
# 게임 중 참가자 퇴장
INFO  - User 123 leaving room 1
INFO  - 참가자 정보 삭제 완료
INFO  - 게임방 현재 인원: 3명

# 게임 종료 시
INFO  - 게임 종료 - roomId: 1, participants: 4
WARN  - 랭킹 처리 중 ID 123 에 해당하는 유저를 찾을 수 없습니다.
INFO  - GameHistory 저장 - userId: 456, rank: 1
INFO  - GameHistory 저장 - userId: 789, rank: 2
INFO  - GameHistory 저장 - userId: 101, rank: 3
# ❌ userId 123은 랭킹에서 누락됨
```

**[시나리오별 문제 상황]**

```
시나리오 1: 일반 참가자가 게임 중 나감
┌─────────────────────────────────────┐
│ 참가자: A(123), B(456), C(789), D(101) │
├─────────────────────────────────────┤
│ 게임 진행 중...                      │
│ A의 점수: 150점 (캐시에 저장됨)      │
│                                     │
│ A가 창을 닫음 (연결 해제)            │
│ → DB에서 GameParticipant 삭제        │
│ → 캐시에는 점수 그대로 남아있음      │
│                                     │
│ 게임 종료                            │
│ → 캐시에서 A의 점수 조회: 150점 ✅   │
│ → DB에서 A의 User 조회: 실패 ❌     │
│ → A는 랭킹에서 제외됨                │
│ → 순위: 1등(B), 2등(C), 3등(D)       │
│   (A가 1등이었는데 누락됨)           │
└─────────────────────────────────────┘

시나리오 2: 도전 차례인 사람이 나감
┌─────────────────────────────────────┐
│ 문제 3번 진행 중                     │
│ 도전 순서: A → B → C                 │
│ 현재 차례: A (currentChallenger=123) │
│                                     │
│ A가 창을 닫음 (연결 해제)            │
│ → DB에서 GameParticipant 삭제        │
│ → 캐시의 currentChallenger는 그대로 │
│                                     │
│ ❌ 게임이 멈춤!                      │
│ → A를 기다리지만 A는 없음            │
│ → B, C는 답을 제출할 수 없음         │
│ → 타임아웃 없어서 영구 대기          │
└─────────────────────────────────────┘
```

---

### 2. 🧐 원인 분석 (Analysis)

#### 가설 1 (가장 유력한 원인): 캐시와 DB의 데이터 불일치
- **문제의 핵심**: 참가자가 퇴장하면 DB에서는 삭제되지만, 캐시에는 데이터가 남아있었습니다.
- `GameRoomLeaveService`는 DB만 정리하고 캐시는 정리하지 않았습니다.
- 게임 종료 시 캐시의 userId로 DB를 조회하면서 불일치가 발생했습니다.

#### 가설 2 (부차적인 원인): 도전 차례 관리 미흡
- 도전 차례인 사람이 나가도 `currentChallenger`가 업데이트되지 않았습니다.
- 다음 도전자에게 자동으로 차례를 넘기는 로직이 없었습니다.

#### 가설 3: 랭킹 순위 계산 로직 오류
- `for (int i = 0; i < sortedScores.size(); i++)`에서 `i`를 그대로 순위로 사용했습니다.
- `continue`로 건너뛰면 순위가 1, 2, 4등으로 건너뛰는 문제가 발생했습니다.

#### 결론
게임 중 퇴장은 **"예외 상황"이 아닌 "정상 시나리오"**로 처리해야 했습니다. 단순히 DB에서 삭제하는 것이 아니라:
1. 캐시에서도 해당 참가자의 데이터를 정리
2. 도전 차례였다면 다음 도전자에게 넘김
3. 게임 종료 시 퇴장한 사람은 랭킹에서 제외

이 세 가지를 모두 처리해야 게임이 정상적으로 진행됩니다.

<br>

**[분석 과정에서 발견한 추가 이슈]**

```java
// GameRoomLeaveService.java - handleParticipantLeave()
private ParticipantEventResponse handleParticipantLeave(...) {
    // 1. 참가자 정보 삭제
    participantRepository.delete(participant);
    
    // 2. 방 인원 수 감소
    room.decrementParticipants();
    
    // ❌ 3. 게임 진행 중인지 확인 안 함
    // ❌ 4. 캐시 정리 안 함
    
    gameRoomRepository.save(room);
    return response;
}
```

**발견**: 게임 상태를 확인하지 않고 무조건 DB만 정리했습니다.

---

### 3. 💡 해결 방안 (Solution)

#### 접근 방식 전환
- **"퇴장은 대기실에서만 발생한다"는 가정**을 버리고, **"게임 중 퇴장도 정상 시나리오"**로 처리하기로 했습니다.
- 퇴장 시 게임 상태를 확인하고, 게임 중이면 캐시도 함께 정리하는 로직을 추가했습니다.

#### 구체적인 해결책

**1. 게임 중 퇴장 감지 및 캐시 정리**

```java
// GameRoomLeaveService.java
private ParticipantEventResponse handleParticipantLeave(...) {
    // 1. 참가자 정보 삭제
    participantRepository.delete(participant);
    
    // 2. 방 인원 수 감소
    room.decrementParticipants();
    
    // ✅ 3. 게임 진행 중이면 캐시 정리
    if (room.getStatus() == GameRoomStatus.IN_PROGRESS) {
        handleGameInProgressLeave(gameRoomId, userId);
    }
    
    gameRoomRepository.save(room);
    return response;
}

private void handleGameInProgressLeave(Long roomId, Long userId) {
    QuizStateCache.GameRoomState roomState = quizStateCache.getOrCreateRoomState(roomId);
    
    // ✅ 1. 캐시에서 점수 제거
    roomState.removeUserScore(userId);
    
    // ✅ 2. 모든 문제에서 도전 순서 제거
    for (int questionNumber = 1; questionNumber <= 8; questionNumber++) {
        Long currentChallenger = roomState.getCurrentChallenger(questionNumber);
        boolean wasCurrentChallenger = userId.equals(currentChallenger);
        
        // 도전 순서에서 제거
        roomState.removeChallenger(questionNumber, userId);
        
        // ✅ 3. 도전 차례였다면 다음 도전자에게 넘김
        if (wasCurrentChallenger) {
            Long nextChallenger = roomState.getNextChallenger(questionNumber);
            if (nextChallenger != null) {
                log.info("다음 도전자로 변경: {}", nextChallenger);
            }
        }
    }
}
```

**2. QuizStateCache에 제거 메서드 추가**

```java
// QuizStateCache.java
public static class GameRoomState {
    // ✅ 사용자 점수 제거
    public void removeUserScore(Long userId) {
        userScores.remove(userId);
    }

    // ✅ 도전자 제거
    public void removeChallenger(Integer questionNumber, Long userId) {
        // 큐에서 제거
        Queue<Long> queue = challengerQueues.get(questionNumber);
        if (queue != null) {
            queue.remove(userId);
        }

        // 순서 맵에서 제거
        Map<Long, Integer> orders = challengerOrders.get(questionNumber);
        if (orders != null) {
            orders.remove(userId);
        }

        // 현재 도전자였다면 제거
        Long currentChallenger = currentChallengers.get(questionNumber);
        if (userId.equals(currentChallenger)) {
            currentChallengers.remove(questionNumber);
        }
    }
}
```

**3. 랭킹 순위 계산 로직 수정**

```java
// QuizService.java - endGame()
@Transactional
public void endGame(Long roomId) {
    // ... 기존 로직 ...
    
    // ✅ 실제 랭킹 카운터 추가
    int rank = 1;
    for (int i = 0; i < sortedScores.size(); i++) {
        Long userId = entry.getKey();
        User user = userMap.get(userId);
        
        if (user == null) {
            // ✅ 게임 중 퇴장한 사용자는 랭킹에서 제외
            log.info("게임 중 퇴장한 사용자 랭킹 제외 - userId: {}", userId);
            continue;  // 순위 증가 안 함
        }
        
        // GameHistory 저장
        historiesToSave.add(GameHistory.builder()
                .score(roundScore)
                .round(completedRound)
                .build());
        
        // User totalScore 업데이트
        user.updateTotalScore(roundScore);
        
        // ✅ 순위 정보 생성 (실제 랭킹 사용)
        rankings.add(RankingInfo.builder()
                .rank(rank)  // ✅ 연속된 순위
                .userId(user.getId())
                .score(roundScore)
                .build());
        
        rank++;  // ✅ 다음 순위로 증가
    }
}
```

#### 최종 해결 결과

```
시나리오 1: 일반 참가자가 게임 중 나감
┌─────────────────────────────────────┐
│ A가 창을 닫음 (연결 해제)            │
│ → DB에서 GameParticipant 삭제 ✅     │
│ → 캐시에서 점수 제거 ✅              │
│ → 모든 문제의 도전 순서에서 제거 ✅  │
│                                     │
│ 게임 종료                            │
│ → 캐시에 A의 점수 없음 ✅            │
│ → A는 랭킹에서 제외됨 ✅             │
│ → 순위: 1등(B), 2등(C), 3등(D) ✅    │
│   (연속된 순위)                      │
└─────────────────────────────────────┘

시나리오 2: 도전 차례인 사람이 나감
┌─────────────────────────────────────┐
│ 현재 차례: A (currentChallenger=123) │
│                                     │
│ A가 창을 닫음 (연결 해제)            │
│ → 캐시에서 currentChallenger 제거 ✅ │
│ → 자동으로 다음 도전자(B)에게 ✅     │
│                                     │
│ ✅ 게임 계속 진행!                   │
│ → B가 답을 제출할 수 있음            │
│ → 게임이 멈추지 않음                 │
└─────────────────────────────────────┘
```

---

### 4.  느낀 점 및 교훈 (Retrospect)

이번 트러블슈팅을 통해 **"데이터 일관성"의 중요성**을 뼈저리게 느꼈습니다.

#### 기술적 교훈

1. **캐시와 DB의 동기화**: 
   - DB를 수정하면 캐시도 함께 수정해야 합니다.
   - 한쪽만 정리하면 데이터 불일치가 발생합니다.

2. **상태 기반 처리**:
   - 단순히 "퇴장 처리"가 아니라 "어떤 상태에서 퇴장했는가"를 확인해야 합니다.
   - 대기실 퇴장과 게임 중 퇴장은 완전히 다른 처리가 필요합니다.

3. **순위 계산 로직**:
   - 반복문의 인덱스를 그대로 순위로 사용하면 안 됩니다.
   - `continue`로 건너뛰는 경우를 고려한 별도의 카운터가 필요합니다.

#### 설계 관점

처음에는 "퇴장은 예외 상황"이라고 생각했습니다. 하지만 실제로는:
- 네트워크 끊김
- 브라우저 강제 종료
- 의도적인 이탈

이 모든 것이 **정상적으로 발생할 수 있는 시나리오**였습니다. 예외 상황을 정상 시나리오로 받아들이고 설계하는 것이 중요하다는 것을 배웠습니다.

#### 협업 관점

이 문제를 해결하면서 **"게임이 멈추지 않는 것"**이 최우선 목표라는 것을 다시 한번 확인했습니다. 
- 퇴장한 사람의 점수를 기록하지 않는 것은 괜찮습니다.
- 하지만 남은 사람들의 게임이 멈춰서는 안 됩니다.

이러한 우선순위를 명확히 하면서, 어떤 기능을 먼저 구현해야 하는지 판단할 수 있었습니다.

#### 가장 큰 깨달음

**"완벽한 데이터보다 안정적인 서비스"**가 더 중요합니다. 
- 퇴장한 사람의 GameHistory가 저장되지 않는 것은 아쉽지만, 게임이 계속 진행되는 것이 더 중요합니다.
- 데이터 정합성을 유지하면서도 서비스의 안정성을 해치지 않는 균형점을 찾는 것이 핵심이었습니다.

이번 경험을 통해 **"방어적 프로그래밍"**의 중요성을 깨달았고, 앞으로는 "이런 일은 일어나지 않을 것"이라는 가정 대신 "이런 일이 일어나면 어떻게 할 것인가"를 먼저 고민하게 되었습니다.
