# 스킬 — 단일 HTML 앱 통합 + 회원 등급 게이팅 레이어링

> 작성일: 2026-07-24 | 작성: 데카(김일영) + Claude Code
> 적용 사례: [[handover-sunvillage-network-merge]]

## 언제 쓰나

이미 검증된 대형 단일 HTML 앱(수천 줄, 로직 완성)에 **다른 사이트의 기능을 합치거나 회원제/권한 계층을 얹어야 할 때.** 처음부터 다시 쓰지 않고 안전하게 확장하는 방법.

## 핵심 원칙: "재작성하지 말고 레이어링하라"

2,000줄 넘는 검증된 앱을 다시 타이핑하면 반드시 회귀 버그가 난다. **원본을 복사한 뒤 그 위에 최소 편집만** 얹는다.

```
1. 원본 파일을 그대로 복사 (Copy-Item)
2. 리브랜드 (title, 로고 텍스트) — 텍스트 치환만
3. 확장 지점에만 국소 편집:
   - 헤더/네비에 인증 UI 삽입
   - 신규 섹션 HTML을 기존 섹션들 사이에 주입 (display:none이므로 위치 무관)
   - 신규 CSS 블록을 </style> 직전에 추가
   - 신규 JS 블록을 초기화(init) 직전에 추가
4. 진입 함수(go/navigate) 한 곳만 게이팅+디스패치로 재작성
```

## 회원 등급 게이팅 패턴 (탭/섹션 공용)

### 1) 탭에 선언적 속성 부여 — 위치 기반 map 금지

```html
<div class="tab" data-sec="filing" data-need="paid" onclick="go('filing',this)">📁 서류관리</div>
<div class="tab" data-sec="admin" data-need="admin" id="tab-admin" style="display:none">🛠 관리자</div>
```

> ⚠️ 기존 코드가 `{dash:0, docs:2, ...}` 같은 **위치 인덱스 map**을 쓰면 탭을 추가하는 순간 전부 어긋난다. `data-sec`로 셀렉트하도록 바꾼다.

### 2) go()에서 게이팅

```js
function go(id, tab){
  const btn = tab || document.querySelector('.tab[data-sec="'+id+'"]');
  const need = btn ? btn.dataset.need : null;
  if (need==='paid' && !canPaid()){ if(!user) openAuth(); else alert('정회원 전용'); return; }
  if (need==='admin' && !isAdmin()){ alert('관리자 전용'); return; }
  // ...섹션 전환 + 렌더러 디스패치
}
```

### 3) 역할 판정은 포함 배열로

```js
const canPaid = () => user && ['paid','menuAdmin','superAdmin'].includes(user.role);
const isAdmin = () => user && ['menuAdmin','superAdmin'].includes(user.role);
const isSuper = () => user && user.role === 'superAdmin';
```

## localStorage 네임스페이스 충돌 방지

두 앱을 합칠 때 키가 겹치면 데이터가 오염된다. **접두어를 분리**한다.
- 원본 유지: `hsv_villages`, `hsv_doccount`
- 신규 기능: `svnet_users`, `svnet_qa`, `svnet_kpi` …
- 헬퍼도 이름 충돌 주의 — 원본에 `load/save`가 있으면 신규는 `loadN/saveN`, `today`는 `todayN`으로.

## 시딩 패턴 (데모 계정/샘플 데이터)

```js
function seedN(){ if (loadN('svnet_users', null)) return; /* 최초 1회만 */ saveN(...); }
```

## 외부 의존(AI 등) 우아한 실패

백엔드(`/api/ai`)가 없는 데모에서 raw 에러 대신 안내 문구:

```js
function aiErrMsg(e){
  const m = (e&&e.message)||'';
  if (/HTTP 40|HTTP 5|Failed to fetch|NetworkError|Load failed/i.test(m)||m==='')
    return '🤖 AI 기능은 서버 연결 시 작동합니다. 지금은 데모 화면입니다.';
  return '⚠️ 오류: ' + m;
}
```

## 체크리스트

- [ ] 원본 복사본에서 작업했는가 (재작성 아님)
- [ ] 위치 기반 인덱스 map을 data-속성 셀렉트로 교체했는가
- [ ] localStorage 키/헬퍼 이름 충돌 없는가
- [ ] 진입 함수 한 곳에서만 게이팅했는가
- [ ] 비로그인/정회원/관리자 3경로 모두 브라우저에서 실행 검증했는가

## 관련

[[handover-sunvillage-network-merge]] · [[lesson-artifact-and-browser-pitfalls]] · [[skill-supabase-auth-pitfalls]]
