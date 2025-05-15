
# 🎨 Figma 디자인을 코드로! Cursor IDE + MCP 연동 가이드

> Figma에서 만든 UI를 자동으로 코드로 생성해주는 대 딸깍의 시대!  
> Cursor IDE와 MCP를 활용해 **Figma → React + Tailwind 코드 자동 생성** 워크플로우를 구축해보자.

---

## ✅ 준비 사항

| 항목            | 설명                                                |
| ------------- | ------------------------------------------------- |
| Cursor IDE    | [cursor.sh](https://www.cursor.sh)에서 설치           |
| Figma API Key | Figma → Settings → Personal Access Token 발급       |
| Node.js       | MCP 서버 실행용                                        |


---

## 1️⃣ Figma API Key 발급

1. [https://www.figma.com/settings](https://www.figma.com/settings) 접속
2. `Personal Access Tokens`에서 `Generate new token`
3. 토큰 복사 후 안전하게 저장

---

## 3 Cursor IDE에 MCP 서버 연동

### MacOS / Linux
```json
{
  "mcpServers": {
    "Framelink Figma MCP": {
      "command": "npx",
      "args": ["-y", "figma-developer-mcp", "--figma-api-key=YOUR-KEY", "--stdio"]
    }
  }
}
```

### Windows
```json
{
  "mcpServers": {
    "Framelink Figma MCP": {
      "command": "cmd",
      "args": ["/c", "npx", "-y", "figma-developer-mcp", "--figma-api-key=YOUR-KEY", "--stdio"]
    }
  }
}
```

> `--stdio` 옵션은 Cursor와 MCP 간의 통신을 위한 필수 설정입니다.

참고 : https://github.com/GLips/Figma-Context-MCP

---

## 4️⃣ Agent Mode 진입 (⌘+I 또는 Ctrl+I)

1. Cursor IDE에서 **Agent Mode** 활성화 (단축키 ⌘+I / Ctrl+I)
2. 다음과 같이 입력:

```bash
이 Figma 디자인을 기반으로 React + Tailwind 코드로 변환해줘:
https://www.figma.com/file/xxxxx/your-design
```

3. Cursor가 Figma MCP를 통해 디자인을 분석하고, 자동으로 컴포넌트/레이아웃/스타일 코드를 생성합니다.

---

## 5️⃣ 생성된 코드 확인 및 프로젝트 반영

Cursor는 보통 다음과 같은 디렉터리에 코드를 생성합니다:

```
/components/YourComponent.tsx
/pages/your-page.tsx
/styles/tailwind.config.ts (필요 시)
```


---

## ✅ 실전 팁

* Figma 내 디자인은 **프레임 명, 컴포넌트 명**을 명확하게 지정하면 더 정확한 코드가 생성됩니다.
* Atom, Molecule 수준의 작은 단위부터 자동화 적용을 추천합니다.
* 코드가 완벽하진 않지만 **초안 작성 + 구조 정리에 매우 효과적**입니다.

---

## 🧠 마무리

Cursor + Figma MCP는 단순한 코드 추천을 넘어서, **디자인 기반의 코드 자동화**를 실현해주는 강력한 도구입니다.

> "복잡한 UI 설계? 이제는 디자이너의 손끝에서 바로 코드로 이어지게 하자."

궁금한 점이나 활용 사례가 있다면 댓글 또는 GitHub에서 공유해주세요!

---

## 🔗 참고 자료

* [Cursor 공식 홈페이지](https://cursor.sh)
* [Figma-Context-MCP GitHub](https://github.com/GLips/Figma-Context-MCP)
* [Figma API Docs](https://www.figma.com/developers/api)
