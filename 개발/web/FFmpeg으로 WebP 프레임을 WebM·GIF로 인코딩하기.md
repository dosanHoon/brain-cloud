# FFmpeg으로 WebP 프레임을 WebM·GIF로 인코딩하기

웹에서 캔버스로 찍은 **WebP 프레임 시퀀스**를 영상 파일로 만들 때, **WebM(VP9)** 과 **GIF** 각각 어떤 옵션을 쓰면 좋은지 정리한 글입니다.  
배치 변환·API 서버에서 실제로 썼던 파이프라인을 기준으로 합니다.

---

## 1. WebP 시퀀스 → WebM (VP9)

### 1.1 파이프 입력

- 브라우저에서 받은 **Base64 디코딩 WebP 버퍼**를 **stdin**으로 FFmpeg에 넘기는 방식이 메모리와 디스크 I/O를 아끼는 데 유리합니다.
- `-f image2pipe -vcodec webp -framerate <fps> -i pipe:0` 으로 “WebP 이미지 스트림”을 입력으로 받습니다.
- Node에서는 `ffmpeg.stdin.write(buffer)` 후 `ffmpeg.stdin.end()` 로 스트림을 닫고, `write` 반환값이 false일 때 `drain` 이벤트를 기다려서 백프레셔를 처리하는 것이 안전합니다.

### 1.2 코덱·품질 옵션

| 옵션 | 의미 | 실무 예 |
|------|------|---------|
| `-c:v libvpx-vp9` | VP9 코덱 사용 | WebM 표준 조합 |
| `-crf` | 품질 (낮을수록 고품질, 0–63) | 18 정도면 시각적으로 충분한 경우 많음 |
| `-b:v 0` | CRF 모드 사용 시 필수 (비트레이트 무시) | CRF와 함께 사용 |
| `-deadline` | 인코딩 속도/품질 트레이드오프 | `good` (기본), `realtime` 등 |
| `-cpu-used` | 속도 (0=느리지만 고품질, 5=빠름) | 1~2 정도로 균형 |
| `-row-mt 1` | 멀티스레드 row 단위 처리 | 권장 |
| `-threads` | 스레드 수 | 4 등 |

### 1.3 픽셀 포맷 (yuv444p vs yuv420p)

- **yuv420p:** 일반적인 영상용. 용량 작고 호환성 좋음. 색차 서브샘플링으로 색상 해상도가 밝기보다 낮음.
- **yuv444p:** Y·U·V 전부 동일 해상도. 선명한 텍스트·그래픽에 유리하지만 파일이 더 커짐.
- 3D 캡처·UI 프리뷰처럼 선명도가 중요하면 **yuv444p**를, 용량과 호환성을 우선하면 **yuv420p**를 선택하면 됩니다.

```bash
# 예시 (의미만)
ffmpeg -y -f image2pipe -vcodec webp -framerate 15 -i pipe:0 \
  -an -c:v libvpx-vp9 -pix_fmt yuv444p -crf 18 -b:v 0 \
  -deadline good -cpu-used 1 -row-mt 1 -threads 4 \
  output.webm
```

---

## 2. WebM → GIF

- GIF는 **256색 팔레트** 제한이 있어서, 그냥 변환하면 색이 깨져 보일 수 있습니다.
- **palettegen** + **paletteuse** 필터를 쓰면, 원본 프레임에서 팔레트를 생성한 뒤 그 팔레트로 다시 색을 넣어서 품질이 좋아집니다.

### 2.1 두 번에 나눠서 하기 (고품질)

1. **1패스:** WebM → palette.png 생성  
   `-vf "fps=15,scale=600:-1:flags=lanczos,palettegen"`
2. **2패스:** WebM + palette.png → GIF  
   `-vf "fps=15,scale=600:-1:flags=lanczos,paletteuse=dither=bayer"`

### 2.2 한 번에 하기 (split + palettegen + paletteuse)

- `split[s0][s1]; [s0]palettegen[p]; [s1][p]paletteuse` 패턴으로, 같은 입력을 두 갈래로 써서 한 번에 GIF로 만들 수 있습니다.
- fps·scale은 용량과 품질에 맞게 조절합니다. (예: fps=15, scale=600:-1)

```bash
# 예시 (의미만)
ffmpeg -y -i input.webm \
  -vf "fps=15,scale=600:-1:flags=lanczos,split[s0][s1];[s0]palettegen[p];[s1][p]paletteuse" \
  output.gif
```

---

## 3. WebM → WebP (정적·애니메이션)

- **정적 WebP:** WebM의 첫 프레임만 추출해 이미지로 저장할 때는, 예를 들어 `-vframes 1` 로 한 장만 뽑고 `-c:v libwebp` 등으로 내보내면 됩니다.
- **애니메이션 WebP:** FFmpeg에서 애니메이션 WebP로 내보내는 옵션도 있지만, 실무에서는 “첫 프레임 = 정적 WebP”, “동영상 = WebM” 조합을 많이 씁니다.

---

## 4. 실전에서 자주 쓰는 값 정리

| 목적 | fps | scale | crf (WebM) | pix_fmt |
|------|-----|-------|------------|---------|
| 배치 WebM (캡처) | 15 | 뷰포트 그대로 | 18 | yuv444p |
| GIF 썸네일/공유 | 15 | 600 width | — | — |
| 정적 썸네일 | — | 720 width | — | — |

- **CRF 18:** 시각적으로 거의 무손실에 가깝게 보이는 경우가 많음.  
- **GIF scale 600:** SNS·채팅 공유용으로 적당한 크기.  
- 에러 처리: FFmpeg 자식 프로세스의 `stderr`를 모아서 exit code가 0이 아니면 마지막 몇 줄만 붙여서 에러 메시지로 쓰면 디버깅에 도움이 됩니다.

---

## 요약

| 단계 | 요약 |
|------|------|
| WebP → WebM | image2pipe + webp 입력, libvpx-vp9, CRF 18, yuv444p(선명도 우선) 또는 yuv420p(용량·호환). |
| WebM → GIF | fps + scale + palettegen + paletteuse (split 한 번에 처리 가능). |
| 파이프 | stdin으로 WebP 버퍼 스트리밍 시 drain 처리로 백프레셔 관리. |

캡처 단계는 `웹에서 캔버스 프레임을 이미지로 캡처하기.md`를, 배치 자동화는 `Puppeteer 헤드리스 브라우저로 배치 캡처하기.md`를 참고하면 됩니다.
