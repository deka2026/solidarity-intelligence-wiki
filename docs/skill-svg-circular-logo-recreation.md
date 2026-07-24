# 스킬 — 원형 문구 로고를 인라인 SVG로 재현하기

> 작성일: 2026-07-24 | 검증: sakyowon.co.kr 상단 로고 (sakyowon-hub/index.html)
> 용도: 붙여넣기 이미지를 파일로 저장할 수 없을 때, 로고를 코드로 재현해 사이트에 직접 심기

## 핵심 구조

```html
<svg viewBox="0 0 600 600" xmlns="http://www.w3.org/2000/svg" role="img">
  <defs>
    <!-- 위쪽 문구용 원호: 왼쪽→오른쪽, 위로 볼록 (sweep=1) -->
    <path id="arcTop" d="M 58 300 A 242 242 0 0 1 542 300" fill="none"/>
    <!-- 아래쪽 문구용 원호: 왼쪽→오른쪽, 아래로 볼록 (sweep=0) → 글자가 뒤집히지 않음 -->
    <path id="arcBottom" d="M 48 300 A 252 252 0 0 0 552 300" fill="none"/>
  </defs>
  <circle cx="300" cy="300" r="200" fill="none" stroke="#1e3d98" stroke-width="12"/>
  <text font-size="42"><textPath href="#arcTop" startOffset="50%" text-anchor="middle">위쪽 문구</textPath></text>
  <text font-size="42"><textPath href="#arcBottom" startOffset="50%" text-anchor="middle">아래쪽 문구</textPath></text>
</svg>
```

## 요령

1. **아래쪽 문구가 뒤집히지 않게 하는 법**: 별도 속성 없이, 원호 path를 **왼쪽에서 오른쪽으로, 아래로 지나가게(sweep=0)** 그리면 된다. 이때 글자 윗부분이 원 중심을 향하므로, 원호 반지름을 본체 원보다 **충분히 크게**(글자 캡 높이 ≈ font-size×0.7만큼 여유) 잡아야 본체 원과 안 겹친다.
2. **원호 대비 문구 길이 검증**: 반원 길이 = π×r. `getComputedTextLength()`가 이보다 짧으면 안전. (이번 건: 503 vs 760)
3. **지하철 노선도 스타일 라인아트**: `stroke-linecap="round" stroke-linejoin="round"` + 모서리는 `Q`(2차 베지어)로 라운드 코너. 색 그룹별로 `<g fill="none" stroke="...">`로 묶으면 관리 쉬움.
4. **선 끝을 원 테두리에 정확히 붙이기**: 원 중심 (cx,cy), 반지름 r일 때 y가 주어지면 x = cx ± √(r² − (y−cy)²). 끝점을 테두리 중심선 좌표에 두면 딱 닿아 보인다.
5. **HTML에 넣을 때**: 컨테이너 div에 `aria-label`, svg에 `role="img"`. CSS는 컨테이너 크기만 주고 `svg { width:100%; height:100% }`.

## 스크린샷 없이 렌더링 검증 (브라우저 패널이 안 보일 때)

Browser pane이 화면에 표시되지 않으면 screenshot이 타임아웃한다. 대신 `javascript_tool`로 기하 검증:

```js
const t = document.querySelector('.logo svg text');
const b = t.getBBox();                 // 문구가 viewBox 밖으로 나가는지
t.getComputedTextLength();             // 원호 길이 대비 문구 길이
```

bbox가 viewBox(0~600) 안이고, 문구 길이 < 원호 길이면 배포해도 안전.

관련: [[handover-sakyowon-hub-logo-swap]] [[skill-pdf-extract-and-pasted-image]]
