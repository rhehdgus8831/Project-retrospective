# 🧯 SignBell 트러블슈팅

> ## **"도전 신청을 하지 않은 참가자로 인한 게임 진행 중단 문제"**
> (일부 참가자가 도전 신청을 하지 않으면 게임이 멈추는 문제)

---

### 1.  문제 상황 (Problem)

#### 시도 및 과정
- 퀴즈 게임의 도전 신청 시스템을 구현했습니다.
- 문제가 시작되면 참가자들이 "도전 신청" 버튼을 클릭하여 선착순 4명까지 도전할 수 있도록 설계했습니다.
- 도전 신청한 사람들이 순서대로 문제를 풀고, 모두 실패하면 다음 문제로 넘어가는 로직을 구현했습니다.

#### 긍정적 관찰
- 모든 참가자가 도전 신청을 하는 경우, 게임이 정상적으로 진행되었습니다.
- 도전자들이 순서대로 문제를 풀고, 정답/오답에 따라 점수가 정확하게 계산되었습니다.

#### 핵심 문제 발생
- **4명이서 게임을 진행하는데 1명이 도전 신청을 하지 않으면**, 3명이 모두 실패해도 다음 문제로 넘어가지만, **도전 신청을 하지 않은 1명은 점수를 얻을 기회가 없었습니다**.
- 더 심각한 문제는 **아무도 도전 신청을 하지 않으면 게임이 완전히 멈춰버리는 상황**이 발생했습니다.

<br>

**[관련 코드: 문제가 발생한 로직]**

```java
// QuizService.java - registerChallenge()
@Transactional
public void registerChallenge(Long roomId, Long userId, Integer questionNumber) {
    // ... 검증 로직 ...
    
    // 도전권 등록 (최대 4명)
    boolean registered = roomState.addChallenger(questionNumber, userId);

    if (registered) {
        Integer order = roomState.getChallengerOrder(questionNumber, userId);
        
        // 첫 번째 도전자라면 차례 알림
        if (order == 1) {
            roomState.setCurrentChallenger(questionNumber, userId);
            
            // ❌ 문제: 첫 번째 도전자에게 바로 차례를 줌
            // → 다른 사람들이 도전 신청할 시간이 없음
            messagingTemplate.convertAndSend(...);
        }
    }
}
```

**[시나리오별 문제 상황]**

```
시나리오 1: 4명 중 1명이 도전 신청 안 함
┌─────────────────────────────────────┐
│ 참가자: A, B, C, D (총 4명)          │
├─────────────────────────────────────┤
│ 문제 1 시작                          │
│ A, B, C만 도전 신청 (D는 안 함)      │
│ A 틀림 → B 틀림 → C 틀림             │
│ → 다음 문제로 이동 ✅                │
│                                     │
│ 결과: D는 점수 0점                   │
│ 문제: D가 계속 도전 안 하면?         │
│ → 게임에 참여하지 않는 것과 같음     │
└─────────────────────────────────────┘

시나리오 2: 아무도 도전 신청 안 함
┌─────────────────────────────────────┐
│ 참가자: A, B, C, D (총 4명)          │
├─────────────────────────────────────┤
│ 문제 1 시작                          │
│ 아무도 도전 신청 안 함               │
│                                     │
│ ❌ 게임이 멈춤!                      │
│ → 다음 문제로 넘어가지 않음          │
│ → 참가자들이 대기만 함               │
└─────────────────────────────────────┘
```

---

### 2.  원인 분석 (Analysis)

#### 가설 1 (가장 유력한 원인): 타임아웃 메커니즘 부재
- **문제의 핵심**: 도전 신청을 "언제까지" 받을지에 대한 시간 제한이 없었습니다.
- 참가자들이 도전 신청을 하지 않으면 게임이 무한정 대기하는 상태가 되었습니다.
- 선착순 4명이라는 제한은 있었지만, "몇 명이 신청해야 게임이 진행되는가"에 대한 로직이 없었습니다.

#### 가설 2 (부차적인 원인): 도전 신청의 강제성 부족
- 도전 신청이 선택사항이었기 때문에, 참가자가 문제를 모르거나 풀기 싫으면 신청하지 않을 수 있었습니다.
- 이는 게임 디자인 관점에서 공정성 문제를 야기했습니다.

#### 결론
현재 구조는 "도전 신청을 하는 사람이 있다"는 가정 하에 설계되었습니다. 하지만 실제 게임에서는:
1. 문제가 어려워서 아무도 도전하지 않을 수 있음
2. 일부 참가자만 도전하고 나머지는 포기할 수 있음
3. 네트워크 문제로 도전 신청이 늦어질 수 있음

이러한 상황들을 고려하지 않아 게임이 멈추는 치명적인 문제가 발생했습니다.

<br>

**[분석 과정에서 발견한 추가 이슈]**

```java
// notifyNextChallenger() 메서드 분석
private void notifyNextChallenger(Long roomId, Integer questionNumber) {
    Long nextChallenger = roomState.getNextChallenger(questionNumber);
    
    if (nextChallenger != null) {
        // 다음 도전자에게 알림
    } else {
        // ✅ 다음 도전자가 없으면 다음 문제로
        moveToNextQuestion(roomId, questionNumber);
    }
}
```

**발견**: 도전자가 모두 실패한 경우는 처리되어 있었지만, **애초에 도전자가 없는 경우**는 처리되지 않았습니다.

---

### 3.  해결 방안 (Solution)

#### 접근 방식 전환
- **"도전 신청을 기다리는" 수동적 방식**에서 **"일정 시간 후 자동으로 진행하는" 능동적 방식**으로 전환했습니다.
- 게임 디자인 관점에서 "타임아웃"을 도입하여 게임의 템포를 유지하기로 결정했습니다.

#### 구체적인 해결책: 10초 타임아웃 시스템

**1. 문제 시작 시 10초 타이머 자동 시작**
```java
@Transactional
public GameStartResponse startGame(Long roomId, Long userId) {
    // ... 기존 로직 ...
    
    // 첫 번째 문제 전송
    messagingTemplate.convertAndSend(...);

    // ✅ 10초 타이머 스케줄링
    scheduleChallengeTimeout(roomId, 1);
    
    return response;
}
```

**2. 타이머 스케줄링 구현**
```java
private void scheduleChallengeTimeout(Long roomId, Integer questionNumber) {
    CompletableFuture.delayedExecutor(10, TimeUnit.SECONDS)
            .execute(() -> {
                try {
                    handleChallengeTimeout(roomId, questionNumber);
                } catch (Exception e) {
                    log.error("타임아웃 처리 중 오류", e);
                }
            });
    
    log.info("도전 신청 타임아웃 스케줄링 - 10초");
}
```

**3. 타임아웃 처리 로직**
```java
private void handleChallengeTimeout(Long roomId, Integer questionNumber) {
    QuizStateCache.GameRoomState roomState = quizStateCache.getOrCreateRoomState(roomId);
    
    // 도전자 수 확인
    Integer challengerCount = roomState.getChallengerCount(questionNumber);
    
    if (challengerCount == 0) {
        // ✅ 아무도 도전 신청 안 함 - 다음 문제로 강제 이동
        messagingTemplate.convertAndSend(
            "/topic/room/" + roomId + "/quiz",
            ApiResponse.success("도전 신청 시간이 종료되었습니다. 다음 문제로 이동합니다.")
        );
        
        moveToNextQuestion(roomId, questionNumber);
    } else {
        // ✅ 도전 신청한 사람이 있음 - 첫 번째 도전자에게 차례 부여
        Long firstChallenger = roomState.getFirstChallenger(questionNumber);
        roomState.setCurrentChallenger(questionNumber, firstChallenger);
        
        messagingTemplate.convertAndSend(
            "/topic/room/" + roomId + "/quiz",
            ApiResponse.success("도전 신청이 마감되었습니다. 도전자 차례입니다.")
        );
    }
}
```

**4. 도전 신청 로직 수정**
```java
@Transactional
public void registerChallenge(Long roomId, Long userId, Integer questionNumber) {
    // ... 검증 로직 ...
    
    boolean registered = roomState.addChallenger(questionNumber, userId);

    if (registered) {
        // ✅ 첫 번째 도전자에게 바로 차례를 주지 않음
        // → 10초 타임아웃 후 일괄 처리
        
        // 개인 메시지: 도전 신청 완료
        messagingTemplate.convertAndSendToUser(...);
        
        // 브로드캐스트: 현재 도전자 수
        messagingTemplate.convertAndSend(...);
    }
}
```

#### 최종 해결 결과

```
시나리오 1: 일부만 도전 신청
┌─────────────────────────────────────┐
│ 문제 시작 → 10초 타이머 시작         │
│ A, B만 도전 신청 (C, D는 안 함)      │
│ 10초 후 자동 마감                    │
│ A에게 차례 → A 틀림                  │
│ B에게 차례 → B 틀림                  │
│ → 다음 문제로 자동 이동 ✅           │
└─────────────────────────────────────┘

시나리오 2: 아무도 도전 신청 안 함
┌─────────────────────────────────────┐
│ 문제 시작 → 10초 타이머 시작         │
│ 아무도 도전 신청 안 함               │
│ 10초 후 자동 마감                    │
│ "도전 신청 시간 종료" 메시지         │
│ → 다음 문제로 자동 이동 ✅           │
└─────────────────────────────────────┘
```

---

### 4.  느낀 점 및 교훈 (Retrospect)

이번 트러블슈팅을 통해 **"사용자가 항상 예상대로 행동한다"는 가정이 얼마나 위험한지** 깨달았습니다.

#### 기술적 교훈
1. **타임아웃은 필수**: 사용자 입력을 기다리는 모든 시스템에는 타임아웃이 필요합니다.
2. **엣지 케이스 고려**: "아무도 행동하지 않는 경우"도 반드시 고려해야 합니다.
3. **비동기 처리의 중요성**: `CompletableFuture`를 활용한 스케줄링으로 게임 흐름을 제어할 수 있었습니다.

#### 게임 디자인 관점
- 처음에는 "도전 신청을 강제로 해야 하나?"라는 고민을 했습니다.
- 하지만 타임아웃 시스템을 도입하면서, **선택의 자유를 주면서도 게임의 템포를 유지**할 수 있다는 것을 알게 되었습니다.
- 도전하지 않는 것도 전략이 될 수 있지만, 그 대가로 점수를 얻을 기회를 잃는다는 명확한 규칙이 생겼습니다.

#### 협업 관점
- 프론트엔드 팀에게 "10초 타이머 UI"를 요청하면서, 백엔드와 프론트엔드의 역할 분담이 명확해졌습니다.
- 백엔드는 타이머 로직과 상태 관리를, 프론트엔드는 시각적 피드백을 담당하는 구조가 자연스럽게 형성되었습니다.

#### 가장 큰 깨달음
**"게임이 멈추지 않게 하는 것"**이 가장 중요한 요구사항이었습니다. 아무리 복잡한 로직을 구현해도, 게임이 멈춰버리면 사용자 경험은 최악이 됩니다. 이번 타임아웃 시스템 도입으로 **어떤 상황에서도 게임이 계속 진행되는 안정성**을 확보할 수 있었습니다.
