# 핸드오버: 사교원 연대지능활동가 아카데미 사이트 (v2)

**작성일:** 2026-06-28  
**작성자:** Claude (데카 요청)  
**이전 핸드오버:** `docs/handover-academy-site.md` (2026-06-27)

---

## 이번 세션 작업 요약

### 완료된 작업

1. **리뷰 5가지 질문 드롭다운 추가**
   - 위키 작성법과 온톨로지 페이지의 리뷰 5가지 질문에 상세 설명 추가
   - 각 질문마다 점검 포인트, ✅/❌ 예시 포함
   - `onclick="toggleStep(this)"` + `<div class="step-body">` 구조

2. **내 진행 상태 진단 도구** (`course-next` 페이지 전면 개편)
   - 15개 항목 점검 프롬프트 제공 (PC도구 5 + AI도구 3 + 클라우드 4 + 위키 3)
   - AI 답변 붙여넣기 → ✅/❌ 자동 파싱 → 진행률 바 + 마일스톤(M1~M4) 판정
   - 미완료 항목별 안내 + 해당 교육 페이지 링크

3. **메뉴별 점검표 엑셀 생성**
   - 파일: `C:\Users\Admin\Desktop\아카데미_메뉴별_점검표.xlsx`
   - 74개 점검 항목, 6대분류, 드롭다운(정상/오류/미완), 조건부 서식
   - 요약 시트 포함

4. **지식 검증 챌린지 게임** (신규 대메뉴)
   - 사이드바에 "검증 게임" 대분류 추가 (지식 검증 챌린지 + 내 검증 기록)
   - 20개 문장 풀에서 10문제 랜덤 출제
   - 좋음/보완/오류 3택 + 판단 이유 작성 + 정답 해설
   - localStorage에 기록 저장, 성과 점수 반영 (verifyPts)
   - `calcScore()`에 `verifyPts` 합산 반영

### 미완료 작업 (다음 세션에서 이어야 할 것)

1. **300문제 생성 및 적용**
   - Python 스크립트 작성 완료: `scratchpad/gen_questions.py`
   - **아직 실행하지 않음** → JSON 생성 → HTML에 임베드 필요
   - 현재 HTML의 `GAME_STATEMENTS` 배열(20개)을 300개로 교체해야 함
   - 게임에서 10문제씩 랜덤 출제하는 로직은 이미 구현되어 있음

2. **품질 리뷰 백엔드 (관리자 대시보드)**
   - 설계만 됨, 구현 안 됨
   - 필요한 기능:
     - 전체 사용자 답변 집계 (문항별 좋음/보완/오류 분포)
     - "오류" 비율 높은 문장 → 해당 위키 문서 품질 경고
     - "보완" 비율 높은 문장 → 보완 필요 사항 리스트
     - 관리자 페이지(`access-admin`)에 탭 추가 또는 별도 페이지
   - localStorage 키: `academy_verify_game` (기존 기록에 이미 details 포함)

3. **모바일 홈화면 바로가기**
   - 미구현
   - 필요한 것:
     - PWA manifest 메타태그 (Artifact 내에서 가능한 범위)
     - "홈화면에 추가" 안내 모달 (iOS: 공유→홈화면에 추가, Android: 메뉴→홈화면에 추가)
     - 사이트 내 "모바일 홈화면 만들기" 버튼 (topbar 또는 대시보드)

---

## 기술 상세

### 현재 Artifact URL
`https://claude.ai/code/artifact/89454744-b73c-46a6-92fa-64d5d1b9ea9a`

### 소스 파일 위치
- 스크래치패드: `C:\Users\Admin\AppData\Local\Temp\claude\C--Users-Admin\4f44ef7e-b29c-4f7c-a385-c0d9e62277a3\scratchpad\academy.html`
- 300문제 생성 스크립트: 같은 scratchpad 폴더의 `gen_questions.py`

### localStorage 키 목록
| 키 | 용도 |
|---|---|
| `academy_pending` | 교육 신청 대기 목록 |
| `academy_users` | 사용자 DB (이메일 기반) |
| `academy_session` | 현재 로그인 세션 |
| `academy_wiki_contrib` | 위키 기여 데이터 (GitHub 동기화) |
| `academy_github_map` | 이메일↔GitHub 매핑 |
| `academy_verify_game` | 검증 게임 기록 (신규) |

### 점수 체계
```javascript
// calcScore — verifyPts 추가됨
(d.docs||0)*10 + (d.merged||0)*5 + (d.reviews||0)*3 + (d.improvements||0)*5 + (d.links||0)*2 + (d.verifyPts||0)

// 검증 게임 점수
// 참여 1건 = +2점, 오류 정확 발견 = +3점 추가, 10건 완료 보너스 = +5점
```

### 접근 제어
```javascript
var userPages = ['contributions','performance-share','learning-portfolio','verify-game','verify-history'];
var adminPages = ['access-admin','schedule-mgmt','notice-board'];
```

### 사이드바 메뉴 구조 (현재)
```
홈: 대시보드
교육과정: 전체 환경 관계도, PC 로컬 도구 설치, AI 도구 설정, 클라우드 서비스 연결, 바이브 코딩 작업 흐름, 현재 상태 & 다음 단계
연대지능: 연대지능 위키(볼드), 연대지능이란, 활동가 역할과 역량, 데이터 주권과 활동의 상, 사회연대경제 AI 구축, 위키 작성법과 온톨로지
검증 게임: 지식 검증 챌린지(볼드), 내 검증 기록
성과: 위키 기여도 평가, 성과 발표와 공유, 학습 포트폴리오
관리: 교육생 관리, 교육 일정, 공지사항
```

---

## 다음 세션 작업 순서 (권장)

1. `gen_questions.py` 실행 → `questions.json` 생성
2. HTML의 `GAME_STATEMENTS` 배열을 300문제 JSON으로 교체
3. 게임에서 출제 수를 10→15 또는 20으로 조정 검토
4. 품질 리뷰 관리자 대시보드 구현
5. 모바일 홈화면 바로가기 기능 추가
6. Artifact 재배포 및 테스트
