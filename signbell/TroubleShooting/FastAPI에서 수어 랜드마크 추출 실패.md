
## 트러블슈팅: FastAPI에서 수어 랜드마크 추출 실패

* **작성자:** [고동현](https://github.com/rhehdgus8831)

-----

### 1\. 문제 현상 (Problem)

> 게임 중 수어 동작을 수행해도 FastAPI 서버가 영상 프레임에서 랜드마크를 감지하지 못하고 계속 0점 처리되는 문제 발생

* **문제 1**: 게임 중 본인 차례에 수어 동작을 해도 FastAPI 서버가 랜드마크를 추출하지 못함
* **문제 2**: 브라우저 콘솔에 `"오류: 랜드마크를 감지하지 못했습니다. (데이터 없음)"` 로그 반복 출력
* **문제 3**: 모든 수어 동작 시도가 0점으로 처리되어 게임 진행 불가

<br>

**[문제 상황 스크린샷 및 로그]**

> FastAPI WebSocket으로부터 랜드마크 감지 실패 메시지를 수신하는 로그
<img width="458" height="529" alt="dh-ts-4" src="https://github.com/user-attachments/assets/0c6b2d2b-f33c-46be-9812-143af7fabd08" />
<img width="1861" height="598" alt="dh-ts5" src="https://github.com/user-attachments/assets/f51b0d96-c8de-4620-b821-7612e102cc43" />

-----

### 2\. 원인 분석 (Analysis)

> FastAPI 연결 및 카메라 스트림 자체는 정상이지만, React 컴포넌트 조건부 렌더링 시 비디오 엘리먼트에 스트림이 제대로 다시 연결되지 않아 Canvas가 빈 프레임을 캡처하여 FastAPI로 전송한 것이 원인

* **원인 1: 비디오 엘리먼트 렌더링 및 스트림 재연결 실패**
    * 게임 단계(`gamePhase`)가 변경될 때 `QuizMainVideo` 컴포넌트가 조건부 렌더링되면서 내부의 `video` 엘리먼트가 재생성됨
    * `useEffect` 훅에서 스트림을 `video` 엘리먼트의 `srcObject`에 할당했지만, 엘리먼트가 DOM에 완전히 렌더링되기 전에 실행되어 연결이 누락됨
    * 결과적으로 `video` 엘리먼트는 화면에 보이지만 실제 스트림 데이터는 연결되지 않은 상태가 됨
* **원인 2: Canvas의 빈 프레임 캡처**
    * `captureFrame` 함수는 `video` 엘리먼트로부터 현재 프레임을 `canvas`에 그림
    * `video` 엘리먼트에 스트림이 연결되지 않았으므로, `canvas`는 빈 화면(검은색) 프레임만 캡처함
    * 이 빈 프레임 데이터가 FastAPI로 전송되니 MediaPipe가 랜드마크를 감지할 수 없었음

-----

### 3\. 해결 방안 (Solution)

> `QuizMainVideo` 컴포넌트 내 `useEffect`에서 비디오 엘리먼트에 스트림을 연결할 때, ref가 유효하지 않으면 `setTimeout`을 이용해 짧은 지연 후 재시도하는 로직 추가

* **[해결 방안 1]**: `video` 엘리먼트의 ref(`videoRef.current`)가 실제로 DOM에 마운트되고 유효한지 확인 후 스트림(`srcObject`)을 할당
* **[해결 방안 2]**: `useEffect` 실행 시점에 `videoRef.current`가 아직 `null`이거나 준비되지 않았을 경우, `setTimeout`으로 100ms 정도 지연시킨 후 스트림 연결을 재시도하여 렌더링 타이밍 문제 해결


<img width="475" height="220" alt="dh-ts-3" src="https://github.com/user-attachments/assets/a78ccff2-83b3-4a22-aa29-241dfa178365" />

-----

### 4\. 교훈 (Lessons Learned)

> MediaPipe 랜드마크 추출은 카메라 스트림이 활성화된 것뿐만 아니라, 해당 스트림이 비디오 엘리먼트에 **정확히 연결**되고 화면에 **실제로 렌더링**되어야만 정상 동작함

* **[교훈 1]**: React에서 조건부 렌더링으로 미디어 엘리먼트(`video`, `audio`)가 재생성될 때는 스트림 연결 로직이 DOM 렌더링 완료 후에 실행되도록 타이밍을 신중하게 관리해야 함 (`useEffect` 의존성 배열, `setTimeout` 등 활용)
* **[교훈 2]**: 미디어 스트림 관련 문제 발생 시, 스트림 활성 여부(`stream.active`) 확인과 더불어 실제 엘리먼트(`videoRef.current`, `srcObject`)에 스트림이 제대로 연결되었는지 반드시 교차 확인해야 함
* **[교훈 3]**: Canvas API(`drawImage`)는 `video` 엘리먼트의 현재 시각적 상태를 캡처하므로, 비디오 엘리먼트에 소스(스트림)가 연결되지 않으면 빈 프레임이 캡처될 수 있음을 인지해야 함
