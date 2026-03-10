# 브라우저를 뒤집다 — Three.js 뷰어를 6만 건 배치 엔진으로 만든 방법

> **시리즈:** 3D 에셋 배치 파이프라인 구축기 (1/3)
> **대상 독자:** Three.js를 써본 프론트엔드 개발자, 브라우저 자동화에 관심 있는 엔지니어

---

## 문제: 6만 개 모션 에셋, 미리보기가 없다

VARCO 애니메이션 플랫폼에는 캐릭터 모션 클립이 약 6만 개 있다.
유저가 "걷기", "달리기", "춤" 같은 모션을 고를 때, 파일 이름만 보고 선택해야 한다면?

미리보기가 없는 에셋 라이브러리는 쓸모가 없다.
각 모션의 **WebP 썸네일 + WebM 루프 영상**을 자동으로 만들어야 했다.

6만 개를. 자동으로.

---

## 기존 환경: Three.js 앱밖에 없다

문제는 3D 렌더링 인프라가 **브라우저 기반 Three.js 앱**으로만 존재한다는 것이었다.

- 서버 사이드 3D 렌더러 없음
- Blender CLI 스크립트 없음
- 별도 렌더팜 없음

처음엔 "Python + Blender headless로 만들까?"를 고민했다.
하지만 기존 Three.js 뷰어가 이미 모든 포맷(GLB/FBX/BVH)을 처리하고, 리타게팅 로직까지 완성되어 있었다.

**"이미 동작하는 브라우저 앱을 그대로 쓰되, 자동화 파이프라인 안에 집어넣으면 어떨까?"**

이것이 이 프로젝트의 핵심 아이디어다.

---

## 해결책: Puppeteer로 브라우저를 제어한다

Puppeteer는 Node.js에서 Chrome을 헤드리스로 제어하는 라이브러리다.
보통 E2E 테스트나 웹 스크래핑에 쓰지만, 여기서는 **3D 렌더링 워커**로 사용했다.

전체 구조는 이렇다.

```
Vite Dev Server (Three.js 앱)
       ▲ HTTP
       │
Puppeteer (Node.js)
  └─ page.evaluate() → window.captureNextFrame()
       │ WebP base64
       ▼
FFmpeg (stdin 스트리밍)
       │ VP9 인코딩
       ▼
output.webm + thumbnail.webp
```

---

## 핵심: Capture 모드를 분리한다

기존 뷰어 앱을 두 가지 모드로 나눴다.

```typescript
// main.tsx
const APP_KIND = import.meta.env.VITE_APP_KIND

if (APP_KIND === 'capture') {
  createRoot(root).render(<CaptureApp />)
} else {
  createRoot(root).render(<App />)   // 기존 UI 뷰어
}
```

**CaptureApp**은 UI가 없다. 대신 `window` 전역에 API를 노출한다.

```typescript
// CaptureApp.tsx — Puppeteer와의 계약
useEffect(() => {
  window.loadCaptureAsset = async ({ path, fps }) => {
    await loadFile(path, fps)
    window.captureReady = true
  }

  window.captureNextFrame = async ({ mimeType, quality }) => {
    advanceMotionFrame(1 / fps)
    renderCurrentScene()

    const canvas = rendererRef.current!.domElement
    const dataUrl = encodeCompositeFrame(canvas, { mimeType, quality })
    const done = window.isMotionFinished

    return { dataUrl, done }
  }
}, [])
```

Puppeteer 쪽은 이걸 이렇게 호출한다.

```javascript
// batch-convert.js
// 1. 에셋 로드
await page.evaluate(({ path, fps }) => {
  window.loadCaptureAsset({ path, fps })
}, { path: 'motions/dance.glb', fps: 15 })

// 2. 로드 완료 대기
await page.waitForFunction(
  () => window.captureReady || window.captureError,
  { timeout: 90_000 }
)

// 3. 프레임 캡처 루프
while (frameCount < MAX_FRAMES) {
  const { dataUrl, done } = await page.evaluate(() =>
    window.captureNextFrame({ mimeType: 'image/webp', quality: 1.0 })
  )
  frames.push(dataUrl)
  frameCount++

  if (done) break
}
```

Three.js 씬 내부를 Node.js에서 직접 다룰 필요가 없다.
**브라우저와 스크립트 사이의 계약(window API)만 명확하면 된다.**

---

## Vite를 "배치 서버"로 쓰는 이유

처음엔 "Vite dev 서버를 배치에 쓰는 게 맞나?"라는 의문이 있었다.

결론적으로 세 가지 이유로 최선의 선택이었다.

**① 기존 Three.js 코드를 그대로 재사용**
FBX/GLB/BVH 로더, 리타게팅 로직, 애니메이션 믹서 — 이미 동작하는 코드를 처음부터 Node.js용으로 다시 만들 필요가 없었다.

**② 커스텀 미들웨어로 로컬 에셋 서빙**
S3에서 내려받은 에셋 파일을 HTTP로 서빙해야 했는데, Vite에 미들웨어 플러그인 하나로 해결됐다.

```javascript
// vite-asset-dir-plugin.js
export default function assetDirPlugin(assetDir) {
  return {
    name: 'asset-dir',
    configureServer(server) {
      server.middlewares.use('/asset', (req, res, next) => {
        const safePath = validatePath(req.url)  // path traversal 방지
        const fullPath = path.join(assetDir, safePath)

        if (!fs.existsSync(fullPath)) { res.statusCode = 404; return res.end() }

        res.setHeader('Content-Type', getMimeType(fullPath))
        res.setHeader('Cache-Control', 'no-store')  // 캐시 방지
        fs.createReadStream(fullPath).pipe(res)
      })
    }
  }
}
```

**③ 개발 중 HMR로 즉시 확인**
캡처 로직을 바꾸면 브라우저에서 바로 결과를 볼 수 있다.

---

## 병렬화: stride 분배

워커 4개를 돌릴 때, 단순히 "앞에서 1/4씩 나누면" 안 된다.
에셋 크기가 파일마다 달라서 특정 워커만 오래 걸린다.

**stride 방식**을 사용하면 에셋이 고르게 분배된다.

```javascript
// 워커 index=1, stride=4 일 때
// [0, 1, 2, 3, 4, 5, 6, 7, 8, ...]
//        ↑           ↑           ↑
// → assets[1, 5, 9, 13, ...]
const myAssets = allAssets.filter((_, i) => i % stride === slot)
```

실행:
```bash
# 워커 4개, 에셋 200개
pnpm run batch:start -- --stride 4 --limit 200
```

`start-batch.js`가 Vite 서버를 띄운 뒤 워커 4개를 별도 프로세스로 spawn한다.
각 워커는 독립적으로 Puppeteer 인스턴스를 가지고 병렬 실행된다.

---

## 결과

| 구분 | 수치 |
|------|------|
| 처리 속도 (단일) | ~5.45초/에셋 |
| 100개 실측 | 9분 5초 |
| stride 4 기준 | 약 1/4로 단축 |
| 6만 건 예상 | ~1일 (stride 4 기준) |
| 출력 포맷 | WebP 썸네일 + WebM 프리뷰 |

기존에 존재하는 Three.js 앱을 Puppeteer로 감싸고, FFmpeg으로 인코딩하는 구조로 **처음부터 새 렌더러를 만들지 않고** 목표를 달성했다.

---

## 마치며

이 프로젝트에서 얻은 가장 큰 인사이트는:

> **"브라우저 앱"과 "자동화 파이프라인"은 서로 다른 세계가 아니다.**

`window` API를 계약으로 삼으면, 브라우저 앱을 Node.js 자동화 파이프라인의 워커로 만들 수 있다.
Three.js, WebGL, canvas — 브라우저에서만 된다고 생각했던 것들이 전부 배치 처리에 쓰일 수 있다.

다음 글에서는 이 파이프라인에서 **Warmup 프레임, 메모리 누수, GPU-less 서버 환경** 등 실제로 부딪힌 문제와 해결법을 다룬다.

---

*태그: Three.js, Puppeteer, WebGL, 배치처리, 프론트엔드, Node.js, FFmpeg*
