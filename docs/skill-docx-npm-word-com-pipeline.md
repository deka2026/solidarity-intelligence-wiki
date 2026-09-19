# 스킬 — docx(npm) 계획서 생성 + Word COM 검증·PDF 변환 파이프라인

> 작성일: 2026-07-24 | 검증 환경: Windows 10, Node 24, MS Office 설치, python/soffice/pdftoppm 없음

## 요지

이 PC에서 **한국어 공문서 스타일 .docx**를 만들 때는 `docx`(npm, 전역 설치됨) 스크립트로 생성하고, **Word COM으로 열어 검증 + PDF 변환**까지 한 번에 끝낸다. hwpx 직접 생성(템플릿 필요)보다 훨씬 안전하다.

## 생성 (Node + docx)

```bash
NODE_PATH="C:\Users\pc\AppData\Roaming\npm\node_modules" node make_plan.js
```

- `docx`는 전역(`npm root -g`)에 이미 있음 — NODE_PATH만 지정하면 require 가능
- A4: `size: { width: 11906, height: 16838 }`, 여백 1134(2cm) → 본문폭 9638 DXA
- **표는 `columnWidths` + 모든 셀 `width`(DXA) 이중 지정** + `layout: TableLayoutType.FIXED` — 안 하면 레이아웃 깨짐
- 셀 음영은 `ShadingType.CLEAR`(SOLID는 검게 나옴), 글꼴은 '맑은 고딕' run마다 지정
- 공문서 수준 기호(ㅇ/-/※)는 numbering 대신 `indent: { left, hanging }` 문단으로 처리하는 게 간단
- `\n` 금지 — 문단 분리. PageBreak는 Paragraph 안에

## 검증 + PDF (Word COM)

```powershell
$word = New-Object -ComObject Word.Application; $word.Visible = $false
$doc = $word.Documents.Open("...docx")
$doc.ComputeStatistics(2)  # 쪽수(2=wdStatisticPages, 0=단어수) — 열리면 손상 아님
$doc.SaveAs([ref]"...pdf", [ref]17)  # 17 = wdFormatPDF
$doc.Close($false); $word.Quit()
```

- **Word가 열고 통계를 반환하면 문서 유효** — 이게 이 PC의 유일한 자동 검증 수단 (soffice/pdftoppm 없음)
- 생성된 PDF를 `pdftotext`로 뽑으면 **한글이 깨진다(글꼴 서브셋 ToUnicode 문제)** — 문서 결함이 아니므로 검증 목적으로는 쪽수·구조 확인만 하면 됨

## Excel COM 대량 시트 추출 (기존 skill-excel-com-no-python 보강)

- 여러 파일×시트 일괄 추출: `$ws.SaveAs("out.csv", 62)` (62 = xlCSVUTF8, BOM 포함)
- 6만 행짜리 서식-패딩 시트도 그대로 내보낸 뒤 Node에서 빈 행 스킵이 빠름
- **파일명에 한글 NFD·쉼표가 섞이면 경로 직타가 실패** → `Get-ChildItem`으로 FileInfo를 잡아 `$f.FullName`으로 열 것 (상세: lesson-nfd-filenames 참고)

## 관련

- [[skill-excel-com-no-python]] — COM 기반 xlsx 추출 원본 스킬
- [[lesson-nfd-filenames-and-data-audit]] — 이번에 걸린 함정
