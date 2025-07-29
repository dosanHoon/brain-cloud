

## 1. CDN에서 이미지 최적화 - URL 쿼리/Path 활용

많은 이미지 CDN은 **url 파라미터(또는 path)** 만 바꿔주면  
자동으로 포맷 변환, 리사이즈, 품질 조절, 압축 등을 해줍니다.

### 🔹 **대표 CDN별 URL 패턴 예시**

#### **Imgix**

```
https://도메인.imgix.net/example.jpg?w=400&h=400&auto=format,compress
```

- `w`, `h`: 리사이즈
    
- `auto=format`: 브라우저별 포맷(webp 등) 자동 전환
    
- `auto=compress`: 압축
    

#### **Cloudflare Images**

```
https://imagedelivery.net/XXX/이미지ID/public
https://imagedelivery.net/XXX/이미지ID/public/w=600,quality=80,format=webp
```

- `w=600`: 너비 지정
    
- `format=webp`: 포맷 강제 지정
    

#### **AWS CloudFront + Lambda@Edge/Thumbor 등**

- 리버스 프록시 방식으로 `?width=300&format=webp` 등 다양한 방식 지원
    

---

## 2. Next.js `<Image />`에서 CDN 이미지 활용

### 1) **외부 이미지 사용시 domains 허용**

**next.config.js**

```js
module.exports = {
  images: {
    domains: ["도메인.imgix.net", "imagedelivery.net", "cloudfront.net"], // 사용하는 CDN 도메인 등록
  },
};
```

### 2) **CDN 최적화 파라미터로 이미지 요청**

```jsx
import Image from "next/image";

// 예시: Imgix
<Image
  src="https://도메인.imgix.net/example.jpg?w=400&h=400&auto=format,compress"
  alt="이미지"
  width={400}
  height={400}
/>

// 예시: Cloudflare Images
<Image
  src="https://imagedelivery.net/XXX/이미지ID/public/w=600,quality=80,format=webp"
  alt="이미지"
  width={600}
  height={400}
/>
```

- **src에 CDN 파라미터가 적용된 URL을 직접 넣으면**  
    브라우저는 해당 조건으로 최적화된 이미지를 받아옴.
    
- CDN에서 `auto=format`(혹은 `format=webp`)을 넣으면  
    브라우저별로 webp/jpg 자동 변환 제공(=폴백까지 자동 처리).
    

---

### 3) **Next.js Image loader 커스텀 (고급)**

이미지 URL의 쿼리를 더 유연하게 컨트롤하고 싶다면  
**커스텀 로더(loader)**도 활용 가능.

```js
// components/MyImgixImage.js
import Image from "next/image";

const myLoader = ({ src, width, quality }) => {
  return `https://도메인.imgix.net/${src}?w=${width}&auto=format&quality=${quality || 75}`;
};

export default function MyImgixImage(props) {
  return <Image loader={myLoader} {...props} />;
}
```

```jsx
<MyImgixImage src="example.jpg" width={400} height={400} />
```

---

## 3. 요약 & 실전 체크리스트

- **CDN URL 파라미터로 리사이즈/포맷 변환 등 최적화**  
    (CDN마다 파라미터 문서 확인)
    
- **Next.js 이미지 도메인 whitelisting 필수**
    
- **CDN 파라미터 적용된 이미지 URL을 `<Image />`에 직접 사용**  
    → 최적화 자동 반영
    
- **필요하면 커스텀 로더로 동적 url 생성**
    

---

### ✅ **실전 요약 코드**

```js
// next.config.js
images: {
  domains: ["도메인.imgix.net", "imagedelivery.net"],
},

// 페이지 컴포넌트
<Image
  src="https://도메인.imgix.net/sample.jpg?w=600&auto=format"
  width={600}
  height={400}
  alt="이미지"
  quality={80}
/>
```

---

더 필요한 CDN별 상세 예시,  
혹은 S3-CloudFront-Thumbor 등 내부 이미지 서버 연동법이 궁금하시면  
추가로 말씀해 주세요!