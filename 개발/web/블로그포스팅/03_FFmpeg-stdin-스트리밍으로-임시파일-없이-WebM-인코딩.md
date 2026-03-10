# FFmpeg stdin 스트리밍으로 임시 파일 없이 WebM 인코딩하기

> **시리즈:** 3D 에셋 배치 파이프라인 구축기 (3/3)
> **대상 독자:** Node.js에서 FFmpeg를 쓰는 개발자, 대용량 파일 처리 최적화에 관심 있는 개발자

---

## 문제: 6만 건 × 수십 프레임 = 수백만 개 임시 파일?

Puppeteer가 캡처한 WebP 프레임을 FFmpeg으로 인코딩하는 방법은 두 가지다.

**방법 A: 파일로 저장 후 FFmpeg에 경로 전달**
```
frame_001.webp, frame_002.webp, ... frame_180.webp
  ↓
ffmpeg -i "frame_%03d.webp" output.webm
  ↓
임시 파일 삭제
```

**방법 B: 메모리에서 FFmpeg stdin으로 직접 스트리밍**
```
WebP buffer (메모리) → ffmpeg stdin → output.webm
```

방법 A의 문제: 에셋 하나당 ~180프레임이면, 6만 건 기준 **최대 1,080만 개 임시 파일**이 생긴다.
병렬 4개 워커가 동시에 돌면 디스크 I/O가 폭발한다.

---

## stdin 스트리밍 구현

FFmpeg는 `-i pipe:0`으로 stdin에서 입력을 받는다.
Node.js의 `child_process.spawn`으로 FFmpeg를 실행하고, stdin에 버퍼를 쓰면 된다.

```javascript
// batch-convert.js
function encodeFramesToWebM(frameBuffers, outputPath, fps) {
  return new Promise((resolve, reject) => {
    const ffmpegArgs = [
      '-y',
      '-f', 'image2pipe',     // stdin에서 이미지 시퀀스 읽기
      '-vcodec', 'webp',      // 입력 코덱: WebP
      '-framerate', String(fps),
      '-i', 'pipe:0',         // stdin을 입력으로

      '-an',                  // 오디오 없음
      '-c:v', 'libvpx-vp9',
      '-pix_fmt', 'yuv420p',
      '-crf', '18',
      '-b:v', '0',
      '-deadline', 'good',
      '-cpu-used', '4',
      '-row-mt', '1',
      '-threads', '8',

      outputPath,
    ]

    const ffmpeg = spawn('ffmpeg', ffmpegArgs, { stdio: ['pipe', 'pipe', 'pipe'] })

    ffmpeg.on('close', code => {
      if (code === 0) resolve()
      else reject(new Error(`FFmpeg exited with code ${code}`))
    })

    ffmpeg.stderr.on('data', data => {
      // FFmpeg 진행 상황 로깅 (선택)
      // process.stderr.write(data)
    })

    // WebP 버퍼를 stdin으로 순서대로 쓰기
    for (const buf of frameBuffers) {
      ffmpeg.stdin.write(buf)
    }
    ffmpeg.stdin.end()  // 스트림 종료 → FFmpeg가 인코딩 완료 후 종료
  })
}
```

Puppeteer에서 받은 data URL을 Buffer로 변환:

```javascript
// base64 data URL → Buffer
function dataUrlToBuffer(dataUrl) {
  // "data:image/webp;base64,/9j/..." 에서 base64 부분만 추출
  const base64 = dataUrl.split(',')[1]
  return Buffer.from(base64, 'base64')
}

// 사용
const frameBuffers = frames.map(dataUrlToBuffer)
await encodeFramesToWebM(frameBuffers, outputPath, fps)
```

---

## FFmpeg 옵션 선택 이유

### VP9를 선택한 이유

처음엔 H.264(libx264)로 시작했다. 인코딩이 빠르고 호환성도 좋다.
그런데 상용 서비스에서 H.264를 쓰려면 **MPEG-LA 특허 라이선스** 문제가 생긴다.

VP9(WebM)은 Google이 개발한 오픈 코덱이라 라이선스 리스크가 훨씬 낮다.
사내 서비스라도 상용 목적이라면 VP9/AV1이 안전하다.

```
H.264(MP4) → MPEG-LA 특허 → 상용 서비스 법무 검토 필요
VP9(WebM)  → 오픈 코덱   → 라이선스 리스크 낮음
```

### CRF 18의 의미

VP9의 CRF(Constant Rate Factor)는 낮을수록 고화질이다.

```
CRF 0  = 무손실 (파일 크기 ↑↑)
CRF 18 = 고화질 (사람 눈에 손실 거의 안 보임)
CRF 33 = 중간 화질
CRF 63 = 최저 화질 (파일 크기 ↓↓)
```

`-b:v 0`은 비트레이트를 무시하고 CRF 기준으로만 품질을 제어하겠다는 의미다.
이 조합이 VP9에서 CRF 모드를 올바르게 활성화한다.

### cpu-used 4의 의미

VP9의 `cpu-used`는 인코딩 속도와 품질의 트레이드오프다.

```
1 = 최고 화질, 가장 느림
4 = 균형점 (실무에서 일반적 선택)
5 = 가장 빠름, 품질 약간 저하
```

배치 처리에서는 `4`가 속도와 품질의 균형이 좋았다.

### deadline good

`-deadline`은 인코딩 전략이다.

```
best     = 최고 압축률, 매우 느림 (오프라인 아카이빙용)
good     = 균형 (실무 기본값)
realtime = 가장 빠름, 압축률 낮음 (실시간 스트리밍용)
```

---

## 첫 프레임을 WebP 썸네일로도 저장

WebM 파일 외에 정적 썸네일(WebP)도 필요하다.
첫 프레임 버퍼를 그냥 파일로 쓰면 된다.

```javascript
async function captureAndEncode(page, item) {
  // ... warmup 생략 ...

  const frameBuffers = []
  let thumbnailSaved = false

  while (frameBuffers.length < MAX_FRAMES) {
    const { dataUrl, done } = await page.evaluate(() =>
      window.captureNextFrame({ mimeType: 'image/webp', quality: 1.0 })
    )

    const buf = dataUrlToBuffer(dataUrl)
    frameBuffers.push(buf)

    // 첫 프레임 → WebP 썸네일
    if (!thumbnailSaved) {
      await fs.promises.writeFile(item.webpPath, buf)
      thumbnailSaved = true
    }

    if (done) break
  }

  // 전체 프레임 → WebM
  await encodeFramesToWebM(frameBuffers, item.webmPath, item.fps)
}
```

FFmpeg를 두 번 돌릴 필요 없이, 첫 번째 프레임 버퍼만 따로 저장하면 된다.

---

## 실패 시 partial output 정리

인코딩 중 실패하면 불완전한 webm 파일이 남는다.
다음 실행에서 `SKIP_EXISTING=true`를 쓰면 이 파일을 완료로 착각할 수 있다.

```javascript
async function captureAndEncode(page, item) {
  try {
    // ... 캡처 + 인코딩 ...
  } catch (err) {
    // 실패 시 partial output 삭제
    await fs.promises.rm(item.webmPath, { force: true })
    // webp는 남겨둬도 됨 (다음 실행에서 webm만 재처리)
    throw err
  }
}
```

```javascript
// skip-existing: webp AND webm 둘 다 있어야 완료로 인정
if (
  SKIP_EXISTING &&
  fs.existsSync(item.webpPath) &&
  fs.existsSync(item.webmPath)
) {
  log(`[skip] ${item.name}`)
  continue
}
```

---

## 전체 성능 수치

100개 에셋 기준 실측:

| 항목 | 수치 |
|------|------|
| 전체 처리 시간 | 9분 5초 |
| 에셋당 평균 | 5.45초 |
| VP9 인코딩 비중 | ~40% |
| Puppeteer 캡처 비중 | ~45% |
| 기타 (로드, I/O) | ~15% |

VP9 소프트웨어 인코딩이 병목의 절반 가까이를 차지한다.
GPU VP9 인코딩(`libvpx-vp9` + GPU)을 쓸 수 있다면 이 부분을 크게 줄일 수 있다.

---

## 마치며

Node.js에서 FFmpeg stdin 스트리밍은 임시 파일 없이 대용량 비디오 인코딩을 처리하는 패턴이다.
Puppeteer의 `canvas.toDataURL()`과 조합하면:

```
브라우저 캔버스 → WebP base64 → Buffer → FFmpeg stdin → WebM
```

이 파이프라인은 디스크를 거치지 않아 I/O 병목이 없고, 실패 시 partial output을 깔끔하게 처리할 수 있다.

3D 에셋 외에도 **차트 캡처, 이미지 시퀀스 인코딩, 애니메이션 GIF 생성** 등 같은 패턴으로 응용할 수 있다.

---

*태그: FFmpeg, Node.js, Puppeteer, VP9, WebM, 비디오인코딩, stdin스트리밍*
