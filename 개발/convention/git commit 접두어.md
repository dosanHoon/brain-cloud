
# 커밋 메시지에 `feat`, `fix` 왜 쓰는 걸까? — 의미 있는 커밋 접두어 정리

개발을 하다 보면 이런 커밋 메시지를 본 적 있을 거예요:

```

feat: 로그인 기능 추가  
fix: 회원가입 오류 수정  
docs: README 업데이트

```

이게 뭘까 싶었다면, 이 글이 도움이 될 거예요.

## Conventional Commits란?

**Conventional Commits**는 커밋 메시지를 의미 있게 작성하기 위한 컨벤션이에요.  
쉽게 말해, **“무엇을 왜 변경했는지 한 줄로 명확하게 알려주는 방식”**이죠.

CI/CD 자동화, 릴리즈 노트 자동 생성, 협업 이력 추적 등에도 유리한 스타일입니다.

---

## 자주 쓰는 커밋 접두어 정리

| 접두어 | 설명 | 예시 |
|--------|------|------|
| `feat` | 새로운 기능 추가 | `feat: 다크모드 토글 기능 추가` |
| `fix` | 버그 수정 | `fix: 로그인 시 토큰 만료 오류 수정` |
| `docs` | 문서 수정 | `docs: README에 설치 방법 추가` |
| `style` | 코드 포맷팅 변경 (기능 변화 없음) | `style: prettier 적용 및 세미콜론 정리` |
| `refactor` | 리팩토링 (기능은 그대로, 코드 구조 개선) | `refactor: useEffect 로직 분리` |
| `perf` | 성능 개선 | `perf: 이미지 lazy loading 적용` |
| `test` | 테스트 코드 추가/수정 | `test: 로그인 유닛 테스트 추가` |
| `chore` | 빌드 설정, 패키지 관리 등 기타 | `chore: eslint 버전 업데이트` |
| `build` | 빌드 관련 설정 변경 | `build: Vite 설정 수정` |
| `ci` | CI 설정 수정 | `ci: GitHub Actions 워크플로 수정` |
| `revert` | 이전 커밋 되돌림 | `revert: api url 변경 되돌리기` |

---

## 부가적으로 쓸 수 있는 접두어들

| 접두어 | 설명 |
|--------|------|
| `wip` | Work In Progress (작업 중, 미완성) |
| `temp` | 임시 커밋, 추후 정리 필요 |
| `hotfix` | 긴급 수정 (대부분 `fix`로 대체 가능) |

---

## 커밋 메시지 구조 예시

```

(optional scope):

```

예:

```

feat(auth): 소셜 로그인 버튼 추가  
fix(api): 유저 데이터 null 예외 처리  
refactor(component): 모달 구조 개선  
chore(deps): react-query 버전 업그레이드

```

---

## 마무리

지금까지 커밋 메시지를 그냥 "수정", "ㅇㅇ" 이렇게 써왔다면,  
오늘부터 한 줄만 더 고민해서 커밋해보세요.

**“지나간 커밋을 보면 내 개발 여정이 보인다”**  
이게 Conventional Commit의 진짜 매력입니다.

---

📌 참고: [Conventional Commits 공식 문서](https://www.conventionalcommits.org/)

