# Puppeteer + Three.js 배치 캡처, 실전에서 터진 4가지 문제와 해결법

> **시리즈:** 3D 에셋 배치 파이프라인 구축기 (2/3)
> **대상 독자:** Puppeteer 자동화를 실무에서 써본 또는 써보려는 개발자

---

## 이론과 실제는 다르다

Puppeteer로 Three.js 앱을 캡처하는 구조는 깔끔해 보인다.

```
Puppeteer → page.evaluate() → window.captureNextFrame() → WebP → FFmpeg
```

그런데 실제로 수백 개를 돌리기 시작하면 이상한 일이 생긴다.
캐릭터가 T-포즈로 굳어 있거나, 4시간 뒤에 메모리가 폭발하거나, GPU 없는 서버에서 검은 화면만 나오거나.

이 글은 **실전에서 부딪힌 4가지 문제와 해결법**을 정리한 것이다.

---

## 문제 1: 첫 프레임이 T-포즈다

Three.js의 `AnimationMixer`는 `update(delta)` 호출 이후부터 포즈가 실제로 반영된다.
`captureReady = true`를 설정한 직후 첫 프레임을 캡처하면 캐릭터가 T-포즈인 경우가 많다.

FBX/BVH 파일은 특히 심하다. 스켈레톤 초기화 과정이 GLB보다 무거워서 수 프레임이 지나야 포즈가 안정된다.

### 해결: Warmup 프레임

처음 N프레임은 캡처하되 FFmpeg에 보내지 않고 버린다.

```javascript
// batch-convert.js
const WARMUP_FRAMES = {
  fbx: 15,
  bvh: 15,
  zip: 15,
  glb: 2,
}

const ext = path.extname(filePath).slice(1).toLowerCase()
const warmup = WARMUP_FRAMES[ext] ?? 2

// Warmup: 렌더는 하되, 버퍼에 쌓지 않는다
for (let i = 0; i < warmup; i++) {
  await page.evaluate(() => window.captureNextFrame({ mimeType: 'image/webp', quality: 1.0 }))
}

// 실제 캡처 시작
const frames = []
while (frames.length < MAX_FRAMES) {
  const { dataUrl, done } = await page.evaluate(() =>
    window.captureNextFrame({ mimeType: 'image/webp', quality: 1.0 })
  )
  frames.push(dataUrl)
  if (done) break
}
```

FBX는 15프레임 정도를 버려야 포즈가 안정된다.
GLB는 2프레임으로 충분하다. 포맷별로 다르게 적용하는 것이 포인트다.

---

## 문제 2: 4시간 뒤 메모리가 폭발한다

Puppeteer 페이지 하나에서 에셋을 계속 로드하고 교체하다 보면, Three.js의 WebGL 리소스가 쌓인다.
geometry, texture, shader — 이것들이 `dispose()`되지 않으면 GPU 메모리와 CPU 메모리가 계속 늘어난다.

실제로 100개 처리 후에는 괜찮았는데, 500개를 넘어가면서 OOM이 발생했다.

### 잘못된 첫 번째 시도: dispose() 꼼꼼히 호출

```typescript
// 이렇게 해봤는데... Three.js 내부 리소스를 전부 추적하기가 너무 복잡하다
scene.traverse(obj => {
  if (obj.geometry) obj.geometry.dispose()
  if (obj.material) {
    Object.values(obj.material).forEach(v => v?.dispose?.())
    obj.material.dispose()
  }
})
renderer.dispose()
```

Three.js는 내부 WebGL 리소스를 완전히 해제하는 게 까다롭다.
특히 `AnimationMixer`, 내부 캐시, shared geometry 등은 수동 dispose만으로 해결이 안 됐다.

### 해결: 페이지 자체를 주기적으로 교체

```javascript
// batch-convert.js
const PAGE_RESTART_INTERVAL = 20

let page = await launchPage(browser)
await openCaptureApp(page)
let pageProcessed = 0

for (const item of assetQueue) {
  // N개마다 페이지를 닫고 새 페이지 열기
  if (pageProcessed > 0 && pageProcessed % PAGE_RESTART_INTERVAL === 0) {
    await page.close()
    page = await launchPage(browser)
    await openCaptureApp(page)
    pageProcessed = 0
  }

  await captureOne(page, item)
  pageProcessed++
}
```

브라우저 인스턴스는 살려두고, **페이지만 교체**한다.
페이지 재시작은 ~1초 정도 걸리지만, 메모리 누수를 완전히 막는다.

> 20개마다 재시작하는 건 경험적으로 정한 수치다.
> 에셋 크기가 크면 더 자주(10개마다), 작으면 줄여도 된다(30~50개마다).

---

## 문제 3: GPU 없는 서버에서 검은 화면

로컬에서 잘 되던 캡처가 CI 서버나 클라우드 인스턴스에서 검은 화면을 뱉었다.

원인은 Chrome이 GPU를 감지하지 못하면 WebGL 렌더링을 거부하거나, GPU 블록리스트에 걸려 렌더링 품질을 낮추기 때문이다.

### 해결: Chrome 실행 플래그 조정

```javascript
// batch-convert.js
const launchArgs = [
  '--no-sandbox',
  '--disable-setuid-sandbox',
  '--disable-dev-shm-usage',
  '--enable-gpu',
  '--ignore-gpu-blocklist',      // GPU 블록리스트 무시
  '--enable-webgl',
  '--enable-webgl2',
  '--use-gl=angle',
  '--use-angle=d3d11',           // Windows
  // '--use-angle=swiftshader',  // GPU 없는 서버용 (CPU 렌더링 fallback)
  '--enable-gpu-rasterization',
  '--enable-accelerated-video-decode',
]

const browser = await puppeteer.launch({
  headless: 'new',
  args: launchArgs,
  executablePath: findChromePath(),  // 시스템 Chrome 우선 사용
})
```

GPU가 없는 환경에서는 `--use-angle=swiftshader`로 소프트웨어 렌더링을 강제한다.
느리지만 검은 화면보다 낫다.

```javascript
// 환경변수로 제어
const FORCE_SWIFTSHADER = process.env.FORCE_SWIFTSHADER === 'true'

if (FORCE_SWIFTSHADER) {
  launchArgs.push('--use-angle=swiftshader')
}
```

### 시스템 Chrome 찾기

```javascript
function findChromePath() {
  const candidates = [
    process.env.PUPPETEER_EXECUTABLE_PATH,
    '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome', // macOS
    '/usr/bin/google-chrome',   // Linux
    '/usr/bin/chromium',
    'C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe',   // Windows
  ]
  return candidates.find(p => p && fs.existsSync(p))
}
```

---

## 문제 4: 캐릭터가 땅에 박히거나 공중에 뜬다

BVH/FBX 모션을 기본 캐릭터(Manny)에 리타게팅할 때, 루트 모션(pelvis 위치) 처리가 문제가 됐다.

모션 파일마다 기준 골격 비율이 달라서, 그냥 애니메이션을 붙이면 캐릭터가 바닥을 뚫거나 허공에 떠 있었다.
루프 재생 시에는 매 루프마다 위치가 누적되어 점점 멀리 날아가는 현상도 있었다.

### 해결: 프레임 0 pelvis 기준 보정

```typescript
// CaptureApp.tsx
function advanceMotionFrame(delta: number) {
  // 첫 프레임에서 pelvis 기준 위치 저장
  if (!startPelvisPosRef.current) {
    startPelvisPosRef.current = pelvis.getWorldPosition(new THREE.Vector3())
    startPelvisRotRef.current = pelvis.getWorldQuaternion(new THREE.Quaternion())
  }

  mixer.update(delta)

  // 현재 pelvis 위치 - 초기 위치 = 이번 프레임 이동 delta
  const currentPos = pelvis.getWorldPosition(new THREE.Vector3())
  const diff = currentPos.clone().sub(startPelvisPosRef.current!)

  // 모델 전체를 반대 방향으로 보정
  model.position.set(
    -diff.x,
    0,          // Y축(높이)은 항상 0으로 고정
    -diff.z,
  )
}
```

루프 시작 시 `startPelvisPosRef`를 리셋해서 누적 이동이 생기지 않도록 한다.

---

## 결국 안정화는 이렇게 된다

실제 배치 워커의 흐름을 요약하면:

```
1. Puppeteer 페이지 열기 (뷰포트: 1280×820)
2. CaptureApp 로드 (VITE_APP_KIND=capture)
3. window.loadCaptureAsset({ path, fps }) 호출
4. window.captureReady 대기 (최대 90초)
5. Warmup 프레임 N개 버리기 (포맷별)
6. captureNextFrame() 루프:
   - WebP data URL 수집
   - 첫 프레임은 thumbnail.webp로 저장
   - window.isMotionFinished 되면 중단
7. 수집된 WebP 버퍼 → FFmpeg stdin 스트리밍 → output.webm
8. 20개 처리 후 page.close() → 새 페이지
```

이 구조로 6만 개 에셋을 4개 워커로 안정적으로 처리할 수 있었다.

---

## 핵심 파라미터 정리

| 파라미터 | 기본값 | 역할 | 조정 기준 |
|----------|--------|------|-----------|
| `WARMUP_FRAMES` | FBX/BVH: 15, GLB: 2 | T-포즈 안정화 | 첫 프레임 품질 확인 후 조정 |
| `PAGE_RESTART_INTERVAL` | 20 | 메모리 누수 방지 | OOM 발생 시 줄임 |
| `LOAD_TIMEOUT_MS` | 90,000 | 에셋 로드 대기 | 큰 파일일수록 늘림 |
| `MAX_CAPTURE_SECONDS` | 12 | 최대 캡처 길이 | 영상 길이 요구사항에 맞춤 |
| `FORCE_SWIFTSHADER` | false | CPU 렌더링 fallback | GPU 없는 서버에서 true |

---

## 마치며

Puppeteer로 Three.js 앱을 배치 캡처할 때 생기는 문제들은 대부분 **타이밍과 메모리** 문제다.

- 타이밍: 로드 완료 신호를 명시적으로 만들고, Warmup으로 안정화 시간을 확보한다.
- 메모리: dispose를 완벽히 하려 하지 말고, 페이지 자체를 주기적으로 교체한다.

"완벽한 리소스 관리"보다 **"실용적인 재시작"** 이 더 안정적이었다는 것이 이 작업의 교훈이다.

---

*태그: Puppeteer, Three.js, 배치처리, 메모리누수, WebGL, 헤드리스브라우저*
