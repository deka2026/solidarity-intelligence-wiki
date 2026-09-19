# 스킬: pptxgenjs로 PPTX 생성 + 한쇼(Hancom Show) 호환 — python 없는 Windows PC

**작성일:** 2026-07-16

## 배경

이 PC는 python(스토어 스텁만)·LibreOffice가 없어 python-pptx나 soffice 검증 스크립트를 못 쓴다. Node(v24)는 있으므로 **pptxgenjs가 PPTX 생성의 주 경로**다. 단, pptxgenjs 산출물을 한쇼로 열면 "일부 텍스트, 이미지, 개체 등이 손상된 문서" 경고가 뜬다 — PowerPoint에서는 멀쩡히 열리는 파일이라도.

## 설치와 기본

```powershell
npm install -g pptxgenjs
```

전역 설치 모듈은 스크립트에서 절대경로로 require:

```js
const pptxgen = require("C:/Users/pc/AppData/Roaming/npm/node_modules/pptxgenjs");
const pres = new pptxgen();
pres.layout = "LAYOUT_WIDE"; // 13.33 x 7.5 인치. 기본값은 10 x 5.625라 반드시 슬라이드 추가 전에 설정
```

- 색상은 `#` 없는 6자리 hex만 (`"FF0000"`). `#` 붙이거나 8자리면 파일 깨짐.
- 한국어 폰트는 `fontFace: "Malgun Gothic"` — 한쇼·PowerPoint 양쪽 안전.
- jszip이 pptxgenjs 안에 동봉됨: `require(".../pptxgenjs/node_modules/jszip")` — 후처리에 재활용.

## 한쇼 "손상된 문서" 원인과 해결

**원인**: pptxgenjs는 노트를 안 써도 (1) 모든 슬라이드에 빈 `notesSlideN.xml`, (2) `notesMaster1.xml`을 만들고, (3) `presentation.xml`에 `<p:notesMasterIdLst>`를 `<p:sldIdLst>` **뒤에** 기록한다. ECMA-376 스키마는 notesMasterIdLst가 sldIdLst **앞**이어야 한다. PowerPoint는 관대하지만 **한쇼는 스키마 위반을 손상으로 판정**한다.

**해결**: 생성 직후 jszip으로 노트 부품을 통째로 제거 (요소를 재배치하는 것보다 안전 — pptxgenjs의 노트마스터는 슬라이드마스터와 테마를 공유해서 순서만 옮기면 PowerPoint 쪽이 깨질 수 있음):

```js
async function stripNotes(file) {
  const zip = await JSZip.loadAsync(fs.readFileSync(file));
  let pres = await zip.file("ppt/presentation.xml").async("string");
  zip.file("ppt/presentation.xml", pres.replace(/<p:notesMasterIdLst>[\s\S]*?<\/p:notesMasterIdLst>/, ""));
  let prels = await zip.file("ppt/_rels/presentation.xml.rels").async("string");
  zip.file("ppt/_rels/presentation.xml.rels", prels.replace(/<Relationship [^>]*notesMaster[^>]*\/>/g, ""));
  let ct = await zip.file("[Content_Types].xml").async("string");
  zip.file("[Content_Types].xml", ct.replace(/<Override [^>]*notes(Master|Slide)[^>]*\/>/g, ""));
  for (const name of Object.keys(zip.files).filter((n) => /^ppt\/slides\/_rels\/slide\d+\.xml\.rels$/.test(n))) {
    let xml = await zip.file(name).async("string");
    zip.file(name, xml.replace(/<Relationship [^>]*notesSlide[^>]*\/>/g, ""));
  }
  Object.keys(zip.files)
    .filter((n) => n.startsWith("ppt/notesMasters/") || n.startsWith("ppt/notesSlides/"))
    .forEach((n) => zip.remove(n));
  let app = await zip.file("docProps/app.xml").async("string");
  zip.file("docProps/app.xml", app.replace(/<Notes>\d+<\/Notes>/, "<Notes>0</Notes>"));
  fs.writeFileSync(file, await zip.generateAsync({ type: "nodebuffer", compression: "DEFLATE" }));
}
```

노트가 실제로 필요하면(`slide.addNotes`) 이 방법은 못 쓰고 notesMasterIdLst를 sldIdLst 앞으로 옮기는 편집을 검토해야 한다 — 미검증.

## 파일 잠금(EBUSY) 대응

사용자가 한쇼로 대상 파일을 열어두면 `writeFile`/`copyFileSync`가 EBUSY로 실패한다. **temp에 생성 → copy 시도 → 실패 시 대체 이름으로 저장** 패턴:

```js
await makeDeck().writeFile({ fileName: tmpPath });
await stripNotes(tmpPath);
try { fs.copyFileSync(tmpPath, dstPath); }
catch (e) { fs.copyFileSync(tmpPath, altPath); /* 사용자에게 최신 파일명 알릴 것 */ }
```

## python 없이 구조 검증 (PowerShell)

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem
$z=[System.IO.Compression.ZipFile]::OpenRead($pptx)
foreach ($e in ($z.Entries | Where-Object {$_.FullName -match "\.(xml|rels)$"})) {
  $sr=New-Object System.IO.StreamReader($e.Open()); $txt=$sr.ReadToEnd(); $sr.Close()
  $x=New-Object System.Xml.XmlDocument; $x.LoadXml($txt)   # 실패하면 XML 깨진 것
  if ($txt -match "notesSlide|notesMaster") { "노트 잔재: " + $e.FullName }
}
$z.Dispose()
```

well-formed 여부와 참조 잔재만 잡는 최소 검증이다. 스키마 검증은 이 PC에서 불가 — 최종 확인은 실제 뷰어(한쇼)로.

## 실전 사례

`C:\Users\pc\클로드로컬\pptx-scripts\gen_final.js` / `gen_light.js` — 다크/라이트 테마 교육자료 4종 생성 전 과정 (생성 → stripNotes → 잠금 대응까지). [[handover-youth-policy-pptx]] 참조.
