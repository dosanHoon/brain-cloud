
# WebP vs SVG, 그리고 Next.js에서의 실전 활용법

웹 성능 최적화를 고민한다면 "WebP와 SVG, 무엇을 언제 써야 할까?"라는 질문에 한 번쯤은 부딪히게 됩니다.  
이 포스팅에서는 WebP와 SVG의 차이점과 각각의 장단점, 그리고 Next.js에서의 활용 노하우를 간단하게 정리해봅니다.

---

## 1. WebP와 SVG의 차이점

|   | **WebP**         | **SVG**                       |
|---|------------------|-------------------------------|
| **포맷** | 비트맵(이미지)     | 벡터(수학적 도형, 코드 기반)     |
| **확대/축소** | 품질 저하 발생      | 무한 확대/축소 가능(손실 無)      |
| **용도** | 사진, 썸네일, UI 배경 등 | 아이콘, 로고, 일러스트, UI 도형 등  |
| **용량** | 매우 작음, 압축 효율 좋음 | 단순 도형일 때 매우 작음           |
| **애니메이션** | 지원(제한적)         | CSS/JS로 자유로운 애니메이션      |
| **스타일링** | 불가                  | CSS로 색상·크기·스타일 동적 제어   |
| **브라우저 지원** | 최신 브라우저 대부분 | 대부분 지원                        |

---

## 2. 언제 무엇을 써야 할까?

- **WebP**  
  - 고해상도 사진, 섬네일, 배경 이미지, 복잡한 그래픽 등  
  - jpg, png 대신 WebP 사용 시 용량 ↓, 로딩속도 ↑

- **SVG**  
  - 로고, 아이콘, 간단한 일러스트, UI의 도형 등  
  - 해상도 무관, 컬러/크기/애니메이션 등 동적 제어 필요할 때  
  - 디자인 시스템, 반응형 UI에서 강력함

---

## 3. Next.js에서의 활용

### (1) **WebP 사용법**

```jsx
import Image from "next/image";

// Next.js는 jpg, png를 넣어도 webp로 자동 변환(지원 브라우저에 한해)
<Image
  src="/banner.jpg"
  width={800}
  height={400}
  alt="메인 배너"
  quality={80}
/>

// 직접 WebP 파일 사용도 가능
<Image
  src="/logo.webp"
  width={120}
  height={60}
  alt="로고"
/>
````

- Next.js는 서버에서 브라우저별 포맷(webp/jpg)로 자동 폴백 제공
    
- 외부 CDN 이미지도 가능 (next.config.js에 도메인 추가 필요)
    

---

### (2) **SVG 사용법**

#### 1. **img 태그/Next.js Image로 외부 SVG 불러오기**

```jsx
<Image
  src="/icon.svg"
  width={32}
  height={32}
  alt="아이콘"
/>
```

- 단순 렌더링용: 외부 SVG 파일, CDN, 퍼블릭 폴더에 저장 후 사용
    

#### 2. **Inline SVG(컴포넌트화, 스타일링/애니메이션)**

```jsx
const Logo = () => (
  <svg viewBox="0 0 100 20" width={100} height={20}>
    <text x="0" y="15" fontSize="16">Brand</text>
  </svg>
);

export default Logo;
```

- **직접 코드로 임포트**하면 CSS, JS로 스타일/애니메이션 제어 가능
    
- 라이브러리 활용: [SVGR](https://react-svgr.com/)  
    (`import { ReactComponent as Icon } from './icon.svg';`)
    

---

## 4. 실전 팁

- 사진·복잡한 그래픽: **WebP**
    
- 아이콘·로고·UI 요소: **SVG (inline 활용 시 확장성 UP)**
    
- Next.js `<Image />`는 대부분 자동 포맷/최적화 지원
    
- SVG를 컴포넌트화하면 테마/색상 등 동적 컨트롤이 매우 쉬워진다
    

---

## 결론

WebP와 SVG는 경쟁 관계가 아니라,  
"목적에 맞는 곳에 각각 활용"하는 것이 성능과 확장성 모두에 최적입니다.  
Next.js 환경이라면 둘 다 쉽게, 그리고 효율적으로 쓸 수 있다는 점도 기억하세요!