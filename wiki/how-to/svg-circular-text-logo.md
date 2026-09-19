# 이미지 파일 없이 원형 문구 로고를 코드(SVG)로 만들기

> **한 줄 요약**: 로고 이미지 파일이 없어도, 둥근 테두리를 따라 글자가 흐르는 엠블럼 로고를 HTML 안에 코드로 직접 그려 넣는 방법

**태그**: `#웹사이트` `#로고` `#SVG` `#ClaudeCode` `#입문`
**작성자**: 데카 (김일영)
**작성일**: 2026-07-24
**난이도**: 보통
**소요 시간**: 약 20분 (읽고 따라하기)

---

## 이 가이드가 필요한 상황

- 로고 시안은 있는데(카톡으로 받은 사진, 채팅에 붙여넣은 이미지 등) **원본 파일이 없을 때**
- 이미지 파일을 서버에 올리지 않고 **HTML 한 파일**로 사이트를 유지하고 싶을 때
- 로고 색·문구를 나중에 **텍스트 편집만으로** 바꾸고 싶을 때

실제 사례: 사교원 허브(sakyowon.co.kr) 상단 로고를 이 방법으로 교체했습니다. 파란 원 테두리 위아래로 "coop change beyound" / "social meta platform" 문구가 흐르는 엠블럼입니다.

## 핵심 아이디어

SVG의 `textPath`를 쓰면 **글자가 보이지 않는 곡선을 따라 흐릅니다**. 원 위쪽에 문구 하나, 아래쪽에 문구 하나를 얹으려면 곡선(원호) 2개만 정의하면 됩니다.

```html
<svg viewBox="0 0 600 600" xmlns="http://www.w3.org/2000/svg" role="img">
  <defs>
    <!-- 위쪽 문구가 탈 원호: 왼쪽→오른쪽으로 위를 지나감 -->
    <path id="arcTop" d="M 58 300 A 242 242 0 0 1 542 300" fill="none"/>
    <!-- 아래쪽 문구가 탈 원호: 왼쪽→오른쪽으로 아래를 지나감 -->
    <path id="arcBottom" d="M 48 300 A 252 252 0 0 0 552 300" fill="none"/>
  </defs>

  <!-- 본체 원 테두리 -->
  <circle cx="300" cy="300" r="200" fill="none" stroke="#1e3d98" stroke-width="12"/>

  <!-- 원호를 따라 흐르는 문구 -->
  <g fill="#1e3d98" font-weight="700" letter-spacing="4">
    <text font-size="42"><textPath href="#arcTop" startOffset="50%" text-anchor="middle">위쪽 문구</textPath></text>
    <text font-size="42"><textPath href="#arcBottom" startOffset="50%" text-anchor="middle">아래쪽 문구</textPath></text>
  </g>
</svg>
```

## 따라하며 이해하기

1. **아래쪽 글자가 뒤집히지 않는 비밀**: 원호를 그리는 방향입니다. `A ... 0 0 1`(sweep=1)은 위로 볼록, `A ... 0 0 0`(sweep=0)은 아래로 볼록한 원호가 되는데, 둘 다 왼쪽→오른쪽으로 그리면 글자가 항상 바로 서 있습니다.
2. **아래쪽 원호는 본체 원보다 넉넉히 크게**: 아래 문구는 글자 윗부분이 원 중심을 향합니다. 원호 반지름이 본체 원과 비슷하면 글자가 테두리를 뚫고 들어갑니다. 글자 크기의 약 0.7배만큼 여유를 두세요(예: 본체 r=200, 글자 42px → 원호 r=252).
3. **문구가 다 들어가는지 계산**: 반원 길이는 약 3.14 × 반지름. 문구가 이보다 길면 글자가 잘립니다. letter-spacing을 줄이거나 글자를 작게.
4. **`startOffset="50%"` + `text-anchor="middle"`**: 문구를 원호 정중앙에 가운데 정렬합니다. 이 두 줄만 있으면 좌우 균형은 자동.
5. **내부 그림(라인아트)**: 지하철 노선도 느낌을 내려면 `stroke-linecap="round"`(선 끝 둥글게) + 모서리를 `Q`(곡선 명령)로 그리면 됩니다. 색깔별로 `<g stroke="색">`으로 묶어두면 나중에 색 바꾸기가 한 줄입니다.

## HTML 사이트에 넣기

로고 자리에 이렇게 넣습니다 (이미지 태그 대신 SVG를 직접):

```html
<div class="logo" aria-label="우리 조직 로고 — 위쪽 문구, 아래쪽 문구">
  <svg viewBox="0 0 600 600" ...>...</svg>
</div>
```

```css
.logo { width: 170px; height: 170px; }
.logo svg { width: 100%; height: 100%; }
```

크기 조절은 CSS의 width/height만 바꾸면 됩니다. SVG는 벡터라 아무리 키워도 안 깨집니다.

## 주의할 점

- **재현본은 재현본**: 사진을 보고 코드로 다시 그린 것이라 원본과 세부가 다를 수 있습니다. 원본 파일을 구하면 교체하는 게 정확합니다.
- **글꼴**: SVG 안 글자는 방문자 컴퓨터의 글꼴로 그려집니다. `font-family`에 흔한 글꼴을 여러 개 나열해 두세요(예: `'Trebuchet MS','Segoe UI',Verdana,sans-serif`).
- **접근성**: 로고 div에 `aria-label`을 꼭 넣으세요. 화면낭독기 사용자에게 로고 내용을 알려줍니다.

## 더 알아보기

- 실제 적용 코드: https://github.com/deka2026/sakyowon-hub 의 index.html
- MDN textPath 문서: https://developer.mozilla.org/en-US/docs/Web/SVG/Element/textPath
