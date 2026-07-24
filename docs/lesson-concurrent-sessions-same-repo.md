# 레슨 — 같은 저장소를 두 세션이 동시에 만질 때 생기는 일

> 작성일: 2026-07-24 | 사례: sakyowon-hub 로고 교체 세션 (다른 세션이 동시에 SEO 작업)

## 무슨 일이 있었나

sakyowon-hub 로고를 교체하는 동안, 같은 저장소에서 다른 Claude 세션이 SEO 작업(제목/메타 변경, robots.txt·sitemap.xml 생성·스테이징, 망남 링크 교체 커밋을 원격에 푸시)을 하고 있었다.

## 교훈 3가지

1. **`git add 파일 && git commit`은 "그 파일만" 커밋하는 게 아니다.** 다른 세션이 스테이징해둔 파일이 있으면 함께 커밋된다. 이번엔 robots.txt·sitemap.xml이 로고 커밋에 딸려 들어갔다(추가성 파일이라 유지했지만, 미완성 작업이었다면 사고).
   - 대응: 커밋 전 `git status`로 스테이징 영역 확인. 남의 스테이징이 보이면 `git commit -- 파일명` 또는 `git restore --staged`로 분리.
2. **푸시 거부(fetch first)는 곧 "다른 세션이 먼저 푸시했다"는 신호.** 무조건 `git pull --rebase` 하지 말고, 먼저 `git fetch` 후 `git log main..origin/main`으로 뭐가 올라왔는지 보고 rebase. 이번엔 겹치는 파일이라도 다른 줄이라 깔끔히 리베이스됨.
3. **Edit 결과에 "file had been modified on disk" 경고가 뜨면 동시 편집 중이라는 뜻.** 주변 내용에 의존하는 편집 전에 반드시 다시 Read. (이번엔 h1 문구가 다른 세션에 의해 바뀌어 있었다.)

## 보너스 함정

- **Browser pane 미표시 상태에서는 screenshot이 안 된다** ("not compositing frames"). 사용자에게 패널 열어달라 하지 말고 `javascript_tool` 기하 검증으로 대체 → [[skill-svg-circular-logo-recreation]] 참조.
- `file:///` 경로로 프로젝트 밖 파일을 열면 정적 스냅샷으로 렌더링됨(핫리로드 없음). 편집 후엔 navigate `force:true`로 다시 열어야 반영.

관련: [[handover-sakyowon-hub-logo-swap]]
