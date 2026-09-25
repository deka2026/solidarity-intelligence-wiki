# 스킬 — 마크다운 위키 문서를 한글(hwpx)로 변환 (python-hwpx, 템플릿 불필요)

> 작성일: 2026-09-25 | 검증 환경: 리눅스(Claude Code 클라우드), Python 3.11, python-hwpx 6.5.0, 한글 프로그램 없음

## 요지

템플릿 hwpx가 없어도 **`pip install python-hwpx`** 로 빈 문서(`HwpxDocument.new()`)에서 시작해 제목·본문·표를 넣고 저장할 수 있다. 다만 python-hwpx는 **첫 문단 말고는 `<hp:linesegarray>`를 쓰지 않으므로**, 저장 후 모든 문단(표 셀 안 문단 포함)에 linesegarray를 보강해야 한다. 이게 없으면 한글이 "손상된 파일"로 거부한다([[skill-hwpx-generate-from-template]]에서 확인된 함정).

## 절차

1. `pip install python-hwpx` (pypi는 클라우드 프록시 예외라 설치됨)
2. `md2hwpx.py 원문.md 출력.hwpx` — 제목(#~####)·표·체크리스트·목록·인용·구분선을 변환
3. `fix_lineseg.py 출력.hwpx` — linesegarray 없는 모든 `hp:p`에 대략값 lineseg 추가, mimetype 첫 엔트리 Stored로 재포장
4. 검증 (한글 없이)
   - `HwpxDocument.open(f).validate().ok == True`
   - zip 첫 엔트리 `mimetype`, compress_type 0, `testzip()` 통과
   - `section0.xml`에서 `<hp:p` 개수 = `<hp:linesegarray` 개수
   - `doc.text.plain()`으로 텍스트 역추출해 핵심 문구 포함 확인

## 변환 규칙 (md2hwpx.py)

- 제목: 크기 18/14/12/11pt, 굵게, 남색(#1F3864). 2단계 이하 제목 앞에 빈 문단
- 표: `doc.add_table` + `set_cell_text`, 머리행 음영 #D9E2F3, 구분행(`---`) 무시
- `**굵게**` → 굵은 run, `[[링크]]`·`[t](url)`·백틱은 평문으로
- ⚠️ 이모지 → ※ (한글 글꼴에서 깨짐 방지), `- [x]` → ☑, `- [ ]` → ☐
- 쪽 번호: `doc.set_page_number()` (6.x에서 deprecated → `doc.page.set_page_number`)

## 함정

- 표를 넣을 때 `add_table`은 스타일을 상속하지 않는 새 앵커 문단을 만든다(개요 번호가 표 옆에 찍히는 문제 방지). 표 뒤에 빈 문단을 하나 넣어야 다음 제목과 붙지 않는다
- lineseg 보강은 lxml로 **직접 자식** `hp:linesegarray` 유무를 봐야 한다(표 셀의 중첩 `hp:p`도 각각 필요)
- 원문 md를 고치면 hwpx는 자동으로 안 바뀐다 → `wiki/assets/README.md`에 "원문 수정 시 재변환" 명시

## md2hwpx.py (전문)

```python
import re, sys
from hwpx import HwpxDocument

def clean(s):
    s = s.replace('⚠️', '※').replace('⚠', '※')
    s = re.sub(r'\[\[([^\]]+)\]\]', r'\1', s)          # [[wikilink]]
    s = re.sub(r'\[([^\]]+)\]\(([^)]+)\)', r'\1(\2)', s) # [t](url)
    s = s.replace('`', '')
    return s

def runs(p, text, size=None, color=None):
    """**굵게** 구간을 굵은 run으로."""
    parts = re.split(r'(\*\*[^*]+\*\*)', clean(text))
    for part in parts:
        if not part: continue
        if part.startswith('**') and part.endswith('**'):
            p.add_run(part[2:-2], bold=True, size=size, color=color)
        else:
            p.add_run(part, size=size, color=color)

def para(doc, text, size=10, color=None, bold_all=False):
    p = doc.add_paragraph('', inherit_style=False, include_run=False)
    if bold_all:
        p.add_run(clean(text).replace('**', ''), bold=True, size=size, color=color)
    else:
        runs(p, text, size=size, color=color)
    return p

def table(doc, rows):
    rows = [[clean(c).replace('**', '').strip() for c in r] for r in rows]
    ncol = max(len(r) for r in rows)
    t = doc.add_table(len(rows), ncol)
    for i, r in enumerate(rows):
        for j in range(ncol):
            t.set_cell_text(i, j, r[j] if j < len(r) else '')
        if i == 0:
            for j in range(ncol):
                t.set_cell_shading(0, j, '#D9E2F3')
    doc.add_paragraph('', inherit_style=False)

def convert(md_path, out_path):
    doc = HwpxDocument.new()
    lines = open(md_path, encoding='utf-8').read().splitlines()
    i = 0
    while i < len(lines):
        ln = lines[i].rstrip()
        if not ln.strip():
            i += 1; continue
        if ln.startswith('|'):
            tbl = []
            while i < len(lines) and lines[i].startswith('|'):
                cells = [c for c in lines[i].strip().strip('|').split('|')]
                if not all(re.fullmatch(r'\s*:?-{2,}:?\s*', c) for c in cells):
                    tbl.append(cells)
                i += 1
            table(doc, tbl); continue
        m = re.match(r'^(#{1,4})\s+(.*)', ln)
        if m:
            lvl = len(m.group(1)); txt = m.group(2)
            size = {1: 18, 2: 14, 3: 12, 4: 11}[lvl]
            if lvl > 1: doc.add_paragraph('', inherit_style=False)
            para(doc, txt, size=size, color='#1F3864', bold_all=True)
        elif re.fullmatch(r'-{3,}', ln.strip()):
            para(doc, '─' * 40, size=8, color='#999999')
        elif ln.startswith('>'):
            para(doc, ln.lstrip('> ').strip(), size=10, color='#7F4F00')
        elif re.match(r'^\s*- \[( |x)\]\s+', ln):
            box = '☑' if '[x]' in ln else '☐'
            para(doc, box + ' ' + re.sub(r'^\s*- \[( |x)\]\s+', '', ln))
        elif re.match(r'^\s*[-*]\s+', ln):
            ind = len(ln) - len(ln.lstrip())
            para(doc, '  ' * (ind // 2 + 1) + '• ' + re.sub(r'^\s*[-*]\s+', '', ln))
        elif re.match(r'^\s*\d+\.\s+', ln):
            para(doc, '  ' + ln.strip())
        else:
            para(doc, ln.strip())
        i += 1
    doc.set_page_number()
    doc.save_to_path(out_path)
    rep = doc.validate()
    print(out_path, 'validate:', getattr(rep, 'ok', rep))

if __name__ == '__main__':
    convert(sys.argv[1], sys.argv[2])
```

## fix_lineseg.py (전문)

```python
import sys, zipfile, math, io
from lxml import etree
HP = 'http://www.hancom.co.kr/hwpml/2011/paragraph'
def fix(path):
    zin = zipfile.ZipFile(path); items = [(i, zin.read(i.filename)) for i in zin.infolist()]; zin.close()
    out = io.BytesIO(); added = 0
    with zipfile.ZipFile(out, 'w') as zout:
        for info, data in items:
            if info.filename.startswith('Contents/section') and info.filename.endswith('.xml'):
                root = etree.fromstring(data); y = 0
                for p in root.iter(f'{{{HP}}}p'):
                    if p.find(f'{{{HP}}}linesegarray') is not None: continue
                    text = ''.join(t.text or '' for r in p.findall(f'{{{HP}}}run') for t in r.findall(f'{{{HP}}}t'))
                    arr = etree.SubElement(p, f'{{{HP}}}linesegarray')
                    for k in range(max(1, math.ceil(len(text) / 46))):
                        etree.SubElement(arr, f'{{{HP}}}lineseg', textpos=str(k*46), vertpos=str(y), vertsize='1000',
                            textheight='1000', baseline='850', spacing='600', horzpos='0', horzsize='42520', flags='393216')
                        y += 1600
                    added += 1
                data = etree.tostring(root, xml_declaration=True, encoding='UTF-8', standalone=True)
            ct = zipfile.ZIP_STORED if info.filename == 'mimetype' else zipfile.ZIP_DEFLATED
            zi = zipfile.ZipInfo(info.filename, date_time=info.date_time); zi.compress_type = ct
            zout.writestr(zi, data)
    open(path, 'wb').write(out.getvalue()); print(path.split('/')[-1], 'linesegarray added:', added)
for f in sys.argv[1:]: fix(f)
```

## 관련

- [[skill-hwpx-generate-from-template]] — 윈도 PC(python 없음)에서 기존 hwpx 템플릿을 node로 조작하는 방식
- [[handover-local-agency-docs-and-list]] — 이 스킬을 쓴 작업
