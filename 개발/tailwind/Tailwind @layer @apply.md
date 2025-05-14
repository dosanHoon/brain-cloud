
---
# 💡 Tailwind CSS 완전 정복: `@layer`, `@apply`, 그리고 재사용 가능한 텍스트 스타일 설계법

Tailwind CSS를 쓰다 보면 마주치는 낯선 문법들,  
`@layer`, `@apply`... 도대체 이건 뭐고 왜 쓰는 걸까?

이번 글에서는 이 둘의 개념과 실무에서 어떻게 효율적으로 활용할 수 있는지를 집중적으로 정리합니다.

---

## 🎯 TL;DR 요약

| 기능       | 목적                                        | 예시                                                     |
| -------- | ----------------------------------------- | ------------------------------------------------------ |
| `@apply` | 여러 유틸리티 클래스를 조합해서 커스텀 클래스 정의              | `.btn { @apply px-4 py-2 bg-blue-500 text-white; }`    |
| `@layer` | Tailwind의 레이어 구조에 스타일을 명시적으로 넣어 최적화/충돌 방지 | `@layer components`, `@layer utilities`, `@layer base` |

---

## ✨ `@apply`란?

Tailwind의 유틸리티 클래스를 CSS 안에서 사용할 수 있도록 해주는 지시어입니다.  
반복되는 스타일을 재사용 가능한 클래스로 묶고 싶을 때 유용하죠.

### ✅ 예시

```css
@layer components {
  .btn-primary {
    @apply bg-blue-600 text-white font-semibold px-4 py-2 rounded;
  }
}
````

이제 HTML에서는:

```html
<button class="btn-primary">버튼</button>
```

이렇게 깔끔하게 사용할 수 있습니다.

---

## 🧱 `@layer`란?

Tailwind는 스타일의 우선순위와 purge 최적화를 위해 **3개의 레이어 구조**를 사용합니다:

### 1. `@layer base`

- 전역 태그에 스타일을 적용하고 싶을 때 사용
    
- reset.css, normalize.css 대체 용도
    

```css
@layer base {
  body {
    font-family: Pretendard, sans-serif;
    background-color: var(--gray-900);
  }
}
```

---

### 2. `@layer components`

- 재사용 가능한 UI 구성요소(Button, Text, Card 등)를 정의할 때 사용
    

```css
@layer components {
  .text-14-medium {
    @apply text-[14px] font-medium leading-[20px] text-[var(--gray-900)];
  }
}
```

---

### 3. `@layer utilities`

- Tailwind의 유틸리티 시스템을 확장할 때 사용
    
- 예: 그림자, 트랜지션, 효과 등을 커스텀하게 정의할 때
    

```css
@layer utilities {
  .shadow-heavy {
    box-shadow: 0px 6px 18px rgba(0, 0, 0, 0.3);
  }
}
```

---

## 🧠 `@layer`를 써야 하는 이유

Tailwind는 빌드 시 실제 사용된 클래스만 포함시키기 위해  
**`content` 설정을 기반으로 purge(제거)**합니다.

즉, 아래처럼 사용하면 purge 대상에서 누락됩니다 ❌

```css
.text-14-medium {
  @apply text-sm font-medium;
}
```

하지만 이렇게 `@layer components` 안에 넣으면 ✅

```css
@layer components {
  .text-14-medium {
    @apply text-sm font-medium;
  }
}
```

**빌드 시 정상 포함되고, 충돌도 방지됩니다.**

---

## 📐 실전 예시: 텍스트 스타일 정의하기

### 💬 반복되는 텍스트 스타일을 한 번에 추상화

```css
@layer components {
  .text-18-bold {
    @apply text-[18px] font-bold leading-[26px] text-[var(--gray-900,#f7f8fa)];
  }

  .text-13-medium {
    @apply text-[13px] font-medium leading-[18px] text-[var(--gray-700,#c6c8cc)];
  }

  .text-13-semibold {
    @apply text-[13px] font-semibold leading-[18px] text-[var(--gray-700,#c6c8cc)];
  }

  .text-14-medium {
    @apply text-[14px] font-medium leading-[20px] text-[var(--gray-900,#f7f8fa)];
  }
}
```

HTML에서 이렇게 사용:

```html
<p class="text-13-medium">설명 텍스트</p>
<p class="text-18-bold">타이틀 텍스트</p>
```

---

## ✅ 실무 팁 요약

| 개념                  | 설명                    | 주 용도                        |
| ------------------- | --------------------- | --------------------------- |
| `@apply`            | 유틸리티 조합해서 새로운 클래스 만들기 | `.btn`, `.text-label`       |
| `@layer base`       | 전역 태그 스타일 (reset)     | `body`, `input` 등           |
| `@layer components` | 재사용 가능한 UI 단위 스타일     | `.btn-primary`, `.card`     |
| `@layer utilities`  | 커스텀 유틸리티 확장           | `.shadow-heavy`, `.blur-xs` |

---

## ✅ 마무리

Tailwind는 단순 유틸리티 클래스만 쓰는 프레임워크가 아닙니다.  
**`@apply`로 의미 있는 스타일 조합을 만들고,  
`@layer`로 올바른 계층에 배치하여 퍼포먼스와 유지보수성을 확보**하는 것이 핵심입니다.

디자인 시스템을 구축 중이거나, 텍스트/버튼 스타일을 반복해서 쓰고 있다면  
지금 바로 `@layer components`를 도입해보세요!


---

