# 레슨 — 아티팩트 배포 & 브라우저 검증 함정

> 작성일: 2026-07-24 | 작성: 데카(김일영) + Claude Code
> 사례: [[handover-sunvillage-network-merge]]

이 PC에서 단일 HTML 사이트를 아티팩트로 배포하고 브라우저 도구로 검증하며 겪은 함정과 우회로.

## 1. 아티팩트는 전체 문서가 아니라 body 콘텐츠만

Artifact 툴은 파일을 `<!doctype><head></head><body>` 스켈레톤으로 **자동 래핑**한다. 완성된 `<!DOCTYPE html>…</html>` 문서를 그대로 넣으면 이중 래핑된다.
- **우회**: `<style>`부터 `</html>` 직전까지 잘라내고 `</head><body>`·`</body></html>`를 제거한 body 전용 파일을 따로 만들어 배포.

```powershell
$t = Get-Content $src -Raw -Encoding UTF8
$body = $t.Substring($t.IndexOf("<style>"), $t.LastIndexOf("</html>") - $t.IndexOf("<style>"))
$body = $body -replace "</head>\s*<body>","`n" -replace "</body>\s*$",""
Set-Content $dst -Value $body -Encoding UTF8 -NoNewline
```

## 2. 아티팩트 CSP가 외부 폰트/CDN/AI서버를 전부 차단

`fonts.googleapis.com`, jsdelivr Pretendard, `/api/ai` 등 외부 호스트 요청이 **strict CSP로 차단**된다.
- 폰트: 링크 걸어도 조용히 시스템 폰트로 폴백 → 자르는 지점이 `<style>`부터라 head의 폰트 `<link>`는 자동 제거됨(문제 없음). 원본 로컬 파일은 정상.
- AI: 아티팩트에선 절대 작동 안 함 → 검토 시 사용자에게 "폰트=시스템, AI=안내메시지"라고 미리 고지.

## 3. 같은 URL로 갱신하려면 `url` 파라미터 필수

아티팩트를 통합본으로 갱신할 때 `url` 파라미터에 기존 주소를 넣어야 같은 링크가 유지된다. 안 넣으면 새 URL이 발급된다. (같은 대화·같은 file_path면 재배포로 유지되지만, 명시가 안전)

## 4. Browser 도구 `computer`(스크린샷) 타임아웃

브라우저 패널이 화면에 표시되지 않으면 스크린샷/클릭이 `computer timed out`/`not compositing frames`로 실패한다.
- **우회**: 검증은 `javascript_tool`로 직접 상태를 읽고 실행하는 방식이 훨씬 안정적. `get_page_text`로 텍스트 확인. 스크린샷은 패널이 보일 때만.

## 5. `javascript_tool`의 top-level `const`는 호출 간 유지된다

같은 탭에서 `javascript_tool`을 여러 번 호출하면 실행 컨텍스트가 공유돼 `const origAlert` 재선언 시 `Identifier already declared` 에러.
- **우회**: 각 실행을 **IIFE**로 감싸고 결과를 `window.__R`에 담아 반환.

```js
(function(){ const oa=window.alert; window.alert=()=>{}; /* ... */ window.__R=JSON.stringify(R); })(); window.__R;
```

## 6. 날짜 +N개월 계산의 월말 오버플로 (경영공시 기한)

`d.setMonth(d.getMonth()+4)`는 원래 날짜가 말일이면 다음 달로 밀린다(1/31+1월=3/3 등). 결산일+4개월 공시기한 계산에서 버그.
- **우회**: 목표 월을 직접 만들고, 월이 어긋나면 그 달 0일(=말일)로 보정.

```js
let d = new Date(y, m-1+4, day);
if (d.getMonth() !== (m-1+4)%12) d = new Date(y, m+4, 0);
```

## 7. file:// 미리보기는 정적 스냅샷

`preview_start`로 로컬 파일을 열면 "files outside the project folder render as static snapshots" 안내가 뜬다. JS는 실행되지만, 서버 필요한 기능(fetch)은 당연히 안 됨 → AI 우아한 실패(#레슨2)와 맞물림.

## 관련

[[skill-html-app-merge-and-tiers]] · [[handover-sunvillage-network-merge]] · [[lesson-pptx-hancom-readability]]
