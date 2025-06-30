
# ✨ SYNC-UP 팀 프로젝트 회고록

## 📌 프로젝트 개요
- **프로젝트명**: Study With Me
- **기간**: 2025년 6월 23일 ~ 2025년 7월 2일
- **주제**: 학습 및 일정 관리를 위한 올인원 웹 학습 플래너 개발
- **목표**: JavaScript를 활용해 스톱워치, ToDoList, AI 학습 보조, 학습 로그 등 핵심 기능을 모듈화하여 구현하고, Git/GitHub을 통한 체계적인 협업 역량 강화

---

## 🛠 사용 기술 스택

- **Frontend**
  
  <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/html5/html5-original.svg" alt="HTML5" width="40" height="40"/>
  <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/css3/css3-original.svg" alt="CSS3" width="40" height="40"/>
  <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/javascript/javascript-original.svg" alt="JavaScript" width="40" height="40"/>

- **협업 & 형상관리**
  
  <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/git/git-original.svg" alt="Git" width="40" height="40"/>
  <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/github/github-original.svg" alt="GitHub" width="40" height="40"/>
  <img src="https://www.svgrepo.com/show/353655/discord-icon.svg" alt="Discord" width="40" height="40"/>
  

---

## 👥 역할 분담
| 역할 | 이름 | 담당 |
|------|------|------|
| **팀장** | **고동현** | **스톱워치 기능 개발, 명언 API 연동, GitHub 관리 및 PR 총괄, README 작성 등 프로젝트 리딩** |
| 팀원 | 김경민 | 할 일 목록(TodoList) 기능 개발 |
| 팀원 | 백승현 | 로컬 스토리지 데이터 기반 학습 업적 로그 생성 |
| 팀원 | 송민재 | Gemini API를 활용한 AI 학습 보조 기능 개발, 최종 발표 |

---

## 🧯 트러블슈팅


### **문제 2: 스톱워치 리셋 후에도 이전 측정 시간이 누적되는 문제**

#### **문제 상황**
1.  **동작:** 스톱워치 '시작' 버튼을 눌러 시간을 측정하고, '일시정지'를 누른 뒤, '리셋' 버튼을 클릭
2.  **화면 반응:** 화면의 타이머(`stopwatch-display`)는 `00:00:00`으로 정상적으로 초기화
3.  **버그 발생:** 이 상태에서 다시 '시작' 버튼을 누르자, 타이머가 `00:00:00`부터 시작하지 않고 리셋하기 직전의 시간부터 이어서 측정되는 문제가 발생. 예를 들어 15분에서 리셋했다면, 다시 시작 시 15분 1초부터 시간이 흘러함.

#### **원인 분석**
-   **핵심 원인:** `resetStopwatch` 함수는 `clearInterval`로 타이머의 반복 동작을 멈추고, `textContent`를 수정해 화면 표시만 초기화. 하지만 스톱워치의 실제 경과 시간을 밀리초(ms) 단위로 저장하는 핵심 상태 변수인 **`elapsedTime`을 0으로 초기화하는 로직이 누락**되어 있었음.
-   **코드 동작 흐름:**
    1.  `startStopwatch` 함수는 `setInterval`을 통해 10ms마다 `elapsedTime` 변수에 시간을 누적
    2.  `pauseStopwatch` 함수는 `setInterval`을 멈추지만, `elapsedTime` 값은 그대로 유지
    3.  이 상태에서 `resetStopwatch`가 호출되었을 때, (버그가 있던 코드에서는) `elapsedTime`이 그대로 남아있었음.
    4.  사용자가 다시 `startStopwatch`를 호출하면, `setInterval`은 0이 아닌, 이전에 누적된 `elapsedTime` 값에 계속해서 10ms를 더하기 시작. 이것이 버그의 직접적인 원인이었음.

-   **문제 원인 코드 (수정 전)**
    ```javascript
    function resetStopwatch(e) {
        // 타이머 반복 중지
        clearInterval(timerIntervalId);
        timerIntervalId = null;

        // 누적 시간 저장 (이 기능은 정상)
        saveTime(elapsedTime);

        // elapsedTime = 0;  // <--- 이 핵심 초기화 코드가 누락된 상태

        // 화면 표시만 초기화
        $stopWatchDisplay.textContent = '00:00:00';
        $pauseBtn.textContent = '일시정지';
        changeState(false);
    }
    ```

#### **해결 방법**
-   **UI와 데이터 상태의 동기화:** `resetStopwatch` 함수 내에서 화면 표시(`textContent`)를 초기화할 뿐만 아니라, 실제 데이터인 `elapsedTime` 변수도 `0`으로 명시적으로 초기화하여 UI와 데이터 상태를 일치시킴.
-   **기록 보존:** 사용자의 총 공부 시간 기록을 위해, `elapsedTime`을 `0`으로 초기화하기 **직전에** `saveTime(elapsedTime)` 함수를 호출했다. 이 순서를 통해 현재 세션의 기록은 안전하게 로컬 스토리지에 누적시키고, 그 후에 현재 타이머 상태만 깨끗하게 초기화할 수 있었음음.

-   **수정 후 최종 코드 (`stopwatch.js`)**
    ```javascript
    function resetStopwatch(e) {
        // 1. 실행 중인 인터벌을 정지시켜 더 이상 시간이 누적되지 않게 함
        clearInterval(timerIntervalId);
        timerIntervalId = null;

        // 2. 현재까지 누적된 elapsedTime을 로컬 스토리지에 저장
        saveTime(elapsedTime);

        // 3. [핵심 수정] 실제 경과 시간 변수를 0으로 초기화
        elapsedTime = 0;

        // 4. 휴식 시간 모달 관련 상태 변수들도 초기화
        hasShownBreak = false;
        breakTargetTime = null;

        // 5. 화면 표시와 버튼 상태를 모두 초기 상태로 되돌림
        $stopWatchDisplay.textContent = '00:00:00';
        $pauseBtn.textContent = '일시정지';
        changeState(false); // 버튼 활성화/비활성화 상태 초기화
    }
    ```
-   위와 같이 수정함으로써 리셋 버튼은 **화면 표시, 내부 데이터, 관련 상태 변수들을 모두 일관성 있게 초기화**하는 본연의 기능을 완벽하게 수행하게 되었음.



---



### **문제 2: API 호출 결과가 명언 모달에 정상적으로 반영되지 않는 문제**

#### **문제 상황**
1.  **API 호출 실패 테스트:** `catch` 블록의 에러 핸들링을 확인하기 위해 의도적으로 API URL을 틀리게 변경 후 '오늘의 명언' 버튼을 클릭
2.  **예상 결과:** 모달 창에 `textEl.textContent = '명언을 불러오지 못했어요 🧐'`가 실행되어 에러 메시지가 표시되어야함.
3.  **실제 결과:** 에러 메시지는 나타나지 않고, HTML에 하드코딩되어 있던 초기 텍스트가 그대로 보임.
4.  **API 정상화 후 테스트:** API URL을 다시 정상으로 수정한 후 실행했지만, 여전히 API에서 받아온 명언이 아닌 초기 텍스트만 표시되었음.

#### **원인 분석**
-   `loadQuote` 함수 내에서 명언을 표시할 DOM 요소를 선택할 때 `document.querySelector('.quote-text')`와 같이 클래스 선택자만을 사용했음.
-   `document.querySelector()`는 문서 전체에서 **가장 먼저 발견되는 요소 하나만**을 반환함.
-   HTML 구조를 확인한 결과, '오늘의 명언' 모달(`id="quoteModal"`)보다 **'휴식 시간' 알림 모달(`id="break-modal"`)이 먼저 선언**되어 있었음.
-   두 모달 모두 동일한 클래스명(`class="quote-text"`)을 가진 `<p>` 태그를 포함하고 있어, 자바스크립트는 항상 의도치 않은 '휴식 시간' 모달 내부의 요소를 선택하고 있었음.
-   결과적으로 API 호출 성공/실패와 관계없이 모든 텍스트 업데이트가 **사용자에게 보이지 않는 '휴식 시간' 모달에서만 일어나고 있었던 것**이 근본적인 원인

#### **해결 방법**
-   DOM 요소를 선택하는 범위를 '오늘의 명언' 모달(`id="quoteModal"`) 내부로 한정하여, 다른 모달의 요소와 혼동되지 않도록 선택자를 구체적으로 수정

-   **수정 전 코드 (`quoteAPI.js`)**
    ```javascript
    const nameEl = document.querySelector('.quote-name');
    const profileEl = document.querySelector('.quote-profile');
    const textEl = document.querySelector('.quote-text');
    ```

-   **수정 후 코드 (`quoteAPI.js`)**
    ```javascript
    // #quoteModal을 앞에 추가하여 선택 범위를 명확히 함
    const nameEl = document.querySelector('#quoteModal .quote-name');
    const profileEl = document.querySelector('#quoteModal .quote-profile');
    const textEl = document.querySelector('#quoteModal .quote-text');
    ```
-   위와 같이 수정함으로써 `loadQuote` 함수는 명확하게 '오늘의 명언' 모달 내의 요소들만 제어하게 되어 문제가 해결

---

## 📚 배운 점 / 느낀 점

- **팀장 역할과 문서화의 중요성**:
  
  이번 프로젝트에서 팀장 역할과 README 작성을 도맡으면서, 단순히 기능을 개발하는 것을 넘어 **프로젝트 전체의 진행 상황을 조율하고 명확한 기록을 남기는 것이 얼마나 중요한지** 깨달았음.\
   특히, 커밋 규칙, 브랜치 전략, 폴더 구조 등을 초기에 문서로 명확히 정의해두면 팀원의 혼동을 줄일 수 있으며 개발에 집중할 수 있다는 것을 느꼈음.

- **모듈화 설계의 중요성**:
  
  기능별로 JS와 CSS 파일을 분리하는 모듈화 방식을 채택한 것은 효율적이였음. 각자 독립된 환경에서 작업할 수 있어 **작업 간섭과 코드 충돌을 최소화**할 수 있었고,\
  `app.js`에서 각 모듈을 `import`하는 방식으로 코드를 통합하니 전체 구조가 매우 깔끔하고 유지보수하기 쉬워졌음.

- **소통을 통한 디버깅**:
  
  `z-index` 충돌이나 HTML 구조 충돌과 같은 문제들은 결국 **팀원들과의 적극적인 소통으로 해결**되었음.\
  매일 진행한 스크럼 회의와 GitHub 이슈, PR 코멘트를 통해 문제를 빠르게 공유하고 해결책을 논의했던 경험을 통해, 기술적인 문제 해결의 시작은 소통이라는 것을 다시 한번 느꼈음.

---

## 🚀 다음 프로젝트에 반영할 점

- **초기 설계 단계에 더 많은 시간 투자하기**:
  
  다음 프로젝트에서는 본격적인 개발에 앞서 **HTML/CSS의 큰 구조와 컴포넌트 규칙,전역 스타일(z-index, 폰트, 색상 등)을 더 구체적으로 설계**해야함.\
  초반에 시간을 조금 더 투자하는 것이 후반부의 불필요한 충돌 해결 비용을 극적으로 줄여준다는 것을 배웠음.

- **체계적인 코드 리뷰 문화 정착**:

  이번에는 PR을 통한 충돌 해결에 집중했지만, 다음에는 **기능적인 부분에 대한 코드 리뷰를 더 활성화**하고 싶음.\
  단순히 코드 병합 승인을 넘어, "이 로직은 이렇게 개선하면 더 효율적일 것 같아요"와 같은 건설적인 피드백을 주고받는 문화를 정착시켜 팀 전체의 코드 품질을 높이고 싶음.

- **PR 템플릿 적극 활용**:
  
  이번 경험을 바탕으로, 다음 프로젝트에서는 **GitHub의 PR 템플릿 기능을 활용하여 '변경 사항 요약', '테스트 방법', '관련 이슈 번호' 등을 의무적으로 작성**하게 양식을 지정할 예정.\
  이를 통해 구두 설명 없이도 누구나 PR의 의도를 명확히 파악하고 효율적으로 리뷰할 수 있게 만들어야함.
