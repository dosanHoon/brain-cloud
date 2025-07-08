
실무에서 한 번쯤 “어, 내 이름 아닌데?” 하는 경우 있죠.  
아래는 **특정 커밋 하나**의 author만 바꾸는 가장 대표적이고 안전한 방법입니다.

---

## 1. 단일 커밋(author, committer 둘 다) 변경 (가장 쉬운 방법: `git commit --amend`)

**마지막 커밋의 author만 바꾸고 싶을 때:**

```bash
git commit --amend --author="새이름 <newemail@example.com>"
```

- 명령을 실행하면 에디터가 열립니다.
    
- 메시지는 그대로 두고 저장 & 종료하면 됩니다.
    
- 이제 author와 committer 정보가 바뀐 상태로 커밋이 교체됩니다.
    

---

## 2. 이전 커밋(여러 개 중 하나) 변경: `git rebase -i`

특정 커밋(예: 마지막에서 두 번째, 세 번째 등)의 author를 바꾸려면 **인터랙티브 리베이스**를 사용해야 합니다.

### 1) 리베이스 실행

바꾸려는 커밋이 마지막에서 몇 번째인지 세서(예시: 마지막 3개 중 하나 바꾼다면)

```bash
git rebase -i HEAD~3
```

### 2) 에디터에서 바꿀 커밋을 `edit`으로 변경

예시:

```
pick aaaaa11 원래커밋1
edit bbbbb22 바꾸고싶은커밋
pick ccccc33 원래커밋2
```

### 3) 해당 커밋에서 정지됨

이제 author를 변경합니다:

```bash
git commit --amend --author="새이름 <newemail@example.com>"
```

저장 후,

```bash
git rebase --continue
```

---

## 3. **(주의)**

- 이렇게 변경한 커밋은 **해시가 바뀌므로** 이미 푸시한 커밋이라면  
    강제 푸시(`git push --force`)가 필요합니다.
    
- 협업 중인 리포지토리라면 팀원과 반드시 상의하세요.
    

---

## 4. 간단 예시

### **마지막 커밋만 바꿀 때**

```bash
git commit --amend --author="후니 <hoony@example.com>"
```

### **몇 커밋 전(author를 바꿀 커밋이 여러 개)**

```bash
git rebase -i HEAD~3
# 바꿀 커밋을 edit
git commit --amend --author="후니 <hoony@example.com>"
git rebase --continue
```

---

필요하다면 커밋 히스토리 바꾸는 명령어나, 커밋 여러 개 한 번에 바꾸는 스크립트 등도 안내해드릴 수 있습니다.  
추가로 궁금한 점 있으면 말씀해 주세요!