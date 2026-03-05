# Puppeteer 헤드리스 브라우저로 배치 캡처하기

웹에서 돌아가는 3D 뷰어(또는 캔버스 앱)를 **여러 에셋에 대해 자동으로 열고, 프레임을 캡처해서 WebP/WebM으로 저장**하려면 Puppeteer로 페이지를 띄우고, 각 URL마다 “캡처 준비 대기 → 한 프레임씩 캡처 → FFmpeg으로 인코딩” 파이프라인을 돌리게 됩니다.  
실전에서 부딪힌 점과 해결 방법을 정리한 글입니다.

---

## 1. 전체 흐름

1. **캡처 앱 서버** 실행 (예: Vite dev, `http://localhost:5173`).
2. **Puppeteer**로 브라우저 실행 → 새 페이지 열기.
3. 에셋별로 **URL 생성** (예: `?path=asset/foo.glb&fps=15`).
4. **page.goto(url)** 후 `window.captureReady`(또는 `captureError`) **대기**.
5. **Warmup:** 처음 N프레임은 버리고, 카메라·포즈가 안정된 뒤부터 캡처.
6. **한 프레임씩** `page.evaluate(() => window.captureNextFrame({ mimeType: 'image/webp', quality: 1 }))` 호출.
7. 반환된 **data URL**을 Base64 디코딩해 **FFmpeg stdin**으로 WebP 버퍼 스트리밍.
8. **window.isMotionFinished**가 true이거나 **최대 프레임 수**에 도달하면 스트림 종료 → WebM 파일 저장.
9. (선택) 첫 프레임 버퍼를 **정적 WebP**로도 저장.

---

## 2. 페이지 준비 대기

- 3D 에셋 로드와 씬 초기화가 끝나야 `captureNextFrame`이 동작합니다.
- 앱에서 **`window.captureReady = true`** (또는 에러 시 `window.captureError = '...'`) 를 설정하도록 하고, Puppeteer에서는:

```js
await page.waitForFunction(
  () => Boolean(window.captureReady) || Boolean(window.captureError),
  { timeout: LOAD_TIMEOUT_MS }
)
const err = await page.evaluate(() => window.captureError || null)
if (err) throw new Error(`capture app error: ${err}`)
```

- 타임아웃은 에셋 크기에 맞게 넉넉히 (예: 90초). `networkidle2` 로 goto하면 대부분의 리소스가 끝난 뒤에 진행할 수 있습니다.

---

## 3. Warmup 프레임

- 첫 몇 프레임은 캐릭터가 아직 T-pose에서 내려오거나, 카메라가 pelvis를 따라잡기 전이라 어색할 수 있습니다.
- **처음 10~15프레임은 “캡처”로 쓰지 않고 버리고**, 그 다음부터 FFmpeg에 넘기면 품질이 안정됩니다.
- Warmup 중에 이미 `isMotionFinished`가 true가 되면, 그 마지막 프레임 하나라도 WebM에 넣어주는 식으로 예외 처리하면 좋습니다.

---

## 4. 페이지 주기적 재시작 (메모리 누수 완화)

- 한 페이지에서 **에셋을 많이 바꾸며 캡처**하면, Three.js/WebGL 리소스가 쌓여서 메모리가 늘어날 수 있습니다.
- **N개 에셋 처리할 때마다 page를 닫고 새 페이지를 열어서** 재사용하는 방식으로 완화할 수 있습니다. (예: 20개마다 `page.close()` 후 `browser.newPage()`.)
- 브라우저는 한 번만 띄우고, **페이지만** 주기적으로 갈아끼우는 것이 일반적입니다.

---

## 5. Chrome/Chromium 실행 경로

- Puppeteer는 기본적으로 번들된 Chromium을 쓰지만, **시스템에 설치된 Chrome**을 쓰고 싶을 때가 있습니다. (예: 코덱·GPU 정책이 다를 때.)
- OS별로 흔한 경로를 배열로 두고 `fs.existsSync`로 확인한 뒤 `executablePath`에 넘기는 방식이 많이 쓰입니다.
  - macOS: `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`
  - Linux: `/usr/bin/google-chrome`, `/usr/bin/chromium` 등
  - Windows: `%LOCALAPPDATA%\Google\Chrome\Application\chrome.exe` 등
- `process.env.PUPPETEER_EXECUTABLE_PATH`가 있으면 그걸 우선 사용하도록 하면 CI/로컬 설정이 유연해집니다.

---

## 6. Headless 환경에서 GPU/SwiftShader - 비추!!

- 서버·CI처럼 **GPU가 없거나 WebGL이 제한된 환경**에서는 `--use-gl=angle --use-angle=swiftshader` 같은 플래그로 **SwiftShader**를 쓰게 할 수 있습니다.
- 이렇게 하면 소프트웨어 렌더링이라 느리지만, “렌더가 아예 안 되는” 상황을 피할 수 있습니다.  
  환경 변수(예: `FORCE_SWIFTSHADER=true`)로 켜고 끄도록 두면 디버깅에 편합니다.

---

## 7. 뷰포트·타임아웃

- **고정 뷰포트:** `page.setViewport({ width: 1280, height: 820, deviceScaleFactor: 1 })` 처럼 고정하면, 캡처 해상도가 에셋마다 일정해져서 인코딩 옵션을 맞추기 쉽습니다.
- **setDefaultTimeout:** 페이지 내 `waitForFunction`, `goto` 등에 공통 타임아웃을 두면, 느린 에셋에서 무한 대기를 막을 수 있습니다.

---

## 8. 실패 로그와 재실행

- 어떤 에셋에서 실패했는지 **경로 + 에러 메시지**를 JSON 배열로 저장해 두면, 나중에 `BATCH_OFFSET` / `BATCH_LIMIT` 으로 구간만 다시 돌리거나, 실패 목록만 재시도하는 식으로 활용할 수 있습니다.
- **SKIP_EXISTING:** 이미 출력 파일(WebP + WebM)이 있으면 건너뛰는 옵션을 두면, 중단된 배치를 이어서 실행하기 편합니다.

---

## 요약

| 항목 | 내용 |
|------|------|
| 대기 | `captureReady` / `captureError` 로 초기화 완료 후에만 캡처. |
| Warmup | 처음 N프레임은 버리고, 그 다음부터 FFmpeg에 전달. |
| 메모리 | N개마다 페이지를 닫고 새 페이지로 교체. |
| Chrome | executablePath를 OS별 후보로 찾거나 env로 지정. |
| Headless GPU | SwiftShader로 소프트웨어 렌더링 fallback. |
| 재실행 | SKIP_EXISTING + 실패 목록 JSON으로 구간/실패만 재시도. |

캡처 API 설계는 `웹에서 캔버스 프레임을 이미지로 캡처하기.md`를, WebM/GIF 인코딩은 `FFmpeg으로 WebP 프레임을 WebM·GIF로 인코딩하기.md`를 참고하면 됩니다.
