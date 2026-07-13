# 스킬: PDF 텍스트 추출 · 붙여넣기 이미지 처리 · Quartz 위키 푸쉬 (이 PC 기준)

**작성일:** 2026-07-13

## 배운 것

### 1. 이 PC의 PDF 도구 상태
- `pdftotext`는 사용 가능: `/mingw64/bin/pdftotext`. 다량 PDF에서 텍스트만 빠르게 뽑을 때 유용.
  - 사용: `pdftotext -enc UTF-8 "입력.pdf" "출력.txt"`
- `pdftoppm`(PDF→이미지 렌더링)은 **미설치**. 그래서 Read 툴로 이미지형 PDF의 페이지를 이미지로 못 읽음 (`poppler-utils` 필요).
- `python` 명령은 **Windows 스토어 스텁**이라 스크립트 실행이 막힘(인터랙티브로 뜸). PDF 파싱을 python(pdfplumber/pymupdf)으로 하려 하지 말 것 → `pdftotext`로 우회.

### 2. 텍스트 PDF vs 이미지 PDF 빠른 판별
- `pdftotext` 출력이 수십 바이트 이하이면 **이미지로 스캔/렌더된 PDF**(텍스트 레이어 없음).
- 이번 사례: 1·2·4팀은 텍스트 있음(8~9KB), 3·5·6팀은 12~14바이트 → 이미지형. 렌더러 없으면 내용 확인 불가.

### 3. 붙여넣기(paste) 이미지는 파일로 저장 불가
- 사용자가 채팅에 **붙여넣은 이미지**는 볼 수는 있어도(내용 전사 가능), **원본 바이너리를 디스크 파일로 꺼낼 수 없다**.
- Temp/세션 디렉터리(`AppData/Local/Temp/claude/...`, `.claude/`)를 뒤져도 paste 캐시 파일이 남지 않았음.
- 대응: (a) 이미지 내용을 **텍스트로 완전 전사** + (b) 필요하면 **SVG로 재현**(재현본임을 명시) + (c) 원본이 필요하면 **실제 파일**을 폴더에 넣어달라고 요청.

### 4. Quartz 위키(solidarity-intelligence-wiki) 배포
- 콘텐츠 저장소는 `si-wiki-tmp` (Quartz 엔진 저장소 `sakyowon-wiki`와 별개). 문서는 `wiki/`, 운영 문서는 `docs/`.
- 배포 = `master`에 직접 `git add → commit → push origin master` (fork/PR 아님, 유지관리자 직접 푸쉬가 관행).
- 이미지 자산은 `wiki/assets/`에 두고 문서에서 상대경로 `../assets/파일`로 삽입.
- 온톨로지 규칙은 `docs/wiki-publish-prompt.md` 참고: 클래스 분류(concepts/field-notes/how-to/case-studies) + 템플릿 + 태그 3~5 + 내부 링크 1개 이상 + 파일명 `소문자-하이픈.md`.

## 재사용 스니펫

```bash
# 폴더 내 모든 PDF 텍스트 추출 (스크래치패드로)
for f in *.pdf; do pdftotext -enc UTF-8 "$f" "$OUT/${f%.pdf}.txt"; done

# 이미지형(텍스트 없는) PDF 골라내기: 출력 파일이 작으면 이미지형
```
