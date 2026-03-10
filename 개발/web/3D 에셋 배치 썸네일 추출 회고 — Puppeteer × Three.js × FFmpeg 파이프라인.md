# 3D 에셋 배치 썸네일 추출 회고 — Puppeteer × Three.js × FFmpeg 파이프라인

> 수만 건의 3D 모션 에셋(GLB/FBX/BVH)에서 WebP 썸네일과 WebM 프리뷰 영상을 자동 생성하는 배치 시스템을 설계·구현한 경험을 정리한 글이다.

---

## 왜 이 작업이 필요했나

VARCO 애니메이션 플랫폼에는 캐릭터 모션 클립이 수만 개 존재한다.
유저가 모션을 고를 때 정적인 파일 이름만 보여주는 것은 UX상 최악이다.
**각 모션이 어떻게 생겼는지 미리보기(WebP 썸네일 + WebM 루프 영상)**를 제공해야 했다.

문제는 이걸 **수작업으로 만들 수 없다**는 것이다.

- 에셋 수: 수만 개 (초기 목표 ~60,000건)
- 포맷 다양성: GLB / FBX / BVH 혼재
- 기존 인프라: 3D 뷰어는 **브라우저 기반 Three.js 앱**으로만 존재

즉, "브라우저에서 돌아가는 3D 뷰어를 자동으로 굴려서 대량으로 캡처하는 시스템"을 처음부터 만들어야 했다.

---

## 시스템 구조 한눈에 보기

```
S3 (varco-animation-res)
      │
      ▼ download-clips.js (AWS SDK, 병렬 20개)
data/asset/          ← GLB / FBX / BVH 원본
      │
      ▼ start-batch.js (Vite 서버 기동 + 워커 N개 spawn)
┌─────────────────────────────────┐
│  Vite Dev Server :5173          │
│  CaptureApp.tsx (Three.js)      │ ← 캡처 전용 모드
│  /asset/* → data/asset/ 서빙   │
└─────────────────────────────────┘
      ▲ (HTTP)
      │  × N 병렬 워커
      ▼
batch-convert.js (Puppeteer)
  ├─ 에셋 로드 대기 (window.captureReady)
  ├─ Warmup 프레임 버림
  ├─ window.captureNextFrame() 호출 반복
  │      └─ Three.js 프레임 렌더 → canvas.toDataURL('image/webp')
  ├─ WebP base64 → Buffer → FFmpeg stdin 스트리밍
  ├─ FFmpeg: libvpx-vp9, CRF 18 → .webm 저장
  └─ 첫 프레임 → .webp 저장
      │
      ▼ upload-clips.js
S3 (web-dataset-previews/)
```

---

## 핵심 설계 결정 3가지

### 1. "브라우저를 뒤집어서" 배치 엔진으로 쓴다

기존 Three.js 뷰어를 **두 가지 모드**로 분리했다.

| 모드 | 파일 | 역할 |
|------|------|------|
| Display | `App.tsx` | 유저가 브라우저에서 보는 일반 뷰어 |
| Capture | `CaptureApp.tsx` | Puppeteer가 자동으로 제어하는 캡처 전용 앱 |

Capture 모드에서는 `window` 전역에 API를 노출한다.

```typescript
window.captureNextFrame  // 한 프레임 렌더 → WebP data URL 반환
window.loadCaptureAsset  // 에셋 로드 요청
window.captureReady      // 로드 완료 신호
window.captureError      // 로드 실패 신호
window.isMotionFinished  // 애니메이션 종료 신호
```

Puppeteer는 `page.evaluate()`로 이 API를 호출하기만 하면 된다.
Three.js 씬을 직접 다룰 필요가 없고, **"브라우저와 스크립트 간의 계약"이 명확**하다.

---

### 2. 워커 stride 분배로 병렬화

단순히 `N개씩 잘라서` 워커에 나눠주면, 에셋 크기 편차 때문에 특정 워커만 오래 걸린다.
대신 **stride 방식**을 사용했다.

```
에셋 목록: [0, 1, 2, 3, 4, 5, 6, 7, ...]
stride = 4 일 때:
  Worker 0 (slot 0): 0, 4, 8, 12, ...
  Worker 1 (slot 1): 1, 5, 9, 13, ...
  Worker 2 (slot 2): 2, 6, 10, 14, ...
  Worker 3 (slot 3): 3, 7, 11, 15, ...
```

크고 작은 에셋이 고루 섞이기 때문에 **워커 간 처리 시간이 균등화**된다.
실행 명령 예시:

```bash
pnpm run batch:start -- --stride 4 --offset 0 --limit 200
```

---

### 3. FFmpeg 직접 스트리밍으로 디스크 I/O 최소화

처음엔 WebP 프레임을 파일로 저장했다가 FFmpeg에 넘기는 방식을 생각했다.
하지만 수만 건 × 수십 프레임 = **수백만 개 임시 파일**이 생기는 문제가 있다.

해결책: Puppeteer에서 받은 base64 data URL을 **Buffer로 디코딩한 뒤 FFmpeg stdin으로 직접 파이프**한다.

```javascript
// base64 → Buffer
const buf = Buffer.from(dataUrl.split(',')[1], 'base64')

// FFmpeg stdin 스트리밍
ffmpegProcess.stdin.write(buf)

// 모든 프레임 완료 후
ffmpegProcess.stdin.end()
```

FFmpeg 설정:
```bash
ffmpeg -f image2pipe -vcodec webp -framerate 15 -i pipe:0
       -c:v libvpx-vp9 -crf 18 -b:v 0 -pix_fmt yuv420p
       -cpu-used 4 -row-mt 1 -threads 8
       output.webm
```

임시 파일 없이 메모리에서 바로 인코딩까지 완료된다.

---

## 부딪혔던 문제들

### 문제 1: 첫 프레임이 T-포즈

에셋 로드가 완료(`captureReady = true`)되어도 캐릭터가 아직 T-포즈인 경우가 있다.
Three.js의 `AnimationMixer`가 첫 `update()` 이후부터 포즈가 적용되기 때문이다.

**해결:** 포맷별 Warmup 프레임 적용

```javascript
const WARMUP_FRAMES = {
  fbx: 15,
  bvh: 15,
  zip: 15,
  glb: 2,   // GLB는 상대적으로 빨리 안정됨
}
```

Warmup 프레임은 캡처하되 FFmpeg에 보내지 않고 버린다.

---

### 문제 2: 장시간 실행 시 메모리 증가

Puppeteer 페이지 하나에서 에셋을 계속 교체하면 Three.js WebGL 컨텍스트 리소스가 누적된다.
20~30개 처리 후 메모리가 눈에 띄게 증가했다.

**해결:** 일정 간격으로 페이지 재시작

```javascript
const PAGE_RESTART_INTERVAL = 20

if (pageProcessed > 0 && pageProcessed % PAGE_RESTART_INTERVAL === 0) {
  await page.close()
  page = await launchPage(browser)  // 새 페이지
  await openCaptureApp(page)
  pageProcessed = 0
}
```

브라우저는 살려두고 페이지만 교체하므로 재시작 오버헤드가 작다.

---

### 문제 3: GPU 없는 서버에서의 렌더링 품질

CI/배포 서버 환경은 GPU가 없다.
Chrome은 기본적으로 GPU를 감지 못하면 렌더링을 거부하거나 품질을 낮춘다.

**해결:** SwiftShader 플래그 + ANGLE 설정

```javascript
launchArgs = [
  '--enable-gpu',
  '--ignore-gpu-blocklist',
  '--enable-webgl',
  '--use-gl=angle',
  '--use-angle=swiftshader',  // CPU 소프트웨어 렌더링
]
```

SwiftShader는 느리지만, 렌더링 자체는 정상적으로 된다.
GPU 있는 환경에서는 `--use-angle=d3d11`(Windows) 또는 기본 ANGLE을 사용한다.

---

### 문제 4: Pelvis sliding — 캐릭터가 공중에 뜨거나 땅에 박힘

BVH/FBX 모션을 Manny 기본 캐릭터에 리타게팅할 때, 루트 모션(pelvis 위치) 처리가 까다롭다.
모션마다 골격 비율이 달라서 그냥 붙이면 캐릭터가 땅 위에 제대로 서 있지 않는다.

**해결:** 프레임 0의 pelvis 위치를 기준으로 매 프레임 보정

```typescript
// 첫 프레임 기준 저장
startPelvisPosRef.current = pelvis.getWorldPosition(new THREE.Vector3())

// 매 프레임 보정
const currentPos = pelvis.getWorldPosition(new THREE.Vector3())
const delta = currentPos.sub(startPelvisPosRef.current)
model.position.sub(delta)
```

루프 재생 시 누적 이동이 생기지 않도록 `repeatCountRef`로 관리한다.

---

## 실측 성능

| 구분 | 수치 |
|------|------|
| 처리 속도 | ~5.45초/에셋 |
| 100개 실측 | 9분 5초 |
| 60,000개 단순 환산 | 90.8시간 (약 3.8일) |
| stride 4 기준 예상 | ~1일 |
| 주요 병목 | VP9 소프트웨어 인코딩 (CPU) |

stride로 워커 4개 병렬 실행 시 약 1/4로 단축된다.
GPU 인코딩을 쓸 수 있는 환경에서는 VP9 인코딩 병목이 사라져 2~3배 더 빠르다.

---

## 운영 설계 — 안정성이 성능보다 먼저다

배치 시스템에서 가장 중요한 교훈은 **"중단 복구를 먼저 설계하라"**는 것이다.

```javascript
// 이미 출력 파일이 있으면 건너뜀
if (SKIP_EXISTING && fs.existsSync(webpPath) && fs.existsSync(webmPath)) {
  log(`[${index}] skip ${item.name}`)
  continue
}
```

`SKIP_EXISTING` 옵션 덕분에:
- 배치가 중간에 실패해도 처음부터 다시 돌릴 필요 없다.
- 특정 구간만 `--offset / --limit`으로 재실행할 수 있다.
- 실패 로그를 JSON으로 저장해서 실패 목록만 별도로 재처리한다.

---

## Vite를 배치 서버로 쓰는 이유

일반적인 배치 시스템에서 Vite dev 서버를 띄우는 것은 낯설다.
그러나 이 구조에는 이유가 있다.

1. **Three.js 앱의 재사용**: 이미 동작하는 뷰어 코드를 그대로 활용
2. **HMR 개발 편의성**: 캡처 로직을 바꿀 때 브라우저에서 바로 확인
3. **`/asset/*` 미들웨어 플러그인**: Vite에 커스텀 미들웨어를 붙여 로컬 디렉토리를 HTTP로 서빙

```javascript
// vite-asset-dir-plugin.js
server.middlewares.use('/asset', (req, res, next) => {
  const safePath = validatePath(req.url)  // path traversal 방지
  const fullPath = path.join(ASSET_DIR, safePath)

  res.setHeader('Content-Type', getMimeType(fullPath))
  res.setHeader('Cache-Control', 'no-store')
  fs.createReadStream(fullPath).pipe(res)
})
```

`Cache-Control: no-store`는 배치 중 같은 에셋이 다른 내용으로 로드될 때 캐시 문제를 방지한다.

---

## 프론트엔드 엔지니어로서 배운 것

### 브라우저는 도구이지 제약이 아니다

`브라우저에서만 돌아가는 Three.js 앱`을 Puppeteer로 제어하면서, 브라우저 환경을 서버 사이드 자동화 파이프라인에 통합할 수 있다는 것을 직접 경험했다.
`canvas.toDataURL()`, `window` API 노출, `page.evaluate()` 조합은 Node.js와 브라우저 환경의 경계를 유연하게 허문다.

### 비동기 API 계약 설계의 중요성

`captureReady`, `captureError`, `isMotionFinished` 같은 명시적 플래그가 없었다면
Puppeteer 쪽에서 "언제 캡처해도 되는지"를 알 수 없다.
**준비 상태를 외부에서 관찰할 수 있는 인터페이스**를 의도적으로 설계해야 한다.

### 숫자로 관리하는 품질-성능 트레이드오프

| 파라미터 | 역할 | 실험 범위 |
|----------|------|-----------|
| `warmupFrames` | T-포즈 안정화 | 2 ~ 20 |
| `fps` | 영상 부드러움 vs 처리량 | 10 ~ 30 |
| `crf` | VP9 화질 vs 파일 크기 | 15 ~ 28 |
| `cpu-used` | 인코딩 속도 vs 화질 | 1 ~ 5 |
| `PAGE_RESTART_INTERVAL` | 메모리 vs 재시작 오버헤드 | 10 ~ 50 |

체감이 아니라 벤치마크 수치로 결정했다.

### 운영 문제는 기술 문제와 다르다

코드가 완성된 뒤에 "이걸 60,000개에 돌리면 며칠 걸려?"가 새로운 문제가 됐다.
단일 머신 성능 최적화보다 **병렬 분할, 재개 가능 구조, 로그 관리**가 훨씬 더 중요하다는 것을 배웠다.

---

## 이 작업이 프론트엔드 커리어에서 의미하는 것

이 프로젝트는 흔히 "프론트엔드"라고 하면 떠올리는 UI 컴포넌트 개발과는 결이 다르다.

- **Three.js WebGL 씬 제어**: 카메라, 조명, 애니메이션 믹서
- **Node.js 배치 시스템 설계**: 프로세스 관리, 워커 병렬화
- **Puppeteer 자동화**: 헤드리스 브라우저 제어, window API 설계
- **FFmpeg 파이프라인 통합**: 코덱 선택, stdin 스트리밍
- **AWS S3 연동**: 대량 다운로드/업로드, 재시도 로직
- **운영 안정성 설계**: skip-existing, 실패 복구, 로그

이 스택을 다룰 줄 안다는 것은 **"브라우저 앱"과 "서버 자동화" 사이의 경계를 직접 설계하고 구현할 수 있다**는 것을 의미한다.
단순 UI 개발자와 차별화되는 **"전체 파이프라인을 아는 프론트엔드"** 포지션을 만들어준다.

---

## 관련 문서

- `Puppeteer 헤드리스 브라우저로 배치 캡처하기.md`
- `Three.js로 3D 에셋 웹에서 렌더링하기.md`
- `웹에서 캔버스 프레임을 이미지로 캡처하기.md`
- `FFmpeg으로 WebP 프레임을 WebM·GIF로 인코딩하기.md`
- `WebM·MP4·코덱 라이선스와 GPU 인코딩 정리.md`
- `3D 에셋 웹 파이프라인 전체 회고.md`
