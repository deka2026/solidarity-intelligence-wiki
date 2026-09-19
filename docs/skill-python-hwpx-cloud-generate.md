# 스킬: 템플릿 없이 python-hwpx로 한글(HWPX) 문서 생성 (클라우드 세션용)

**작성일:** 2026-09-19 (2026-09-11 세무사 인터뷰 질문지에서 검증)

## 언제 쓰나

- 한글 프로그램도, 서식을 빌릴 템플릿 hwpx도 없는 환경(Claude Code 클라우드 세션 등)에서 표가 있는 한글 문서를 새로 만들 때.
- 기존 스킬 [[skill-hwpx-generate-from-template]]은 "템플릿 hwpx의 서식 재사용 + node zip 수술" 경로다. 이 스킬은 **pip 패키지 하나로 빈 문서부터 만드는 경로**로, 클라우드(pypi 접근 가능, python3.11)에서 더 빠르다.

## 절차 (검증 완료)

```bash
pip install python-hwpx        # 6.4.0에서 검증. lxml 동반 설치됨
python3 build.py out.hwpx      # 아래 API로 작성
```

핵심 API (`from hwpx.document import HwpxDocument`):

| 하고 싶은 것 | 호출 |
|---|---|
| 빈 문서 | `doc = HwpxDocument.new()` (내장 Skeleton.hwpx, 스타일 0=바탕글, 2~11=개요 1~10) |
| 용지·여백 | `doc.page.setup(paper_size="A4", margins_mm={"left":20,"right":20,"top":20,"bottom":18,"header":8,"footer":8})` |
| 글자 스타일 id | `cid = doc.ensure_run_style(bold=True, size=14, color="#1F3A5F", font="함초롬돋움")` → charPr id 문자열 |
| 문단 | `doc.add_paragraph(text, char_pr_id_ref=cid, inherit_style=False, para_pr_id_ref=0, style_id_ref=0)` |
| 문단 서식 | `doc.set_paragraph_format(paragraph_index=i, alignment="CENTER", spacing_before_pt=…, spacing_after_pt=…, line_spacing_percent=160, indent_left_mm=3, keep_with_next=True)` |
| 표 | `t = doc.add_table(rows, cols, width=48000)`; `t.set_column_widths([0.9,0.7,7.4,4.0])` (가중치) |
| 셀 텍스트·서식 | `cell = t.cell(r,c); cell.set_text(s)`; 굵게 등은 `for run in cell.paragraphs[0].runs: run.char_pr_id_ref = cid` |
| 셀 안 여러 문단 | `cell.add_paragraph(text, char_pr_id_ref=cid, para_pr_id_ref=0, style_id_ref=0)` |
| 셀 음영 | `t.set_cell_shading(r, c, "#DCE6F1")` |
| 쪽번호 | `doc.set_page_number(target="footer", align="CENTER", prefix="- ", suffix=" -")` |
| 검증·저장 | `doc.validate().ok` → `doc.save_to_path(path)` |

## 함정과 대응

1. **`add_heading`(개요 스타일)은 한글이 자동 번호를 붙일 수 있다.** 직접 "1단계." 같은 번호를 쓰는 문서면 개요 대신 `add_paragraph` + 굵은 charPr로 제목을 만든다.
2. **`add_table`은 앵커 문단 스타일을 상속하지 않는다(의도된 동작).** 표 앞 제목의 번호가 표 옆에 튀어나오는 것을 막기 위함. 그대로 두면 된다.
3. **`inherit_style=True`(기본)면 직전 문단의 charPr을 그대로 물려받아** 제목 크기가 본문에 전염된다. 문단마다 `inherit_style=False` + 명시적 charPr을 주는 헬퍼를 만들 것.
4. **python-hwpx는 `<hp:linesegarray>`를 문단에 넣지 않는다** (361문단 중 1개만 있었음). 리포의 선행 경험([[skill-hwpx-generate-from-template]] "모든 hp:p에 linesegarray 없으면 한글이 손상 거부")을 존중해 **저장 후 lxml로 후처리**했다:
   ```python
   for p in root.iter('{%s}p' % HP):
       if p.find('{%s}linesegarray' % HP) is None:
           n = max(1, ceil(len(text)/46))   # 대략값, 열 때 재계산됨
           # lineseg: textpos=i*46, vertpos 누적 1600, vertsize/textheight 1000, baseline 850, spacing 600, horzsize 42520, flags 393216
   ```
   후처리 뒤 `zipfile`로 재포장: `mimetype`은 첫 엔트리 + `ZIP_STORED`, 나머지 `ZIP_DEFLATED`. 재포장본도 `HwpxDocument.open().validate()` 통과.
5. **한글 없이 할 수 있는 검증**: `validate().ok`, `zipfile.testzip()`, 첫 엔트리 mimetype·method 0, `<hp:p` 열림/닫힘 수 일치, `export_text()`로 44문항 정규식 카운트. `export_html()`은 서식 없는 골격만 나오므로 시각 검증 대용이 안 된다. **최종 렌더링 확인은 사용자 PC의 한글에서.**
6. 리포에 넣을 파일명은 ASCII로 (`tax-accountant-interview-questions.hwpx`). 한글 파일명은 NFD 함정([[lesson-nfd-filenames-and-data-audit]]).

## 실행 원본

`wiki/assets/tax-accountant-interview-questions.build.py` (본 리포). 표 헬퍼 `qtable()`(번호·★·질문·메모 4열, ★ 셀 붉은 음영)과 `table()`(헤더 음영·첫 열 굵게)을 그대로 재사용 가능.

## 관련 문서

- [[handover-tax-accountant-interview-hwpx]] — 이 스킬을 쓴 작업
- [[skill-hwpx-generate-from-template]] — 템플릿 재사용 경로(Windows·node)
- [[handover-hwpx-line-fix]] — hwpx 내부 구조(header/section/PrvImage) 진단법
