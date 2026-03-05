# Node HTTP 서버에서 대용량 업로드 스트리밍 처리

이미지·동영상 파일을 **POST body**로 업로드받을 때, body 전체를 메모리에 올린 뒤 처리하면 **용량 제한**과 **메모리 폭증** 위험이 있습니다.  
**스트리밍**으로 받으면서 크기 제한을 두고, 바로 파일로 쓰는 패턴을 정리한 글입니다. 프론트엔드 개발자가 간단한 업로드 API를 만들 때나, 다른 사람에게 공유할 때 그대로 쓸 수 있습니다.

---

## 1. 문제: body를 한 번에 읽을 때

- `req.on('data', chunk => buffers.push(chunk))` 후 `req.on('end', () => { ... Buffer.concat(buffers) })` 로 **전체 body**를 모으면, 업로드 크기가 크면 메모리가 그만큼 늘어납니다.
- **최대 허용 크기**를 두고, 초과 시 **즉시 req.destroy()** 하고 413 등을 반환하는 편이 안전합니다.
- 더 나은 방법은 **스트리밍**: body를 메모리에 다 모으지 않고, **읽는 즉시 파일 스트림으로 pipe** 하는 것입니다.

---

## 2. 최대 크기 제한 + 버퍼로 읽기 (간단한 API용)

- JSON 등 **작은 body**를 다 받은 뒤 파싱할 때는 “총 바이트 수”를 누적하다가 **한도 초과 시 reject** 하는 방식이면 충분합니다.

```js
function readBodyBuffer(req, maxBytes = 5 * 1024 * 1024) {
  return new Promise((resolve, reject) => {
    const chunks = []
    let total = 0
    let settled = false
    const fail = (err) => {
      if (settled) return
      settled = true
      reject(err)
    }
    req.on('data', (chunk) => {
      if (settled) return
      total += chunk.length
      if (total > maxBytes) {
        fail(new Error(`Payload too large: ${maxBytes} bytes limit`))
        req.destroy()
        return
      }
      chunks.push(chunk)
    })
    req.on('end', () => {
      if (settled) return
      settled = true
      resolve(Buffer.concat(chunks))
    })
    req.on('error', fail)
  })
}
```

- **413 Payload Too Large** 를 보낼 때는, 위에서 만든 에러 메시지를 구분해서 statusCode만 413으로 설정하면 됩니다.

---

## 3. 스트리밍: 요청을 파일로 바로 쓰기

- **대용량 파일 업로드**는 **메모리에 올리지 않고** `req` (Readable)를 **파일 쓰기 스트림**으로 pipe 합니다.
- 중간에 **Transform** 스트림으로 “지금까지 읽은 바이트 수”를 세고, **한도 초과 시 에러**를 날리면, 스트림이 중단되고 메모리는 크게 쓰이지 않습니다.

```js
const { pipeline } = require('stream/promises')
const { Transform } = require('stream')
const fs = require('fs')

async function writeRequestBodyToFileStreaming(req, filePath, maxBytes = 128 * 1024 * 1024) {
  const tmpPath = `${filePath}.tmp-${process.pid}-${Date.now()}`
  const limiter = new Transform({
    transform(chunk, _encoding, callback) {
      this.bytesRead = (this.bytesRead || 0) + chunk.length
      if (this.bytesRead > maxBytes) {
        callback(new Error(`Payload too large: ${maxBytes} bytes limit`))
        return
      }
      callback(null, chunk)
    },
  })

  await pipeline(req, limiter, fs.createWriteStream(tmpPath))
  fs.renameSync(tmpPath, filePath)
}
```

- **임시 파일**에 다 쓴 뒤 **rename** 하면, 완료된 파일만 노출되고 중간에 실패하면 임시 파일만 정리하면 됩니다.
- **에러 처리:** pipeline이 reject하면 catch에서 임시 파일 삭제 후 413/500 응답을 보내면 됩니다.

---

## 4. 실전 팁

- **Content-Length:** 클라이언트가 보내면, 한도보다 크면 **요청을 읽기 전에** 413을 반환하고 req.destroy() 할 수 있습니다. (선택.)
- **타임아웃:** 요청이 너무 오래 끌리면 서버 리소스를 잡고 있으므로, `req.setTimeout(ms)` 로 끊는 것도 고려할 만합니다.
- **동시 업로드:** 동시에 많은 대용량 업로드가 들어오면 스트리밍이라도 디스크 I/O가 부하될 수 있으므로, 큐나 동시 처리 개수 제한을 두는 것이 좋습니다.

---

## 요약

| 항목 | 내용 |
|------|------|
| 작은 body | 총 바이트 누적 후 한도 초과 시 destroy + 413. |
| 대용량 body | 메모리에 모으지 말고, Transform으로 크기 제한 두고 pipeline(req, limiter, createWriteStream). |
| 안전 | 임시 파일에 쓰고 완료 시 rename; 실패 시 임시 파일 삭제. |
| 공유 | 프론트엔드가 간단한 업로드 API·도구를 만들 때 그대로 활용 가능. |

스트림의 “쓸 때 버퍼가 차면 기다리기”는 `Node에서 스트림 파이프할 때 백프레셔 처리하기.md`를 참고하면 됩니다.
