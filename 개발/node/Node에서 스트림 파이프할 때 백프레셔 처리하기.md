# Node에서 스트림 파이프할 때 백프레셔 처리하기

자식 프로세스(예: FFmpeg)의 **stdin**에 데이터를 **write**로 넘길 때, 한꺼번에 많이 넣으면 버퍼가 차서 메모리나 데드락에 가까운 상황이 생길 수 있습니다.  
**백프레셔(backpressure)**를 처리하는 방법을 정리한 글입니다. Puppeteer에서 캡처한 프레임을 FFmpeg으로 넘기는 배치 스크립트 등에서 그대로 쓸 수 있고, 다른 사람들에게도 공유하기 좋은 패턴입니다.

---

## 1. 왜 백프레셔가 필요한가?

- **Writable stream**의 `write(chunk)` 는 **버퍼에 여유가 있으면** `true`, **꽉 찼으면** `false` 를 반환합니다.
- `false` 인데 계속 `write` 를 호출하면, 그 데이터는 메모리에 쌓입니다. (고속으로 프레임을 넘길 때 특히 그렇습니다.)
- 자식 프로세스가 stdin을 읽는 속도보다 우리가 쓰는 속도가 빠르면, **버퍼가 한계에 도달**하고, 제대로 처리하지 않으면 에러나 비정상 종료로 이어질 수 있습니다.
- 따라서 **`write` 가 false를 반환하면 “drain” 이벤트를 기다린 뒤** 이어서 쓰는 방식이 안전합니다.

---

## 2. write + drain 패턴

```js
const { spawn } = require('child_process')
const { once } = require('events')

const ffmpeg = spawn('ffmpeg', [...], { stdio: ['pipe', 'ignore', 'pipe'] })

async function writeWithBackpressure(stream, chunk) {
  if (stream.destroyed) throw new Error('stream already closed')
  const ok = stream.write(chunk)
  if (!ok) await once(stream, 'drain')
}

// 사용 예: 프레임 버퍼를 순서대로 넘길 때
for (const frameBuffer of frameBuffers) {
  await writeWithBackpressure(ffmpeg.stdin, frameBuffer)
}
ffmpeg.stdin.end()
```

- **write(chunk)** 가 false이면, **다음 drain 이벤트가 날 때까지** await 합니다.  
  drain은 “버퍼에 공간이 생겼다”는 뜻이므로, 그다음 write를 해도 안전합니다.
- **stream.destroyed** 를 보면, 이미 닫힌 스트림에 쓰는 실수를 막을 수 있습니다.
- **stream.end()** 는 모든 write가 소비된 뒤에 호출하는 것이 좋습니다. (위 예는 for 루프가 끝난 뒤 한 번만 호출.)

---

## 3. 에러 처리

- 자식 프로세스가 **에러로 종료**하면 stdin에 write하다가 예외가 날 수 있습니다.
- **프로세스의 'error' 이벤트**, **stdin의 'error' 이벤트**, 그리고 **프로세스의 'close' (non-zero code)** 를 처리해 두면, “이미 종료된 ffmpeg에 쓰려다 실패” 같은 상황을 깔끔하게 잡을 수 있습니다.
- `writeWithBackpressure` 안에서 `stream.destroyed` 체크를 하는 것만으로도, 상위에서 프로세스를 kill한 뒤 실수로 다시 write하는 경우를 줄일 수 있습니다.

---

## 4. 실전 팁

- **한 번에 하나의 chunk만** 보내고 drain을 기다리는 방식이면, 대부분의 “프레임 시퀀스 → FFmpeg stdin” 파이프라인에 충분합니다.
- **병렬로 여러 스트림**에 쓸 때는 Promise.all + 각 스트림별 write/drain을 조합하면 됩니다.
- **stream.pipeline** (Node 내장)을 쓰면, Readable → Writable 구간에서 백프레셔를 자동으로 처리해 줍니다.  
  다만 “우리가 버퍼를 하나씩 만들어서 넘기는” 구조라면, 위처럼 수동으로 write + drain을 다루는 편이 직관적일 수 있습니다.

---

## 요약

| 항목 | 내용 |
|------|------|
| 문제 | write가 버퍼가 꽉 차면 false 반환. 무시하고 쓰면 메모리·불안정. |
| 해결 | write 반환값이 false이면 `once(stream, 'drain')` 후 다음 write. |
| 안전 | stream.destroyed 체크, 자식 프로세스 error/close 처리. |
| 활용 | FFmpeg stdin, 그 외 자식 프로세스에 순차적으로 데이터를 넘길 때. |

FFmpeg으로 WebP 프레임을 넘기는 전체 파이프라인은 `../web/FFmpeg으로 WebP 프레임을 WebM·GIF로 인코딩하기.md`를 참고하면 됩니다.
