# 스킬: Python 없는 Windows PC에서 Excel 파일(xls/xlsx) 읽기 — Excel COM + PowerShell

**작성일:** 2026-07-15

## 배경

이 PC(사무실 Windows 10)는 `python`·`py`가 없고(스토어 스텁만 존재), LibreOffice(`soffice`)도 없다. 그래서 openpyxl/pandas 기반의 xlsx 스킬 스크립트는 전부 못 쓴다. **설치된 MS Excel의 COM 자동화가 유일하고 확실한 경로**다.

## 배운 것

### 1. Excel COM으로 읽기 (파일이 Excel에서 열려 있어도 OK)

```powershell
$excel = New-Object -ComObject Excel.Application
$excel.Visible = $false; $excel.DisplayAlerts = $false
$wb = $excel.Workbooks.Open($path, 0, $true)   # 0=링크 갱신 안 함, $true=읽기전용
foreach ($ws in $wb.Worksheets) {
  $data = $ws.UsedRange.Value2   # 2차원 배열 [row, col], 1-기반
}
$wb.Close($false); $excel.Quit()
[System.Runtime.Interopservices.Marshal]::ReleaseComObject($excel) | Out-Null
```

- 사용자가 해당 파일을 Excel로 열어둔 상태여도 **읽기전용 Open은 성공**한다.
- 반면 `[System.IO.File]::ReadAllBytes()`는 잠김 오류 — 헤더만 볼 거면 `File.Open($p,'Open','Read',[System.IO.FileShare]::ReadWrite)`로 우회.
- `.xls`(구형 BIFF)도 COM은 그냥 읽는다. 셀 단위 루프 대신 `UsedRange.Value2`로 한 번에 배열을 받아야 빠르다.

### 2. 함정들

- **날짜가 OADate 시리얼(예: 46203)로 나옴** → `[DateTime]::FromOADate([double]$v)`로 변환.
- **홈택스 .xls는 HTML 위장인 경우가 많다**던데, 이번 파일은 진짜 OLE 바이너리(헤더 `D0 CF 11 E0`)였다. 먼저 헤더 몇 바이트를 확인하고 경로를 정할 것.
- **한글 CSV는 UTF-8 BOM으로 저장**해야 Excel에서 안 깨짐: `New-Object System.Text.UTF8Encoding $true`.
- **멀티라인 헤더가 든 CSV 재파싱**은 직접 split 말고 `Microsoft.VisualBasic.FileIO.TextFieldParser` 사용 (따옴표 안 줄바꿈 처리됨):

```powershell
Add-Type -AssemblyName Microsoft.VisualBasic
$parser = New-Object Microsoft.VisualBasic.FileIO.TextFieldParser($path, [System.Text.Encoding]::UTF8)
$parser.TextFieldType = 'Delimited'; $parser.SetDelimiters(','); $parser.HasFieldsEnclosedInQuotes = $true
while (-not $parser.EndOfData) { $row = $parser.ReadFields() }
```

### 3. 사업자등록번호 추출·매칭 패턴

- 정규식 `^\d{3}-\d{2}-\d{5}$`로 걸러내면 헤더·빈칸·주민번호(형식이 다름)가 자연히 제외된다.
- 자사 번호는 명시적으로 제외할 것.
- 상호명 매칭은 금물 — 전각 괄호 `（주）`, 지사명 붙음("한국전력공사 완도지사"), 카드가맹점명 등 표기가 제각각.

## 관련 문서

- [[skill-pdf-extract-and-pasted-image]] — 이 PC 도구 상태(pdftotext 있음, python/poppler 없음)
- [[handover-vat-2026h1-bizno-match]] — 이 스킬을 쓴 실제 작업
