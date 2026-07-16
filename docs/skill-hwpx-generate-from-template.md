# 스킬: 기존 hwpx를 템플릿 삼아 새 문서를 node로 생성 (스크립트 전문 수록)

**작성일:** 2026-07-16 (2026-07-15 완도군 사업계획서에서 재검증)

## 배경

이 PC는 python이 스토어 스텁뿐이고 한글 COM도 미등록이라, hwpx 신규 생성은 **기존 hwpx의 서식을 재사용하는 node 직접 조작**이 유일한 경로다. 7/15 이전 세션에서 검증한 방법인데 스크립트가 세션 스크래치패드와 함께 사라져 재작성했다. **이번엔 전문을 여기 수록한다.**

## 절차 요약

1. 템플릿 hwpx의 zip 엔트리를 파싱해 `Contents/section0.xml` 추출
2. 문단을 깊이 추적으로 분리 → 각 스타일(제목/소제목/본문)의 `charPrIDRef`/`paraPrIDRef` 파악
3. **P0(첫 top-level 문단, secPr 포함)는 통째로 보존**하고 `<hp:t>` 텍스트만 교체
4. 새 문단들을 템플릿 문단과 같은 속성 + **linesegarray(대략값)** 로 생성해 이어붙임
5. mimetype 선두·Stored 규칙으로 zip 재포장

## 핵심 함정 (재확인됨)

- 모든 `<hp:p>`에 `<hp:linesegarray>` 없으면 한글이 "손상" 거부. 값은 대략만 맞으면 됨(열 때 재계산). 긴 문단은 `ceil(len/46)`개 lineseg를 넣었더니 문제없음
- P0 경계는 정규식 한 방이 아니라 `<hp:p\b`/`</hp:p>` **깊이 추적**으로 찾을 것 (머리말 ctrl에 nested hp:p가 있을 수 있음)
- **node v24는 `zlib.crc32()` 내장** (Node 20.15+/22.2+) — 예전 pack-hwpx.js의 수동 CRC 테이블 불필요
- .NET NoCompression은 Deflate로 기록되므로 zip은 node로 직접 써야 함 (아래 writeZip)
- `Preview/PrvText.txt`는 원본 인코딩(UTF-16LE BOM 여부) 감지해서 맞춰 갱신, PrvImage.png는 그대로 둬도 무해(미리보기만 구식)

## 검증 체크리스트 (한글 없이)

1. 첫 로컬헤더: 시그니처 `0x04034b50`, method=0(Stored), 이름 `mimetype`
2. 전 엔트리 압축 해제 후 `zlib.crc32` 재계산 = 중앙디렉터리 CRC
3. section0.xml에서 `<hp:p` 열림/`</hp:p>` 닫힘 개수 일치, linesegarray 개수 = 문단 수
4. 모든 XML을 PowerShell `[System.Xml.XmlDocument]::Load()`로 well-formed 확인
   - 주의: bash에서 `node -e "..." && powershell -Command "..."` 체이닝은 인용이 깨진다. PowerShell은 별도 도구 호출로

## extract.js — hwpx 텍스트 추출기 (전문)

```javascript
const fs = require('fs');
const zlib = require('zlib');

function readZipEntries(buf) {
  const entries = {};
  let eocd = -1;
  for (let i = buf.length - 22; i >= 0; i--) {
    if (buf.readUInt32LE(i) === 0x06054b50) { eocd = i; break; }
  }
  if (eocd < 0) throw new Error('EOCD not found');
  const cdOffset = buf.readUInt32LE(eocd + 16);
  const count = buf.readUInt16LE(eocd + 10);
  let p = cdOffset;
  for (let i = 0; i < count; i++) {
    const method = buf.readUInt16LE(p + 10);
    const compSize = buf.readUInt32LE(p + 20);
    const nameLen = buf.readUInt16LE(p + 28);
    const extraLen = buf.readUInt16LE(p + 30);
    const commentLen = buf.readUInt16LE(p + 32);
    const lho = buf.readUInt32LE(p + 42);
    const name = buf.slice(p + 46, p + 46 + nameLen).toString('utf8');
    const lNameLen = buf.readUInt16LE(lho + 26);
    const lExtraLen = buf.readUInt16LE(lho + 28);
    const dataStart = lho + 30 + lNameLen + lExtraLen;
    const raw = buf.slice(dataStart, dataStart + compSize);
    entries[name] = { method, raw };
    p += 46 + nameLen + extraLen + commentLen;
  }
  return entries;
}

function getEntry(entries, name) {
  const e = entries[name];
  if (!e) return null;
  return e.method === 8 ? zlib.inflateRawSync(e.raw) : e.raw;
}

function extractText(xml) {
  const paras = [];
  const parts = xml.split(/<\/hp:p>/);
  for (const part of parts) {
    const ts = [...part.matchAll(/<hp:t[^>]*>([\s\S]*?)<\/hp:t>/g)].map(m => m[1]);
    let text = ts.join('');
    text = text.replace(/<[^>]+>/g, '').replace(/&lt;/g, '<').replace(/&gt;/g, '>')
      .replace(/&amp;/g, '&').replace(/&quot;/g, '"').replace(/&apos;/g, "'");
    if (text.trim()) paras.push(text.trim());
    else if (part.includes('<hp:p')) paras.push('');
  }
  return paras.join('\n');
}

const file = process.argv[2];
const entries = readZipEntries(fs.readFileSync(file));
const names = Object.keys(entries).filter(n => /Contents\/section\d+\.xml/.test(n)).sort();
for (const n of names) {
  console.log('===== ' + n + ' =====');
  console.log(extractText(getEntry(entries, n).toString('utf8')));
}
```

## build.js — 생성기 골격 (전문, 내용부만 예시로 축약)

```javascript
const fs = require('fs');
const zlib = require('zlib');
const SRC = '템플릿.hwpx', OUT = '산출물.hwpx';

// readEntries: 위 extract.js와 동일하되 {names(순서), data(Buffer)} 반환

// zip 쓰기 — mimetype 첫 엔트리 + Stored
function writeZip(order, data) {
  const chunks = []; const central = []; let offset = 0;
  for (const name of order) {
    const content = data[name];
    const stored = name === 'mimetype';
    const comp = stored ? content : zlib.deflateRawSync(content, { level: 9 });
    const method = stored ? 0 : 8;
    const crc = zlib.crc32(content) >>> 0;
    const nameBuf = Buffer.from(name, 'utf8');
    const lh = Buffer.alloc(30);
    lh.writeUInt32LE(0x04034b50, 0); lh.writeUInt16LE(20, 4); lh.writeUInt16LE(0, 6);
    lh.writeUInt16LE(method, 8); lh.writeUInt16LE(0, 10); lh.writeUInt16LE(0x5AEF, 12);
    lh.writeUInt32LE(crc, 14); lh.writeUInt32LE(comp.length, 18); lh.writeUInt32LE(content.length, 22);
    lh.writeUInt16LE(nameBuf.length, 26); lh.writeUInt16LE(0, 28);
    chunks.push(lh, nameBuf, comp);
    const ch = Buffer.alloc(46);
    ch.writeUInt32LE(0x02014b50, 0); ch.writeUInt16LE(20, 4); ch.writeUInt16LE(20, 6);
    ch.writeUInt16LE(method, 10); ch.writeUInt16LE(0x5AEF, 14);
    ch.writeUInt32LE(crc, 16); ch.writeUInt32LE(comp.length, 20); ch.writeUInt32LE(content.length, 24);
    ch.writeUInt16LE(nameBuf.length, 28); ch.writeUInt32LE(offset, 42);
    central.push(Buffer.concat([ch, nameBuf]));
    offset += lh.length + nameBuf.length + comp.length;
  }
  const cdBuf = Buffer.concat(central);
  const eocd = Buffer.alloc(22);
  eocd.writeUInt32LE(0x06054b50, 0);
  eocd.writeUInt16LE(order.length, 8); eocd.writeUInt16LE(order.length, 10);
  eocd.writeUInt32LE(cdBuf.length, 12); eocd.writeUInt32LE(offset, 16);
  return Buffer.concat([...chunks, cdBuf, eocd]);
}

// 문단 생성 — lineseg는 대략값, vertpos는 누적만 시키면 됨
function esc(s){return s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');}
let y = 2400;
function linesegs(text, size, spacing) {
  const n = Math.max(1, Math.ceil(text.length / 46));
  const segs = [];
  for (let i = 0; i < n; i++) {
    segs.push(`<hp:lineseg textpos="${i*46}" vertpos="${y}" vertsize="${size}" textheight="${size}" baseline="${Math.round(size*0.85)}" spacing="${spacing}" horzpos="0" horzsize="42520" flags="393216"/>`);
    y += size + spacing;
  }
  return `<hp:linesegarray>${segs.join('')}</hp:linesegarray>`;
}
function para(text, charPr, paraPr, size, spacing) {
  const run = text ? `<hp:run charPrIDRef="${charPr}"><hp:t>${esc(text)}</hp:t></hp:run>`
                   : `<hp:run charPrIDRef="${charPr}"/>`;
  return `<hp:p id="0" paraPrIDRef="${paraPr}" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0">${run}${linesegs(text||' ', size, spacing)}</hp:p>`;
}
// 템플릿(MOU 제안서)의 스타일 매핑: 표지제목 charPr7/paraPr20(1500), 기관 charPr8, 날짜 charPr9,
// 소제목 charPr10/paraPr0(1200), 본문 charPr0/paraPr0(1000). 들여쓰기는 공백 4/8칸 + "- "/"①"

// P0(secPr 문단) 경계는 깊이 추적으로
const xml = data['Contents/section0.xml'].toString('utf8');
let depth = 0, p0start = -1, p0end = -1;
const re = /<hp:p\b[^>]*?(\/?)>|<\/hp:p>/g; let m;
while ((m = re.exec(xml))) {
  if (m[0].startsWith('</')) { depth--; if (depth === 0) { p0end = re.lastIndex; break; } }
  else if (m[1] !== '/') { if (depth === 0) p0start = m.index; depth++; }
}
const head = xml.slice(0, p0start);
let p0 = xml.slice(p0start, p0end).replace('<hp:t>기존 제목</hp:t>', `<hp:t>${esc('새 제목')}</hp:t>`);
const newXml = head + p0 + bodyParas.join('') + '</hs:sec>';
data['Contents/section0.xml'] = Buffer.from(newXml, 'utf8');

// PrvText 원본 인코딩 감지 후 갱신
const prv = data['Preview/PrvText.txt'];
const isU16 = prv[0] === 0xFF && prv[1] === 0xFE;
data['Preview/PrvText.txt'] = isU16
  ? Buffer.concat([Buffer.from([0xFF,0xFE]), Buffer.from(prvText,'utf16le')])
  : Buffer.from(prvText,'utf8');

fs.writeFileSync(OUT, writeZip(names, data));
```

실행 원본: 2026-07-15 세션 스크래치패드 `work/build.js` (완도군 사업계획서 74문단 생성, 전 검증 통과). 위 골격은 그 원본에서 문서 내용부만 뺀 것이다.
