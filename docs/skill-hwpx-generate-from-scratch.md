# 스킬: 템플릿 없이 마크다운 → hwpx 신규 생성 (한글 열림 확인, 스크립트 전문 수록)

**작성일:** 2026-07-24 (시민기업펀드 계획서에서 검증)

## 배경 / 기존 스킬과의 차이

- `docs/skill-hwpx-generate-from-template.md`는 **기존 hwpx의 서식을 재사용**하는 방식이다. 템플릿 hwpx가 있어야 한다.
- 이번엔 **템플릿이 전혀 없었다**(시민기금은 PDF로만 받음). 그래서 OWPML 패키지 전체를 **처음부터 생성**했다.
- **결과: 한글에서 정상 열림 확인.** 사용자가 생성물을 한글로 열어 편집·재저장했고, 재업로드 파일에 한글이 추가하는 `Preview/PrvImage.png`·`META-INF/container.rdf`가 붙어 있었다(= 한글이 열고 다시 썼다는 증거).

## 핵심 포인트

- **패키지 엔트리(순서 중요)**: `mimetype`(반드시 첫 엔트리·Stored·`application/hwp+zip`) → `version.xml` → `Contents/content.hpf` → `Contents/header.xml` → `Contents/section0.xml` → `settings.xml` → `META-INF/container.xml` → `META-INF/manifest.xml` → `Preview/PrvText.txt`.
- **header.xml에 refList 전부 정의**: fontfaces(7개 lang: HANGUL/LATIN/HANJA/JAPANESE/OTHER/SYMBOL/USER 각각 font 목록), borderFills, charProperties, tabProperties(1개라도), paraProperties, styles(최소 1개). `charPr`/`paraPr`/`borderFill`이 참조(IDRef)하는 id가 **반드시 존재**해야 한다(tabPrIDRef="0"도 tabProperties 필요).
- **section0.xml 첫 문단**: 첫 `hp:p`의 첫 `hp:run` 안에 `hp:secPr`를 넣고 그 뒤에 `hp:t`. secPr에는 grid/startNum/visibility/lineNumberShape/pagePr(margin)/footNotePr/endNotePr/pageBorderFill를 넣었다.
- **linesegarray 포함**(대략값): 전 문단에 넣었다. 값은 열 때 재계산되므로 대략이면 된다(템플릿 스킬의 교훈과 동일). 표 안 셀 문단에도 넣었다.
- **표**: `hp:tbl`(rowCnt/colCnt/borderFillIDRef) → `hp:tr` → `hp:tc`(borderFillIDRef) → 셀 안은 `hp:subList` → `hp:p`. `hp:tc` 자식 **순서**: subList, cellAddr, cellSpan, cellSz, cellMargin. 표는 `treatAsChar="1"`로 문단에 인라인.
- **단위**: 1 HWPUNIT = 1/7200 inch. A4 세로 width=59528 height=84188. 본문 폭 ≈ 42520(좌우 여백 8504씩).
- **zip**: node `zlib.crc32()` 내장(v20.15+/22.2+). mimetype만 Stored(method 0), 나머지 deflateRaw. 버전메이드 0x5AEF(한컴 관례) 사용.
- **검증(한글 없이)**: python `zipfile`로 `testzip()`(CRC 전수) + `xml.dom.minidom.parseString`(전 XML well-formed) + 태그 균형(hp:p/hp:tbl/hp:tr/hp:tc open=close). 이 3개를 통과시키고 나서 전달.

## 함정

- **PDF 추출**: 이 계열 환경엔 poppler(pdftotext) 없음. `pip install pymupdf`(import fitz)가 가장 확실. `pypdf`는 cryptography 바인딩(`_cffi_backend`) 깨져서 import 실패.
- 구조검증을 통과해도 **"한글이 실제로 여는지"는 100% 보장 아님**(사소한 필수 자식 누락으로 손상 거부 가능). 열림 미검증 시 그 사실을 사용자에게 명시하고, 안 열리면 **사용자 hwpx를 템플릿으로 받아** 템플릿 방식으로 전환하는 게 확실.
- 표가 많은 문서일수록 셀 자식 순서·borderFill 참조가 깨지기 쉽다. 위 순서를 지킬 것.

## build_hwpx.js — 전문

```javascript
'use strict';
const fs = require('fs');
const zlib = require('zlib');

const MD = process.argv[2];
const OUT = process.argv[3];
const src = fs.readFileSync(MD, 'utf8');

function esc(s){return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;').replace(/'/g,'&apos;');}
function cleanInline(s){
  return s.replace(/\*\*(.+?)\*\*/g,'$1').replace(/`([^`]+)`/g,'$1').replace(/\[([^\]]+)\]\([^)]*\)/g,'$1').trim();
}

// ---- parse markdown into blocks ----
const lines = src.split(/\r?\n/);
const blocks = [];
let i = 0, sawH1 = false, inCode = false;
while (i < lines.length){
  let ln = lines[i];
  if (/^```/.test(ln)){ inCode = !inCode; i++; continue; }
  if (inCode){ if(ln.trim()) blocks.push({t:'body', s:cleanInline(ln)}); i++; continue; }
  ln = ln.replace(/\s+$/,'');
  if (ln.trim()===''){ i++; continue; }
  if (/^#\s+/.test(ln)){ blocks.push({t:'title', s:cleanInline(ln.replace(/^#\s+/,''))}); sawH1=true; i++; continue; }
  if (/^##\s+/.test(ln)){ blocks.push({t:'h2', s:cleanInline(ln.replace(/^##\s+/,''))}); i++; continue; }
  if (/^###\s+/.test(ln)){ blocks.push({t:'h3', s:cleanInline(ln.replace(/^###\s+/,''))}); i++; continue; }
  if (/^---+$/.test(ln.trim())){ blocks.push({t:'hr'}); i++; continue; }
  if (/^>\s?/.test(ln)){ blocks.push({t:'quote', s:cleanInline(ln.replace(/^>\s?/,''))}); i++; continue; }
  if (/^\s*\|.*\|\s*$/.test(ln)){
    const tbl=[];
    while(i<lines.length && /^\s*\|.*\|\s*$/.test(lines[i])){ tbl.push(lines[i]); i++; }
    const rows=[];
    for(const r of tbl){
      if (/^\s*\|?[\s:\-|]+\|?\s*$/.test(r)) continue; // separator row
      const cells = r.trim().replace(/^\|/,'').replace(/\|$/,'').split('|').map(c=>cleanInline(c.trim()));
      rows.push(cells);
    }
    if(rows.length){ blocks.push({t:'table', rows}); }
    continue;
  }
  let m;
  if ((m=/^(\s*)([-*])\s+(.*)$/.exec(ln))){ blocks.push({t:'bullet', s:cleanInline(m[3]), indent:Math.floor(m[1].length/2)}); i++; continue; }
  if ((m=/^(\s*)(\d+)\.\s+(.*)$/.exec(ln))){ blocks.push({t:'num', s:cleanInline(m[3]), n:m[2], indent:Math.floor(m[1].length/2)}); i++; continue; }
  if(!sawH1){ blocks.push({t:'subtitle', s:cleanInline(ln)}); i++; continue; }
  blocks.push({t:'body', s:cleanInline(ln)}); i++;
}

// ---- XML building ----
const NS_HH='http://www.hancom.co.kr/hwpml/2011/head';
const NS_HP='http://www.hancom.co.kr/hwpml/2011/paragraph';
const NS_HS='http://www.hancom.co.kr/hwpml/2011/section';
const NS_HC='http://www.hancom.co.kr/hwpml/2011/core';
const TEXTW = 42520; // 본문 폭 (hwpunit)

function linesegs(text, size, spacing, width, startY){
  const cpl = Math.max(1, Math.floor(width/ Math.max(600,size*0.95)));
  const n = Math.max(1, Math.ceil((text.length||1)/cpl));
  let y = startY||0; const segs=[];
  for(let k=0;k<n;k++){
    segs.push(`<hp:lineseg textpos="${k*cpl}" vertpos="${y}" vertsize="${size}" textheight="${size}" baseline="${Math.round(size*0.85)}" spacing="${spacing}" horzpos="0" horzsize="${width}" flags="393216"/>`);
    y += size + spacing;
  }
  return {xml:`<hp:linesegarray>${segs.join('')}</hp:linesegarray>`, next:y};
}
let gy = 0;
function P(text, charPr, paraPr, size, spacing){
  const seg = linesegs(text||' ', size, spacing, TEXTW, gy); gy = seg.next + spacing;
  const run = text!=='' ? `<hp:run charPrIDRef="${charPr}"><hp:t>${esc(text)}</hp:t></hp:run>` : `<hp:run charPrIDRef="${charPr}"><hp:t></hp:t></hp:run>`;
  return `<hp:p id="0" paraPrIDRef="${paraPr}" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0">${run}${seg.xml}</hp:p>`;
}
function cellP(text, charPr, paraPr, size, width){
  const seg = linesegs(text||' ', size, 600, Math.max(2000,width-560), 0);
  return `<hp:p id="0" paraPrIDRef="${paraPr}" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0"><hp:run charPrIDRef="${charPr}"><hp:t>${esc(text||'')}</hp:t></hp:run>${seg.xml}</hp:p>`;
}
let tblId = 0;
function TABLE(rows){
  const nCols = Math.max(...rows.map(r=>r.length)), nRows = rows.length;
  const colW = Math.floor(TEXTW / nCols), rowH = 2800, totalH = rowH*nRows;
  let tr='';
  for(let r=0;r<nRows;r++){
    let tcs='';
    for(let c=0;c<nCols;c++){
      const isH = r===0, bf = isH?4:3, cp = isH?5:0, pp = isH?4:3;
      const txt = rows[r][c]!==undefined?rows[r][c]:'';
      tcs += `<hp:tc name="" header="${isH?1:0}" hasMargin="0" protect="0" editable="0" dirty="0" borderFillIDRef="${bf}">`
        + `<hp:subList id="" textDirection="HORIZONTAL" lineWrap="BREAK" vertAlign="CENTER" linkListIDRef="0" linkListNextIDRef="0" textWidth="0" textHeight="0" hasTextRef="0" hasNumRef="0">`
        + cellP(txt, cp, pp, 900, colW) + `</hp:subList>`
        + `<hp:cellAddr colAddr="${c}" rowAddr="${r}"/><hp:cellSpan colSpan="1" rowSpan="1"/>`
        + `<hp:cellSz width="${colW}" height="${rowH}"/><hp:cellMargin left="280" right="280" top="140" bottom="140"/></hp:tc>`;
    }
    tr += `<hp:tr>${tcs}</hp:tr>`;
  }
  const seg = linesegs(' ', 1000, 600, TEXTW, gy); gy += totalH + 600; tblId++;
  const tbl = `<hp:tbl id="${tblId}" zOrder="0" numberingType="NONE" textWrap="TOP_AND_BOTTOM" textFlow="BOTH_SIDES" lock="0" dropcapstyle="None" pageBreak="CELL" repeatHeader="1" rowCnt="${nRows}" colCnt="${nCols}" cellSpacing="0" borderFillIDRef="3" noAdjust="0">`
    + `<hp:sz width="${TEXTW}" widthRelTo="ABSOLUTE" height="${totalH}" heightRelTo="ABSOLUTE" protect="0"/>`
    + `<hp:pos treatAsChar="1" affectLSpacing="0" flowWithText="1" allowOverlap="0" holdAnchorAndSO="0" vertRelTo="PARA" horzRelTo="COLUMN" vertAlign="TOP" horzAlign="LEFT" vertOffset="0" horzOffset="0"/>`
    + `<hp:outMargin left="0" right="0" top="0" bottom="283"/><hp:inMargin left="0" right="0" top="0" bottom="0"/>`
    + tr + `</hp:tbl>`;
  return `<hp:p id="0" paraPrIDRef="0" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0"><hp:run charPrIDRef="0">${tbl}</hp:run>${seg.xml}</hp:p>`;
}

const secPr = `<hp:secPr id="" textDirection="HORIZONTAL" spaceColumns="1134" tabStop="8000" tabStopVal="4000" tabStopUnit="HWPUNIT" outlineShapeIDRef="1" memoShapeIDRef="0" textVerticalWidthHead="0" masterPageCnt="0">`
  + `<hp:grid lineGrid="0" charGrid="0" wonggojiFormat="0" strtnum="0" strtnumType="1"/>`
  + `<hp:startNum pageStartsOn="BOTH" page="0" pic="0" tbl="0" equation="0"/>`
  + `<hp:visibility hideFirstHeader="0" hideFirstFooter="0" hideFirstMasterPage="0" border="SHOW_ALL" fill="SHOW_ALL" hideFirstPageNum="0" hideFirstEmptyLine="0" showLineNumber="0"/>`
  + `<hp:lineNumberShape restartType="0" countBy="0" distance="0" startNumber="0"/>`
  + `<hp:pagePr landscape="WIDELY" width="59528" height="84188" gutterType="LEFT_ONLY"><hp:margin header="4252" footer="4252" gutter="0" left="8504" right="8504" top="5669" bottom="4252"/></hp:pagePr>`
  + `<hp:footNotePr><hp:autoNumFormat type="DIGIT" userChar="" prefixChar="" suffixChar=")" supscript="0"/><hp:noteLine length="-1" type="SOLID" width="0.12 mm" color="#000000"/><hp:noteSpacing betweenNotes="850" belowLine="567" aboveLine="567"/><hp:numbering type="CONTINUOUS" newNum="1"/><hp:placement place="EACH_COLUMN" beneathText="0"/></hp:footNotePr>`
  + `<hp:endNotePr><hp:autoNumFormat type="DIGIT" userChar="" prefixChar="" suffixChar=")" supscript="0"/><hp:noteLine length="14692344" type="SOLID" width="0.12 mm" color="#000000"/><hp:noteSpacing betweenNotes="0" belowLine="567" aboveLine="850"/><hp:numbering type="CONTINUOUS" newNum="1"/><hp:placement place="END_OF_DOCUMENT" beneathText="0"/></hp:endNotePr>`
  + `<hp:pageBorderFill type="BOTH" borderFillIDRef="1" textBorder="PAPER" headerInside="0" footerInside="0" fillArea="PAPER"><hp:offset left="1417" right="1417" top="1417" bottom="1417"/></hp:pageBorderFill>`
  + `<hp:pageBorderFill type="EVEN" borderFillIDRef="1" textBorder="PAPER" headerInside="0" footerInside="0" fillArea="PAPER"><hp:offset left="1417" right="1417" top="1417" bottom="1417"/></hp:pageBorderFill>`
  + `<hp:pageBorderFill type="ODD" borderFillIDRef="1" textBorder="PAPER" headerInside="0" footerInside="0" fillArea="PAPER"><hp:offset left="1417" right="1417" top="1417" bottom="1417"/></hp:pageBorderFill>`
  + `</hp:secPr>`;

const body = []; let first = true;
for (const b of blocks){
  if (b.t==='hr'){ continue; }
  if (first){
    // 첫 문단(어떤 블록이든)에 secPr 부착
    const s = b.s||' ', isTitle = b.t==='title';
    const cp = isTitle?2:3, pp=1, size=isTitle?2000:1100, sp=isTitle?900:700;
    const seg = linesegs(s, size, sp, TEXTW, gy); gy = seg.next+sp;
    body.push(`<hp:p id="0" paraPrIDRef="${pp}" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0"><hp:run charPrIDRef="${cp}">${secPr}<hp:t>${esc(s)}</hp:t></hp:run>${seg.xml}</hp:p>`);
    first=false; continue;
  }
  switch(b.t){
    case 'title': body.push(P(b.s,2,1,2000,900)); break;
    case 'subtitle': body.push(P(b.s,3,1,1100,700)); break;
    case 'h2': body.push(P('')); body.push(P(b.s,1,2,1400,700)); break;
    case 'h3': body.push(P(b.s,7,2,1150,600)); break;
    case 'quote': body.push(P('  '+b.s,8,0,950,600)); break;
    case 'bullet': body.push(P('   '.repeat(b.indent||0)+'· '+b.s,0,0,1000,600)); break;
    case 'num': body.push(P('   '.repeat(b.indent||0)+b.n+'. '+b.s,0,0,1000,600)); break;
    case 'table': body.push(TABLE(b.rows)); break;
    default: body.push(P(b.s,0,0,1000,600)); break;
  }
}
if(!body.length){ body.push(P(' ',0,0,1000,600)); }

const section0 = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>\n`
  + `<hs:sec xmlns:hs="${NS_HS}" xmlns:hp="${NS_HP}" xmlns:hc="${NS_HC}" xmlns:hh="${NS_HH}">` + body.join('') + `</hs:sec>`;

// header.xml
function fontface(lang){
  return `<hh:fontface lang="${lang}" fontCnt="2">`
    + `<hh:font id="0" face="함초롬바탕" type="TTF" isEmbedded="0"><hh:typeInfo familyType="FCAT_MYUNGJO" weight="0" proportion="0" contrast="0" strokeVariation="0" armStyle="0" letterform="0" midline="0" xHeight="0"/></hh:font>`
    + `<hh:font id="1" face="함초롬돋움" type="TTF" isEmbedded="0"><hh:typeInfo familyType="FCAT_GOTHIC" weight="0" proportion="0" contrast="0" strokeVariation="0" armStyle="0" letterform="0" midline="0" xHeight="0"/></hh:font>`
    + `</hh:fontface>`;
}
const langs=['HANGUL','LATIN','HANJA','JAPANESE','OTHER','SYMBOL','USER'];
const fontfaces = `<hh:fontfaces itemCnt="${langs.length}">` + langs.map(fontface).join('') + `</hh:fontfaces>`;
function noB(){return `<hh:leftBorder type="NONE" width="0.1 mm" color="#000000"/><hh:rightBorder type="NONE" width="0.1 mm" color="#000000"/><hh:topBorder type="NONE" width="0.1 mm" color="#000000"/><hh:bottomBorder type="NONE" width="0.1 mm" color="#000000"/><hh:diagonal type="SOLID" width="0.1 mm" color="#000000"/>`;}
function soB(){return `<hh:leftBorder type="SOLID" width="0.12 mm" color="#000000"/><hh:rightBorder type="SOLID" width="0.12 mm" color="#000000"/><hh:topBorder type="SOLID" width="0.12 mm" color="#000000"/><hh:bottomBorder type="SOLID" width="0.12 mm" color="#000000"/><hh:diagonal type="SOLID" width="0.1 mm" color="#000000"/>`;}
const borderFills = `<hh:borderFills itemCnt="4">`
  + `<hh:borderFill id="1" threeD="0" shadow="0" centerLine="NONE" breakCellSeparateLine="0">${noB()}<hh:slash type="NONE" Crooked="0" isCounter="0"/><hh:backSlash type="NONE" Crooked="0" isCounter="0"/></hh:borderFill>`
  + `<hh:borderFill id="2" threeD="0" shadow="0" centerLine="NONE" breakCellSeparateLine="0">${noB()}<hh:slash type="NONE" Crooked="0" isCounter="0"/><hh:backSlash type="NONE" Crooked="0" isCounter="0"/></hh:borderFill>`
  + `<hh:borderFill id="3" threeD="0" shadow="0" centerLine="NONE" breakCellSeparateLine="0">${soB()}<hh:slash type="NONE" Crooked="0" isCounter="0"/><hh:backSlash type="NONE" Crooked="0" isCounter="0"/></hh:borderFill>`
  + `<hh:borderFill id="4" threeD="0" shadow="0" centerLine="NONE" breakCellSeparateLine="0">${soB()}<hh:slash type="NONE" Crooked="0" isCounter="0"/><hh:backSlash type="NONE" Crooked="0" isCounter="0"/><hh:fillBrush><hh:winBrush faceColor="#E6E6E6" hatchColor="#000000" alpha="0"/></hh:fillBrush></hh:borderFill>`
  + `</hh:borderFills>`;
function charPr(id, height, o){o=o||{}; const f=o.gothic?1:0;
  return `<hh:charPr id="${id}" height="${height}" textColor="${o.color||'#000000'}" shadeColor="none" useFontSpace="0" useKerning="0" symMark="NONE" borderFillIDRef="2">`
    + `<hh:fontRef hangul="${f}" latin="${f}" hanja="${f}" japanese="${f}" other="${f}" symbol="${f}" user="${f}"/>`
    + `<hh:ratio hangul="100" latin="100" hanja="100" japanese="100" other="100" symbol="100" user="100"/>`
    + `<hh:spacing hangul="0" latin="0" hanja="0" japanese="0" other="0" symbol="0" user="0"/>`
    + `<hh:relSz hangul="100" latin="100" hanja="100" japanese="100" other="100" symbol="100" user="100"/>`
    + `<hh:offset hangul="0" latin="0" hanja="0" japanese="0" other="0" symbol="0" user="0"/>`
    + (o.bold?`<hh:bold/>`:``) + `</hh:charPr>`;}
const charProperties = `<hh:charProperties itemCnt="9">`
  + charPr(0,1000,{}) + charPr(1,1400,{gothic:1,bold:1}) + charPr(2,2000,{gothic:1,bold:1})
  + charPr(3,1100,{gothic:1}) + charPr(4,800,{color:'#555555'}) + charPr(5,1000,{gothic:1,bold:1})
  + charPr(6,1000,{bold:1}) + charPr(7,1200,{gothic:1,bold:1}) + charPr(8,950,{color:'#333333'})
  + `</hh:charProperties>`;
function paraPr(id, align, linePct, before, after){
  return `<hh:paraPr id="${id}" tabPrIDRef="0" condense="0" fontLineHeight="0" snapToGrid="1" suppressLineNumbers="0" checked="0">`
    + `<hh:align horizontal="${align}" vertical="BASELINE"/><hh:heading type="NONE" idRef="0" level="0"/>`
    + `<hh:breakSetting breakLatinWord="KEEP_WORD" breakNonLatinWord="BREAK_WORD" widowOrphan="0" keepWithNext="0" keepLines="0" pageBreakBefore="0" lineWrap="BREAK"/>`
    + `<hh:autoSpacing eAsianEng="0" eAsianNum="0"/>`
    + `<hh:margin><hc:intent value="0" unit="HWPUNIT"/><hc:left value="0" unit="HWPUNIT"/><hc:right value="0" unit="HWPUNIT"/><hc:prev value="${before||0}" unit="HWPUNIT"/><hc:next value="${after||0}" unit="HWPUNIT"/></hh:margin>`
    + `<hh:lineSpacing type="PERCENT" value="${linePct||160}" unit="HWPUNIT"/>`
    + `<hh:border borderFillIDRef="1" offsetLeft="0" offsetRight="0" offsetTop="0" offsetBottom="0" connect="0" ignoreMargin="0"/></hh:paraPr>`;
}
const paraProperties = `<hh:paraProperties itemCnt="5">`
  + paraPr(0,'JUSTIFY',165,0,0) + paraPr(1,'CENTER',150,0,300) + paraPr(2,'LEFT',150,300,150)
  + paraPr(3,'LEFT',140,0,0) + paraPr(4,'CENTER',140,0,0) + `</hh:paraProperties>`;
const tabProperties = `<hh:tabProperties itemCnt="1"><hh:tabPr id="0" autoTabLeft="0" autoTabRight="0"/></hh:tabProperties>`;
const styles = `<hh:styles itemCnt="1"><hh:style id="0" type="PARA" name="바탕글" engName="Normal" paraPrIDRef="0" charPrIDRef="0" nextStyleIDRef="0" langID="1042" lockForm="0"/></hh:styles>`;
const header = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>\n`
  + `<hh:head xmlns:hh="${NS_HH}" xmlns:hp="${NS_HP}" xmlns:hc="${NS_HC}" version="1.4" secCnt="1">`
  + `<hh:beginNum page="1" footnote="1" endnote="1" pic="1" tbl="1" equation="1"/>`
  + `<hh:refList>` + fontfaces + borderFills + charProperties + tabProperties + paraProperties + styles + `</hh:refList>`
  + `<hh:compatibleDocument targetProgram="HWP201X"><hh:layoutCompatibility/></hh:compatibleDocument></hh:head>`;

// 기타 패키지 파일
const version = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>\n<hv:HCFVersion xmlns:hv="http://www.hancom.co.kr/hwpml/2011/version" tagetApplication="WORDPROCESSOR" major="5" minor="0" micro="5" buildNumber="0" os="1" xmlVersion="1.4" application="Hancom Office Hangul" appVersion="9, 0, 0, 0"/>`;
const settings = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>\n<ha:HWPApplicationSetting xmlns:ha="http://www.hancom.co.kr/hwpml/2011/app"><ha:CaretPosition listIDRef="0" paraIDRef="0" pos="0"/></ha:HWPApplicationSetting>`;
const container = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>\n<ocf:container xmlns:ocf="urn:oasis:names:tc:opendocument:xmlns:container" xmlns:hpf="http://www.hancom.co.kr/schema/2011/hpf"><ocf:rootfiles><ocf:rootfile full-path="Contents/content.hpf" media-type="application/hwpml-package+xml"/></ocf:rootfiles></ocf:container>`;
const manifest = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>\n<odf:manifest xmlns:odf="urn:oasis:names:tc:opendocument:xmlns:manifest:1.0" version="1.2"><odf:file-entry full-path="/" media-type="application/hwp+zip"/><odf:file-entry full-path="Contents/header.xml" media-type="application/xml"/><odf:file-entry full-path="Contents/section0.xml" media-type="application/xml"/></odf:manifest>`;
const contentHpf = `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>\n<hpf:HWPML xmlns:hpf="http://www.hancom.co.kr/schema/2011/hpf" xmlns:opf="http://www.idpf.org/2007/opf/" xmlns:dc="http://purl.org/dc/elements/1.1/" version="1.4"><opf:package version="1.4" unique-identifier="" id=""><opf:metadata><opf:title>문서</opf:title><opf:language>ko</opf:language></opf:metadata><opf:manifest><opf:item id="header" href="Contents/header.xml" media-type="application/xml"/><opf:item id="section0" href="Contents/section0.xml" media-type="application/xml"/><opf:item id="settings" href="settings.xml" media-type="application/xml"/></opf:manifest><opf:spine><opf:itemref idref="section0" linear="yes"/></opf:spine></opf:package></hpf:HWPML>`;

// zip (mimetype 첫 엔트리·Stored)
function writeZip(order, data){
  const chunks=[], central=[]; let offset=0;
  for(const name of order){
    const content = Buffer.isBuffer(data[name])?data[name]:Buffer.from(data[name],'utf8');
    const stored = name==='mimetype';
    const comp = stored?content:zlib.deflateRawSync(content,{level:9});
    const method = stored?0:8, crc = zlib.crc32(content)>>>0, nameBuf = Buffer.from(name,'utf8');
    const lh=Buffer.alloc(30);
    lh.writeUInt32LE(0x04034b50,0); lh.writeUInt16LE(20,4); lh.writeUInt16LE(0,6); lh.writeUInt16LE(method,8);
    lh.writeUInt16LE(0,10); lh.writeUInt16LE(0x5AEF,12); lh.writeUInt32LE(crc,14); lh.writeUInt32LE(comp.length,18);
    lh.writeUInt32LE(content.length,22); lh.writeUInt16LE(nameBuf.length,26); lh.writeUInt16LE(0,28);
    chunks.push(lh,nameBuf,comp);
    const ch=Buffer.alloc(46);
    ch.writeUInt32LE(0x02014b50,0); ch.writeUInt16LE(20,4); ch.writeUInt16LE(20,6); ch.writeUInt16LE(method,10);
    ch.writeUInt16LE(0x5AEF,14); ch.writeUInt32LE(crc,16); ch.writeUInt32LE(comp.length,20); ch.writeUInt32LE(content.length,24);
    ch.writeUInt16LE(nameBuf.length,28); ch.writeUInt32LE(offset,42);
    central.push(Buffer.concat([ch,nameBuf])); offset += lh.length+nameBuf.length+comp.length;
  }
  const cd=Buffer.concat(central), eocd=Buffer.alloc(22);
  eocd.writeUInt32LE(0x06054b50,0); eocd.writeUInt16LE(order.length,8); eocd.writeUInt16LE(order.length,10);
  eocd.writeUInt32LE(cd.length,12); eocd.writeUInt32LE(offset,16);
  return Buffer.concat([...chunks,cd,eocd]);
}
const data = {
  'mimetype':'application/hwp+zip', 'version.xml':version, 'settings.xml':settings,
  'Contents/header.xml':header, 'Contents/section0.xml':section0, 'Contents/content.hpf':contentHpf,
  'META-INF/container.xml':container, 'META-INF/manifest.xml':manifest, 'Preview/PrvText.txt':'문서',
};
const order = ['mimetype','version.xml','Contents/content.hpf','Contents/header.xml','Contents/section0.xml','settings.xml','META-INF/container.xml','META-INF/manifest.xml','Preview/PrvText.txt'];
fs.writeFileSync(OUT, writeZip(order, data));
console.log('WROTE', OUT, fs.statSync(OUT).size, 'bytes');
```

## 검증 스크립트 (python)

```python
import zipfile, xml.dom.minidom as M
z=zipfile.ZipFile("out.hwpx")
assert z.namelist()[0]=="mimetype" and z.getinfo("mimetype").compress_type==0
for n in z.namelist():
    if n.endswith((".xml",".hpf")): M.parseString(z.read(n))  # well-formed
assert z.testzip() is None  # CRC 전수
print("PASS")
```
