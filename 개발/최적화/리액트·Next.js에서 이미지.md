
# 리액트·Next.js에서 이미지 최적화 실전 — WebP와 폴백 처리까지

이미지 최적화는 웹 성능 개선의 핵심입니다.  
특히 **WebP** 같은 최신 이미지 포맷은 품질을 유지하면서 용량을 대폭 줄여줍니다.  
하지만 모든 브라우저가 WebP를 지원하지는 않으니, **폴백(Fallback) 처리**까지 꼼꼼하게 챙기는 게 중요하죠.

---

## 1. WebP란?

- Google이 개발한 **차세대 이미지 포맷**
- 같은 품질 기준에서 jpg, png보다 용량을 확 줄일 수 있음
- 요즘 크롬, 사파리 등 대부분 브라우저에서 지원  
  [WebP 지원현황](https://caniuse.com/webp)

---

## 2. `<picture>` 태그로 폴백 처리하기 (Vanilla/React)

모든 브라우저에서 WebP를 안전하게 쓰려면 `<picture>` 태그를 활용합니다.

```jsx
<picture>
  <source srcSet="/image.webp" type="image/webp" />
  <img src="/image.jpg" alt="설명" loading="lazy" />
</picture>
````

- **srcSet + type**: 브라우저가 webp 지원하면 해당 소스를, 아니면 img 태그의 jpg를 자동 선택
    
- React에서도 똑같이 사용 가능
    

---

## 3. Next.js `<Image />`에서 WebP 활용하기

Next.js의 `<Image />` 컴포넌트는  
이미지 최적화, 지연 로딩, 자동 사이즈 조절, 포맷 변환까지 대부분 자동 지원합니다.

### 기본 사용법

```jsx
import Image from "next/image";

<Image
  src="/image.jpg" // 원본 jpg면, Next.js가 webp로 자동 변환해서 제공
  alt="설명"
  width={600}
  height={400}
  quality={80}
  priority
/>
```

- **src에 jpg/png를 넣어도**, Next.js는 클라이언트가 webp를 지원하면 자동 변환하여 전달
    
- 직접 webp 이미지를 src에 넣을 수도 있음
    

### 자동 WebP 변환과 폴백

- Next.js의 `<Image />`는 **브라우저가 webp 지원하면 webp로**,  
    **지원하지 않으면 원본(jpg/png)**으로 자동 처리함  
    (서버에서 accept 헤더 보고 변환)
    
- 따로 `<picture>` 태그와 폴백 소스를 지정할 필요 없음
    

---

## 4. 기타 실전 팁

- **CDN + Next.js Image** 조합:  
    외부 이미지도 최적화하려면 `next.config.js`에 도메인 whitelisting 추가
    
- **이미지 용량**:  
    원본 이미지는 가볍게, 필요 없다면 해상도 낮추기
    

---

## 마무리

WebP는 성능을 확 올려주는 강력한 포맷입니다.  
Vanilla/React 환경에서는 `<picture>`로 폴백을,  
Next.js에서는 `<Image />`만 잘 써도 자동 변환/폴백이 지원됩니다.

"이미지, 폰트 최적화만 해도 퍼포먼스는 두 배!"

---

**요약:**  
- `<picture>` 태그로 webp/jpg 등 폴백 지정  
- Next.js `<Image />`는 자동 webp 변환+폴백 처리됨  
- `src`에 jpg/png만 넣어도 ok, 직접 webp 넣어도 ok
