# 위키에 문서를 기여하는 방법

> **한 줄 요약**: GitHub 웹에서 문서를 작성하고 Pull Request로 제출하는 전체 과정

**태그**: `#위키` `#GitHub` `#기여방법` `#입문`  
**작성자**: 데카 (김일영)  
**작성일**: 2026-06-27  
**난이도**: 쉬움  
**소요 시간**: 약 15분

---

## 이 가이드가 필요한 상황

연대지능 활동가 교육을 받고, 처음으로 위키에 글을 올리고 싶을 때 따라하세요.

## 시작 전 준비

- GitHub 계정이 있어야 합니다 (없으면 [온보딩 가이드 1단계](../../docs/onboarding-guide.md) 참고)
- 이 저장소를 이미 Fork한 상태여야 합니다 (없으면 [온보딩 가이드 2단계](../../docs/onboarding-guide.md) 참고)

## 단계별 진행

### 1단계: 내 Fork 저장소로 이동

브라우저에서 `https://github.com/내아이디/solidarity-intelligence-wiki`에 접속합니다.

### 2단계: 적절한 폴더로 이동

글의 성격에 맞는 폴더를 클릭합니다:

| 이런 글이면 | 이 폴더로 |
|------------|----------|
| 개념을 정리했다 | `wiki/concepts/` |
| 현장 활동을 기록했다 | `wiki/field-notes/` |
| 도구 사용법을 정리했다 | `wiki/how-to/` |
| 사례를 분석했다 | `wiki/case-studies/` |

### 3단계: 템플릿 복사

1. `templates/` 폴더에서 해당 유형의 템플릿 파일을 엽니다
2. **Raw** 버튼을 클릭하면 원본 텍스트가 보입니다
3. 전체 선택(Ctrl+A) → 복사(Ctrl+C)

### 4단계: 새 파일 만들기

1. 다시 글을 넣을 폴더로 돌아갑니다
2. **Add file** → **Create new file** 클릭
3. 파일 이름 입력 (예: `ai-prompt-basics.md`)
   - 한글도 가능: `마을회의-진행법.md`
   - 반드시 `.md`로 끝나야 합니다
4. 복사한 템플릿을 붙여넣기(Ctrl+V)
5. 템플릿의 괄호 안 내용을 지우고 자기 글로 채웁니다

### 5단계: 저장 (Commit)

1. 페이지 아래 **Commit changes** 클릭
2. 커밋 메시지는 자동으로 채워져 있으니 그대로 둬도 됩니다
3. **Commit changes** 한번 더 클릭

### 6단계: Pull Request 제출

1. 내 Fork 저장소 메인 화면으로 돌아갑니다
2. **Contribute** → **Open pull request** 클릭
3. 제목과 설명을 채우고 **Create pull request** 클릭

## 잘 되었는지 확인하기

- PR이 만들어지면 중앙 저장소의 **Pull requests** 탭에 내 PR이 보입니다
- 리뷰어가 코멘트를 남기면 이메일 알림이 옵니다

## 자주 하는 실수

| 실수 | 해결 방법 |
|------|----------|
| 파일 확장자 `.md` 빠뜨림 | 파일명 끝에 `.md` 추가 |
| 원본 저장소에 직접 글을 씀 | 반드시 내 Fork에서 작성 (Fork 권한이 없으면 원본에 쓸 수 없으니 걱정 없음) |
| PR을 안 만들고 Fork에만 저장 | Fork에 저장한 것은 내 것만. PR을 해야 중앙에 반영됨 |
| Sync fork를 안 해서 충돌 발생 | PR 전에 Sync fork → Update branch 먼저 실행 |

## 관련 문서

- [[collaborative-intelligence]]

## 참고 자료

- [GitHub 공식 Fork 가이드](https://docs.github.com/en/get-started/quickstart/fork-a-repo)
