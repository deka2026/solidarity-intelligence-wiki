# 핸드오버 — sakyowon.co.kr 상단 로고 교체 (coop change beyound 엠블럼)

> 작성일: 2026-07-24 | 작성: 데카(김일영) + Claude Code
> 대상: `C:\Users\pc\클로드로컬\sakyowon-hub\index.html` → https://sakyowon.co.kr/
> 배포 커밋: `8b135ea` (main, GitHub Pages 반영 확인 완료)

## 한 줄 요약

sakyowon.co.kr(= **sakyowon-hub** 저장소가 CNAME으로 서비스) 상단의 초록 원 `社` 로고를, 사용자가 붙여넣은 새 엠블럼(파란 원 + "coop change beyound / social meta platform" 원형 문구 + 노랑·빨강·초록 라인아트)의 **SVG 재현본**으로 교체했다.

## 도메인 사실 (새로 확인)

- **sakyowon.co.kr = deka2026/sakyowon-hub** (CNAME 파일로 커스텀 도메인 연결). 기존 생태계 지도에는 deka2026.github.io/sakyowon-hub/만 기록돼 있었음.
- 즉 "사교원 대표 도메인"이 허브 관문 페이지를 가리킨다. 사교원 홈페이지(sakyowon.poomasi.org)와는 별개.

## 변경 내용

1. `.logo` CSS: 56px 초록 원 → 170px 투명 컨테이너 + `svg` 100%.
2. `<div class="logo">社</div>` → 인라인 SVG 엠블럼 (viewBox 600×600):
   - 파란 원 테두리 r=200, stroke 12, `#1e3d98`
   - 위 원호 문구 "coop change beyound" / 아래 원호 문구 "social meta platform" (textPath, 42px, letter-spacing 4)
   - 내부 라인아트: 노랑 `#f2cf5b` 아치+우상단 연결선, 빨강 `#e2342b` 좌측 곡선+기둥 2개+수평선, 초록 `#0a9d49` 고리+연결선+하단 세로선/곡선. 모든 끝점은 파란 원 테두리 중심선에 닿도록 계산(dx=√(r²−dy²)).
3. 접근성: `aria-label` + `role="img"`.

## ⚠️ 재현본임 (원본 파일 아님)

이 PC는 채팅 붙여넣기 이미지를 파일로 저장 못 함 → 원본을 보고 만든 SVG 재현본. 전체 구성은 같으나 내부 라인 세부 형상은 다를 수 있음. **원본 PNG/SVG 파일을 sakyowon-hub 폴더에 넣으면 그대로 교체 예정.** 문구는 원본 표기 그대로 `beyound`(오탈자 여부 미확인) 유지.

## 배포 과정에서 겪은 일 (동시 세션 충돌)

- 작업 중 같은 저장소를 **다른 세션이 동시 편집** 중이었음: h1/title/SEO 메타 변경 + robots.txt·sitemap.xml이 스테이징돼 있었고, 원격에는 망남 링크 교체 커밋(ec06439)이 먼저 올라가 있었음.
- 내 커밋에 다른 세션이 스테이징해둔 robots.txt·sitemap.xml이 **함께 포함**됨(의도치 않았으나 추가성 파일이라 유지). 푸시 거부 → `git pull --rebase` 후 푸시 성공.

## 남은 일

- [ ] 파비콘·PWA 아이콘(icon-192/512, apple-touch-icon, manifest)은 아직 기존 `社` 디자인 → 새 로고로 갱신할지 결정
- [ ] 원본 로고 파일 확보 시 SVG 재현본 교체
- [ ] "beyound" 철자 확인 (브랜드 표기인지 오탈자인지)

관련: [[skill-svg-circular-logo-recreation]] [[lesson-concurrent-sessions-same-repo]]
