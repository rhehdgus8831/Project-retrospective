# 트러블슈팅: WebRTC 웹캠 상태 관리 및 비디오 렌더링 문제

**작성자:** 고동현  
**작성일:** 2025-10-23

---

## 1. 문제 현상 (Problem)

> 게임 페이지에서 웹캠이 자동으로 켜지지 않고, 상대방 카메라가 보이지 않으며, 작은 플레이어 카드의 비디오가 깜빡이는 문제 발생

### 문제 상세

* **문제 1**: 대기실에서 웹캠을 켜고 게임 페이지로 이동하면 웹캠 상태가 초기화되어 수동으로 다시 켜야 함
* **문제 2**: 게임 페이지에서 상대방의 웹캠 화면이 보이지 않고, remoteStreams에 자기 자신의 스트림이 포함됨
* **문제 3**: 게임 페이지 하단의 작은 플레이어 카드에서 내 웹캠이 처음에 보이지 않다가, 도전 후 돌아오면 보이며, 비디오가 계속 깜빡임

### 문제 상황 로그

> 웹캠 상태가 true인데도 화면에 표시되지 않고, 자기 자신의 스트림을 구독하는 문제

```bash
# 웹캠 상태는 true이지만 화면에 안 보임
📹 웹캠 초기화 체크: {isWebcamOn: true, initialized: false, attempted: false}
✅ 웹캠 이미 켜져 있음 - 유지

# 자기 자신의 스트림을 구독하는 문제
📺 User 2 연결 시도: {hasVideoElement: true, hasRemoteStream: true, streamActive: true}
✅ 비디오 엘리먼트에 스트림 연결 완료, User ID: 2
# (myUserId가 2인데 User 2의 스트림을 remoteStreams에서 가져옴)

# 비디오가 계속 재연결되면서 깜빡임
✅ 작은 카드 - 내 스트림 연결
✅ 작은 카드 - 내 스트림 연결
✅ 작은 카드 - 내 스트림 연결
```

---

## 2. 원인 분석 (Analysis)

> 컴포넌트 간 상태 공유 부재, Janus WebRTC 자기 구독 문제, React ref 타이밍 이슈가 복합적으로 발생

### 원인 1: 웹캠 상태가 컴포넌트 간 공유되지 않음

* 대기실(`QuizWaitingRoom`)과 게임 페이지(`QuizGamePage`)가 각각 독립적으로 `useWebcam` 훅을 호출
* 각 컴포넌트가 자체 로컬 상태로 웹캠을 관리하여, 페이지 이동 시 상태가 초기화됨
* MediaStream 객체는 직렬화할 수 없어 localStorage에 저장 불가능

**문제 코드:**
```javascript
// QuizWaitingRoom.jsx
const { stream, isWebcamOn, startWebcam } = useWebcam(); // 로컬 상태

// QuizGamePage.jsx
const { stream, isWebcamOn, startWebcam } = useWebcam(); // 별도의 로컬 상태
// → 페이지 이동 시 상태가 공유되지 않음
```

### 원인 2: Janus WebRTC에서 자기 자신의 스트림을 구독

* Janus VideoRoom의 `publishers` 목록에 자기 자신도 포함되는데, 이를 필터링하지 않고 모두 구독
* `remoteStreams` 객체에 자신의 스트림이 들어가서 상대방 스트림을 덮어씀
* 게임 페이지에서 하드코딩된 `tempUserId = 1`을 사용하여 실제 사용자 ID와 불일치

**문제 코드:**
```javascript
// 기존 참가자 구독 - 자기 자신도 구독함
if (msg['publishers']) {
  msg['publishers'].forEach((publisher) => {
    const userId = parseInt(publisher.display);
    userIdToFeedIdRef.current[userId] = publisher.id;
    subscribeToFeed(publisher.id, userId); // 자기 자신도 구독!
  });
}

// 하드코딩된 사용자 ID
const tempUserId = 1; // 실제 사용자 ID와 다를 수 있음
```

### 원인 3: React ref와 스트림 연결 타이밍 문제

* 여러 video 엘리먼트가 하나의 `videoRef`를 공유하여 마지막 엘리먼트만 ref를 가짐
* useEffect로 스트림을 연결하려 했지만, ref가 설정되기 전에 실행되어 연결 실패
* 컴포넌트가 렌더링될 때마다 ref 콜백이 실행되면서 `srcObject`를 계속 재설정하여 비디오 깜빡임

**문제 코드:**
```javascript
// 여러 video가 같은 ref 사용
const videoRef = useRef(null);

// 메인 비디오
<video ref={videoRef} />

// 작은 비디오 (같은 ref 사용!)
<video ref={videoRef} />

// useEffect로 연결 시도 (타이밍 문제)
useEffect(() => {
  if (videoRef.current && stream) {
    videoRef.current.srcObject = stream; // ref가 아직 null일 수 있음
  }
}, [stream]);

// ref 콜백에서 매번 재설정 (깜빡임)
<video
  ref={el => {
    if (el && stream) {
      el.srcObject = stream; // 매 렌더링마다 실행!
    }
  }}
/>
```

---

## 3. 해결 방안 (Solution)

> Zustand를 활용한 전역 상태 관리, Janus 구독 필터링, ref 콜백 최적화로 문제 해결

### 해결 방안 1: Zustand store로 웹캠 상태 전역 관리

* `useWebcamStore.js` 생성하여 `stream`, `isWebcamOn`, `startWebcam()`, `stopWebcam()` 등을 전역 상태로 관리
* 대기실과 게임 페이지 모두 동일한 store를 참조하여 페이지 이동 시에도 웹캠 상태 유지
* MediaStream 객체는 메모리에만 저장하고 persist 사용하지 않음

**해결 코드:**
```javascript
// frontend/src/stores/useWebcamStore.js
import { create } from 'zustand';

const useWebcamStore = create((set, get) => ({
  // 웹캠 스트림
  stream: null,
  
  // 웹캠 켜짐 여부
  isWebcamOn: false,
  
  // 에러 메시지
  error: null,

  // 웹캠 시작
  startWebcam: async () => {
    try {
      const mediaStream = await navigator.mediaDevices.getUserMedia({
        video: {
          width: { ideal: 1280 },
          height: { ideal: 720 },
          facingMode: 'user'
        },
        audio: false
      });

      set({ 
        stream: mediaStream, 
        isWebcamOn: true, 
        error: null 
      });

      // 트랙 종료 감지
      mediaStream.getTracks().forEach(track => {
        track.onended = () => {
          console.warn('⚠️ 웹캠 트랙이 종료됨');
          set({ isWebcamOn: false });
        };
      });

      console.log('✅ 웹캠 시작 성공');
      return mediaStream;
    } catch (err) {
      console.error('❌ 웹캠 시작 실패:', err);
      set({ 
        error: err.message, 
        isWebcamOn: false 
      });
      throw err;
    }
  },

  // 웹캠 중지
  stopWebcam: () => {
    const { stream } = get();
    if (stream) {
      stream.getTracks().forEach(track => {
        track.stop();
      });
      set({ 
        stream: null, 
        isWebcamOn: false 
      });
      console.log('🛑 웹캠 중지');
    }
  },

  // 웹캠 토글
  toggleWebcam: async () => {
    const { isWebcamOn, startWebcam, stopWebcam } = get();
    if (isWebcamOn) {
      stopWebcam();
    } else {
      await startWebcam();
    }
  },
}));

export default useWebcamStore;
```

**사용 예시:**
```javascript
// QuizWaitingRoom.jsx & QuizGamePage.jsx
import useWebcamStore from '../../stores/useWebcamStore';

const stream = useWebcamStore(state => state.stream);
const isWebcamOn = useWebcamStore(state => state.isWebcamOn);
const startWebcam = useWebcamStore(state => state.startWebcam);
const stopWebcam = useWebcamStore(state => state.stopWebcam);
```

### 해결 방안 2: Janus WebRTC 구독 시 자기 자신 필터링

* `publishers` 목록을 순회할 때 `userId !== myUserId` 조건으로 자기 자신 제외
* 게임 페이지에서 Zustand의 실제 `myUserId` 사용 (하드코딩 제거)
* 대기실과 게임 페이지 모두 동일한 필터링 로직 적용

**해결 코드:**
```javascript
// QuizGamePage.jsx
// Zustand에서 실제 사용자 ID 가져오기
const actualUserId = myUserId;
if (!actualUserId) {
  console.error('❌ 사용자 ID가 없습니다.');
  return;
}

// 기존 참가자 구독 (자기 자신 제외)
if (msg['publishers']) {
  msg['publishers'].forEach((publisher) => {
    const userId = parseInt(publisher.display);
    console.log('📺 기존 참가자 발견:', publisher.display, 'Feed ID:', publisher.id, '내 ID:', actualUserId);
    
    // 자기 자신은 구독하지 않음
    if (userId !== actualUserId) {
      userIdToFeedIdRef.current[userId] = publisher.id;
      subscribeToFeed(publisher.id, userId);
    } else {
      console.log('⏭️ 자기 자신은 구독 스킵');
    }
  });
}

// 새 참가자 입장 (자기 자신 제외)
if (msg['publishers']) {
  msg['publishers'].forEach((publisher) => {
    const userId = parseInt(publisher.display);
    console.log('📺 새 참가자 입장:', publisher.display, 'Feed ID:', publisher.id, '내 ID:', actualUserId);
    
    // 자기 자신은 구독하지 않음
    if (userId !== actualUserId) {
      userIdToFeedIdRef.current[userId] = publisher.id;
      subscribeToFeed(publisher.id, userId);
    } else {
      console.log('⏭️ 자기 자신은 구독 스킵');
    }
  });
}
```

### 해결 방안 3: ref 콜백으로 즉시 스트림 연결 및 중복 방지

* 각 video 엘리먼트에 ref 콜백을 사용하여 생성 즉시 스트림 연결
* `el.srcObject !== stream` 조건으로 이미 연결된 스트림은 재설정하지 않음
* 메인 비디오와 작은 비디오를 분리하여 독립적으로 관리

**해결 코드:**
```javascript
// QuizGamePage.jsx
// 메인 비디오 ref (분리)
const mainVideoRef = useRef(null);

// 메인 비디오 (내 차례일 때)
<video
  ref={mainVideoRef}
  autoPlay
  playsInline
  muted
  className={styles.webcamVideo}
/>

// 작은 플레이어 카드 - 내 비디오
{player.isMe && isWebcamOn ? (
  gamePhase === 'myTurn' ? (
    <span>내 차례</span>
  ) : (
    <video
      ref={el => {
        // 이미 연결된 스트림이면 재설정하지 않음
        if (el && stream && el.srcObject !== stream) {
          el.srcObject = stream;
        }
      }}
      autoPlay
      playsInline
      muted
      className={styles.smallWebcamVideo}
    />
  )
) : null}

// 작은 플레이어 카드 - 상대방 비디오
{!player.isMe && remoteStreams[player.id] ? (
  <video
    ref={el => {
      // 이미 연결된 스트림이면 재설정하지 않음
      if (el && remoteStreams[player.id] && el.srcObject !== remoteStreams[player.id]) {
        remoteVideosRef.current[player.id] = el;
        el.srcObject = remoteStreams[player.id];
      }
    }}
    autoPlay
    playsInline
    className={styles.smallWebcamVideo}
  />
) : null}
```

---

## 4. 교훈 (Lessons Learned)

> React 상태 관리, WebRTC 구독 로직, ref 타이밍에 대한 깊은 이해 필요

### 교훈 1: 페이지 간 공유가 필요한 상태는 전역 상태 관리 도구 사용

* 컴포넌트 로컬 상태는 언마운트 시 소멸되므로, 페이지 이동 시 유지가 필요한 상태는 Zustand, Redux 등 전역 상태 관리 필수
* MediaStream 같은 직렬화 불가능한 객체는 persist 없이 메모리에만 저장
* Context API보다 Zustand가 더 간결하고 성능이 좋음

**Best Practice:**
```javascript
// ❌ 나쁜 예: 컴포넌트 로컬 상태
const [stream, setStream] = useState(null);

// ✅ 좋은 예: Zustand 전역 상태
const stream = useWebcamStore(state => state.stream);
```

### 교훈 2: WebRTC에서 자기 자신을 구독하지 않도록 명시적 필터링 필요

* Janus VideoRoom의 `publishers` 목록에는 자기 자신도 포함될 수 있으므로 항상 userId 비교 필수
* 하드코딩된 ID 대신 실제 사용자 ID를 사용하여 일관성 유지
* 디버깅 시 로그에 `myUserId`를 함께 출력하여 구독 대상 확인

**Best Practice:**
```javascript
// ❌ 나쁜 예: 모든 참가자 구독
msg['publishers'].forEach((publisher) => {
  subscribeToFeed(publisher.id, userId);
});

// ✅ 좋은 예: 자기 자신 제외
msg['publishers'].forEach((publisher) => {
  const userId = parseInt(publisher.display);
  if (userId !== myUserId) {
    subscribeToFeed(publisher.id, userId);
  }
});
```

### 교훈 3: React에서 동적 ref 관리 시 ref 콜백과 중복 방지 로직 활용

* 여러 엘리먼트가 같은 ref를 공유하면 마지막 엘리먼트만 참조되므로 각각 독립적으로 관리
* useEffect보다 ref 콜백이 타이밍 문제 없이 즉시 실행되어 더 안정적
* `srcObject` 재설정 시 기존 값과 비교하여 불필요한 재연결 방지로 깜빡임 해결

**Best Practice:**
```javascript
// ❌ 나쁜 예: 여러 video가 같은 ref 사용
const videoRef = useRef(null);
<video ref={videoRef} />
<video ref={videoRef} /> // 마지막 것만 참조됨

// ❌ 나쁜 예: 매번 재설정 (깜빡임)
<video
  ref={el => {
    if (el && stream) {
      el.srcObject = stream; // 매 렌더링마다 실행
    }
  }}
/>

// ✅ 좋은 예: ref 콜백 + 중복 방지
<video
  ref={el => {
    if (el && stream && el.srcObject !== stream) {
      el.srcObject = stream; // 다른 스트림일 때만 연결
    }
  }}
/>
```

---

## 5. 결과

### 해결 완료 항목

✅ 대기실에서 게임 페이지로 이동 시 웹캠 상태 유지  
✅ 상대방 카메라 정상 표시  
✅ 작은 플레이어 카드에서 내 카메라 즉시 표시  
✅ 비디오 깜빡임 현상 제거  

### 성능 개선

* 불필요한 스트림 재연결 제거로 렌더링 성능 향상
* 자기 자신 구독 제거로 네트워크 트래픽 감소
* 전역 상태 관리로 코드 일관성 및 유지보수성 향상

---

## 6. 참고 자료

* [Zustand 공식 문서](https://github.com/pmndrs/zustand)
* [Janus WebRTC Gateway 문서](https://janus.conf.meetecho.com/docs/)
* [React useRef 공식 문서](https://react.dev/reference/react/useRef)
* [MediaStream API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream)
