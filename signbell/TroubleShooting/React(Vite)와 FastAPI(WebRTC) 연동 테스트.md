## 🧯 트러블슈팅: React(Vite)와 FastAPI(WebRTC) 연동 테스트

### **문제: React 클라이언트와 FastAPI AI 서버 간의 통신 실패**

#### **문제 상황**

1.  **초기 증상**: React(Vite) 클라이언트에서 FastAPI 서버로 API 요청 시, 브라우저 콘솔에 `TypeError: Failed to fetch`, `CORS policy` 위반, `404 Not Found` 등 다양한 네트워크 오류가 복합적으로 발생했습니다.
2.  **서버 상태**: FastAPI 서버는 로컬에서 정상적으로 실행되고 모델 로딩까지 완료된 상태였지만, 클라이언트의 요청을 제대로 처리하지 못하는 것처럼 보였습니다.
3.  **연결 불확실성**: Vite 프록시, FastAPI의 CORS 설정 등 여러 해결 방법을 시도했음에도 불구하고, 연결 자체가 불안정하거나 특정 방식(WebSocket)에서 연결이 전혀 이루어지지 않는 문제가 지속되었습니다.

#### **원인 분석**

  - **1차 원인 (환경 설정 미비)**: 개발 초기 단계에서 Windows 환경의 **인코딩 문제**(`UnicodeDecodeError: 'cp949'`)나 **C++ 빌드 도구 부재**(`Microsoft Visual C++ 14.0 is required`)로 인해 서버의 의존성 라이브러리가 정상적으로 설치되지 않았습니다.
  - **2차 원인 (CORS 및 프록시 설정 오류)**: 개발 환경에서 React(`http://localhost:5173`)와 FastAPI(`http://127.0.0.1:8000`)는 서로 다른 출처(Origin)를 가집니다. 이로 인한 브라우저의 CORS 정책을 해결하기 위해 FastAPI의 `CORSMiddleware`와 Vite의 `server.proxy` 설정을 적용했지만, WebSocket(`ws: true`) 프록시 설정 누락 등 초기 설정이 미흡했습니다.
  - **근본 원인 (아키텍처 불일치)**: 가장 핵심적인 원인은 **클라이언트와 서버가 서로 다른 통신 방식을 기대**하고 있었다는 점입니다.
      * \*\*클라이언트(React)\*\*는 초기 버전에서 '5초 녹화 후 파일 업로드'(`fetch` API) 방식을 시도했습니다.
      * \*\*서버(FastAPI)\*\*는 최종적으로 '실시간 프레임 스트리밍'(`WebSocket` + `WebRTC DataChannel`) 방식의 연결을 기다리고 있었습니다.
      * 결과적으로 클라이언트는 서버에 존재하지 않는 API 주소(`/predict_video`)로 계속 요청을 보내 `404 Not Found` 오류가 발생했고, 서버는 클라이언트로부터 어떠한 WebSocket 연결 시도도 받지 못하는 교착 상태에 빠졌습니다.

-----

### **해결 방법**

#### **1. 서버 환경 안정화 (Dependency Hell 탈출)**

  - **인코딩 문제 해결**: PowerShell에서 `$env:PYTHONUTF8=1` 환경 변수를 설정하여 `requirements.txt` 파일을 UTF-8로 읽도록 강제했습니다.
  - **C++ 컴파일 문제 우회**: 무거운 Visual Studio 빌드 도구를 설치하는 대신, [Unofficial Windows Binaries for Python Extension Packages](https://www.lfd.uci.edu/~gohlke/pythonlibs/)에서 미리 컴파일된 `av` 라이브러리(`.whl` 파일)를 다운로드하여 직접 설치(`pip install <파일명>.whl`)하는 방식으로 의존성 문제를 해결했습니다.

##### 인코딩 문제 로그
```
 SignBell-FASTAPI  pip install -r requirements.txt
ERROR: Exception:
Traceback (most recent call last):
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\cli\base_command.py", line 160, in exc_logging_wrapper
    status = run_func(*args)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\cli\req_command.py", line 247, in wrapper
    return func(self, options, args)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\commands\install.py", line 363, in run
    reqs = self.get_requirements(args, options, finder, session)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\cli\req_command.py", line 433, in get_requirements
    for parsed_req in parse_requirements(
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 145, in parse_requirements
    for parsed_line in parser.parse(filename, constraint):
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 327, in parse
    yield from self._parse_and_recurse(filename, constraint)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 332, in _parse_and_recurse
    for line in self._parse_file(filename, constraint):
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 363, in _parse_file
    _, content = get_file_content(filename, self._session)
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\req\req_file.py", line 541, in get_file_content
    content = auto_decode(f.read())
  File "C:\Users\user\Desktop\kdh\SignBell-FASTAPI\venv\lib\site-packages\pip\_internal\utils\encoding.py", line 34, in auto_decode
    return data.decode(
UnicodeDecodeError: 'cp949' codec can't decode byte 0xec in position 27: illegal multibyte sequence
```

##### C++ 빌드 도구 오류 (Microsoft Visual C++ 14.0) 로그

```
Building wheels for collected packages: av
  Building wheel for av (pyproject.toml) ... error
  error: subprocess-exited-with-error

  × Building wheel for av (pyproject.toml) did not run successfully.
  │ exit code: 1
  ╰─> [98 lines of output]
      C:\Users\user\AppData\Local\Temp\pip-build-env-w31n3ni9\overlay\Lib\site-packages\setuptools\_distutils\dist.py:289: UserWarning: Unknown distribution option: 'test_suite'
        warnings.warn(msg)
      C:\Users\user\AppData\Local\Temp\pip-build-env-w31n3ni9\overlay\Lib\site-packages\setuptools\dist.py:759: SetuptoolsDeprecationWarning: License classifiers are deprecated.
      !!
      running bdist_wheel
      running build
      running build_py
      running build_ext
      building 'av.buffer' extension
      error: Microsoft Visual C++ 14.0 or greater is required. Get it with "Microsoft C++ Build Tools": https://visualstudio.microsoft.com/visual-cpp-build-tools/
      [end of output]

  note: This error originates from a subprocess, and is likely not a problem with pip.
  ERROR: Failed building wheel for av
Failed to build av
ERROR: Could not build wheels for av, which is required to install pyproject.toml-based projects
```

#### **2. 프론트엔드 환경 안정화 (Legacy Library 연동)**

**🚨 문제 상황: `janus.js`와 같은 레거시(Legacy) 라이브러리 연동 오류**

`janus.js`와 같이 ES 모듈을 지원하지 않는 구식 라이브러리를 React 컴포넌트에서 `import` 하려고 할 때, `export default`가 없다는 모듈 시스템 오류나 `adapter is not defined` 와 같은 의존성 참조 오류가 발생했습니다.

**🧐 원인 분석**

`janus.js`는 `window` 전역 객체에 자신을 등록하는 방식으로 동작하며, WebRTC 브라우저 호환성을 위한 `adapter.js` 라이브러리가 먼저 로드되어야 하는 의존성을 가집니다. React의 현대적인 모듈 시스템은 이러한 구식 라이브러리와 직접 호환되지 않아 여러 오류를 발생시킵니다.

**✅ 해결 과정 및 최종 해결책**

이 문제를 해결하기 위해 두 단계의 접근 방식을 사용했습니다.

**1단계: 빠른 기능 검증을 위한 전역 로드 방식 (초기 테스트)**

> **목표**: 일단 가장 확실한 방법으로 라이브러리를 동작시켜 통신 기능 자체의 구현에 집중한다.

초기 테스트 단계에서는 `public/index.html`의 `<head>` 태그 안에 `<script>` 태그를 직접 추가하여 라이브러리를 전역으로 로드했습니다. 이는 구식 라이브러리를 연동하는 가장 간단하고 확실한 방법이지만, 다음과 같은 **성능 문제**를 야기합니다.

  - **문제점**: WebRTC 기능이 필요 없는 페이지(예: 메인 페이지, 로그인 페이지)를 방문하는 사용자에게도 불필요한 `janus.js`와 `adapter.js` 파일을 다운로드하게 하여 초기 로딩 속도를 저하시킵니다.

따라서 이 방법은 기능 연동을 빠르게 확인하기 위한 임시 방편으로만 사용했으며, 프로젝트의 구조가 안정화됨에 따라 아래의 최종 해결책으로 리팩토링할 것을 계획했습니다.

**2단계: 성능 최적화를 위한 동적 스크립트 로딩 (최종 해결책)**

> **목표**: WebRTC 기능이 **실제로 필요한 컴포넌트(`GameRoom.jsx`)가 렌더링될 때만** 스크립트를 로드하여 초기 로딩 성능을 최적화한다.

`useEffect` Hook을 사용하여 `GameRoom` 컴포넌트가 마운트되는 시점에 스크립트 파일을 동적으로 생성하고, 로드가 완료되면 Janus를 초기화하는 방식으로 코드를 개선했습니다.

1.  **`index.html` 정리**: `public/index.html`에 추가했던 `<script>` 태그 두 줄을 완전히 **삭제**합니다.

2.  **`GameRoom.jsx`에 동적 로더 구현**: 컴포넌트 내부에 스크립트를 순서대로 불러오는 로직을 추가합니다.

    ```jsx
    // GameRoom.jsx
    const [isJanusLoaded, setIsJanusLoaded] = useState(false); // 스크립트 로드 상태

    useEffect(() => {
        const loadScript = (url) => {
            return new Promise((resolve, reject) => {
                if (document.querySelector(`script[src="${url}"]`)) {
                    return resolve();
                }
                const script = document.createElement('script');
                script.src = url;
                script.async = true;
                script.onload = () => resolve();
                script.onerror = (err) => reject(err);
                document.head.appendChild(script);
            });
        };

        // 의존성 순서에 맞춰 동적으로 스크립트 로드
        loadScript('https://cdnjs.cloudflare.com/ajax/libs/webrtc-adapter/8.2.3/adapter.min.js')
            .then(() => loadScript('/janus.js')) // public 폴더에 janus.js가 위치해야 함
            .then(() => {
                if (window.Janus) {
                    setIsJanusLoaded(true); // 로드 완료!
                } else {
                    throw new Error("Janus object not found.");
                }
            })
            .catch(error => console.error("Script loading error:", error));

    }, []); // 컴포넌트 마운트 시 1회만 실행

    useEffect(() => {
        // 모든 스크립트가 로드된 후에 Janus 초기화 실행
        if (isJanusLoaded) {
            window.Janus.init({
                // ... Janus 초기화 로직 ...
            });
        }
    }, [isJanusLoaded]); // isJanusLoaded 상태가 true로 바뀌면 실행
    ```

**개선 결과**: 이 방식을 통해 WebRTC 기능이 불필요한 페이지의 성능 저하 문제를 해결하고, 관련 로직을 `GameRoom` 컴포넌트 내에 캡슐화하여 코드의 유지보수성을 높였습니다.

#### **3. 통신 아키텍처 통일 및 프록시 설정 완성**

  - **아키텍처 통일**: 서버의 '실시간 스트리밍' 방식에 맞춰, React 클라이언트(`GameRoom.jsx`)의 로직을 `WebSocket`과 `WebRTC DataChannel`을 사용하는 방식으로 전면 수정하여 클라이언트와 서버가 동일한 프로토콜로 대화하도록 만들었습니다.
  - **Vite 프록시 완성**: `vite.config.js`에 `server.proxy` 설정을 추가하여 HTTP 요청뿐만 아니라 WebSocket 요청까지 FastAPI 서버로 전달되도록 설정했습니다. 이 과정에서 `ws: true` 옵션을 추가하는 것이 결정적이었습니다.
    ```javascript
    // vite.config.js
    server: {
      proxy: {
        '/api': {
          target: 'http://12- 7.0.0.1:8000',
          changeOrigin: true,
          ws: true, // WebSocket 프록시 활성화
          rewrite: (path) => path.replace(/^\/api/, '')
        }
      }
    }
    ```

#### FAST API 서버 로그

<img width="982" height="131" alt="image" src="https://github.com/user-attachments/assets/30277e51-8ae2-4394-8fb1-70a0c24fa2a8" />

#### REACT 콘솔 로그

<img width="574" height="534" alt="image" src="https://github.com/user-attachments/assets/880b2a91-c483-4e6f-8ada-2100ba311954" />


#### AI 추론 결과

<img width="1137" height="507" alt="image" src="https://github.com/user-attachments/assets/822b17aa-d1dc-4b0c-ac59-d6ebc51d8788" />



-----

### **느낀 점**

이번 트러블슈팅은 단순히 코드 한 줄을 고치는 것이 아니라, 서로 다른 기술 스택으로 이루어진 각 서버의 환경과 아키텍처를 깊이 이해하는 과정이었습니다. `CORS`, `Proxy`, `WebSocket` 등 네트워크 개념들이 단순히 이론으로만 존재하는 것이 아니라, 실제 개발 과정에서 어떻게 서로 얽히고 문제를 일으키는지 명확하게 체감할 수 있었습니다.

특히 "분명히 코드는 맞는데 왜 안 되지?"라는 막막함 속에서, 브라우저와 서버 양쪽의 로그를 끈질기게 추적하며 클라이언트와 서버가 완전히 다른 기대를 하고 있었다는 **'아키텍처 불일치'** 라는 근본 원인을 찾아냈을 때 큰 희열을 느꼈습니다.

이 경험을 통해, 풀스택 개발에서 가장 중요한 것은 단순히 각 영역의 코드를 작성하는 능력을 넘어, **전체 시스템의 데이터 흐름을 조망하고 각 컴포넌트의 통신 방식을 명확하게 정의하는 '설계'의 중요성**이라는 것을 뼈저리게 깨달았습니다.
