# 핸드오버 — 세무사 인터뷰 질문지 (마을협동조합 세무자료 자동 이관·공동구매) + 한글 파일 생성·배치

> 작성일: 2026-09-19 (작업 2026-09-11 ~ 09-17) | 작성: 데카(김일영) + Claude Code
> 실행 환경: **Claude Code 클라우드 세션** (PC 로컬 폴더 접근 불가, python3·node 사용 가능, 한글 프로그램 없음)
> 세션: https://claude.ai/code/session_01U29PdKj4YCNQdAn8T6ZiwS

## 한 줄 요약

햇빛소득마을을 신청·운영하는 마을협동조합의 세무업무 자동화(세무사 자료 자동 이관 + 연합회 서비스 공동구매)를 설계하기 위한 **세무사 인터뷰 질문지 44문항**을 위키 how-to로 쓰고, 이를 **한글(HWPX) 인쇄용 파일**로 만들어 위키 assets와 사교원 사이트 저장소(poomasi-site/sakyowon/downloads)에 배치했다.

## 요구사항 (사용자 원문 요지)

- 왜: 햇빛소득마을 신청·운영 마을협동조합의 업무 자동화 시스템을 만들고 있다.
- 무엇: 세무업무 자동화를 위해 세무사에게 자료를 자동 이관하는 체계 구축에 필요한 사항을 세무사와 인터뷰한다.
- 어떻게: 마을협동조합이나 연합회가 어떤 작업을 해야 서비스 공동구매로 비용을 절감할 수 있는가.
- 후속: "질문지를 한글파일로", "볼 수 있는 URL", "햇빛발전협동조합 업무자동화 사이트 폴더에 저장".

## 산출물

| 위치 | 파일 | 내용 |
|------|------|------|
| solidarity-intelligence-wiki (브랜치 `claude/tax-accountant-interview-questions-4xi2is`) | `wiki/how-to/tax-accountant-interview-questions.md` | 질문지 원본. 6단계 44문항, ★ 필수 17개, 단계별 "확인하려는 가설", 사전 준비물, 인터뷰 후 요구사항→기능 매핑 표, 자주 하는 실수 |
| 〃 | `wiki/assets/tax-accountant-interview-questions.hwpx` | 한글 인쇄용 (A4, 번호·★·질문·답변 메모란 표, 바닥글 쪽번호) |
| 〃 | `wiki/assets/tax-accountant-interview-questions.build.py` | 생성 스크립트 (python-hwpx 6.x). 질문 수정 후 재생성 가능 |
| poomasi-site (브랜치 `sakyowon-tax-interview-hwpx-20260917`) | `sakyowon/downloads/tax-accountant-interview-questions.hwpx` | 사교원 사이트 폴더 배치본. **main 미반영** (아래 남은 작업) |

URL:
- 원본 문서: https://github.com/deka2026/solidarity-intelligence-wiki/blob/claude/tax-accountant-interview-questions-4xi2is/wiki/how-to/tax-accountant-interview-questions.md
- 한글 파일(raw): https://github.com/deka2026/solidarity-intelligence-wiki/raw/claude/tax-accountant-interview-questions-4xi2is/wiki/assets/tax-accountant-interview-questions.hwpx
- 사이트 저장소 배치본: https://github.com/deka2026/poomasi-site/blob/sakyowon-tax-interview-hwpx-20260917/sakyowon/downloads/tax-accountant-interview-questions.hwpx

## 질문지 구조 (요약)

| 단계 | 시간 | 핵심 ★ 질문 |
|------|------|-------------|
| 1. 워밍업 — 세무사 현재 업무 방식 | 10분 | 협동조합·발전사업 경험 / 자료 수령 경로와 병목 |
| 2. 세무 구조 | 25분 | 사업자등록 / 연간 신고 달력 / 매출·보조금별 부가세 / 보조금 매입세액 공제 / 배당 처리 / **주민 햇빛소득 소득구분·원천징수** |
| 3. 자료 자동 이관 체계 | 25분 | 월별 자료 목록·형식·마감 / 자동화 우선 3가지·금지 항목 / 반려·되묻기 흐름 / 이관 방식(CSV·임포트·ASP·스크래핑) |
| 4. 공동구매·비용절감 | 20분 | 수수료 산정 기준 / 20·50·100곳 묶을 때 단가 / 연합회가 해 줄 것(표준 양식·1차 검수·단일 창구·교육·일정 분산) |
| 5. 책임·리스크 | 5분 | 시스템 자료 오류 시 책임 분담 조항 |
| 6. 마무리 | 5분 | "먼저 갖출 것 3가지" / 파일럿 의향 |

설계 원칙: 세법 지식 질문(2단계)보다 **"세무사가 무엇을 어떤 형식으로 받고 싶은가"(3단계)와 "무엇이 갖춰지면 단가가 내려가나"(4단계)에 시간을 배정**. 리포의 기존 과제([[handover-vat-2026h1-bizno-match]]의 "보조금 집행분 매입세액 공제 확인")를 Q10으로 흡수.

## 배치 판단 기록 (사이트 폴더)

- 사용자가 말한 "햇빛발전협동조합 업무자동화 사이트 폴더"의 실체 후보: ① PC 로컬 `C:\Users\pc\클로드로컬\sunvillage-network\` (통합 사이트, 저장소 없음) ② `poomasi-site/sakyowon/sunvillage.html` (햇빛소득마을 종합 업무도우미, 사교원 사이트) ③ 기타 저장소.
- 클라우드 세션은 ①에 쓸 수 없음. deka2026 저장소 21개를 list_repos로 훑고 sakyowon-ai·hometown-love·sakyowon-server·mangnam-coop를 얕은 클론으로 확인 → 햇빛 관련 사이트는 ②만 존재.
- AskUserQuestion으로 위치를 물었으나 "선호 없음" 응답 → ②에 배치. poomasi-site 거버넌스(main 직접 푸시 금지, 패미는 staging만, 지미가 검토·머지)를 따라 **날짜 브랜치**로 푸시하고 main·staging은 건드리지 않음. sunvillage.html에 다운로드 링크는 추가하지 않음(요청 범위 밖).

## 남은 작업

1. **한글에서 실제로 열어 렌더링 확인** — 클라우드에는 한글이 없어 스키마 검증·텍스트 복원만 했다. 표 폭(48000 HWP unit ≈ 170mm)과 셀 글꼴 크기가 어색하면 `build.py`의 `BODY_W`·`ensure_run_style(size=…)` 조정 후 재생성.
2. poomasi-site 배치본을 사이트에서 접근 가능하게 하려면 `sakyowon-tax-interview-hwpx-20260917` → staging → main 머지 (지미 검토). 필요하면 sunvillage.html 서류관리/자료 영역에 `downloads/tax-accountant-interview-questions.hwpx` 링크 추가.
3. 인터뷰 실행: 개인 세무사 1 + 세무법인 1 이상. 결과는 how-to 문서의 "인터뷰 후 정리" 표에 채우고, 조합 개별 금액·거래처는 위키에 올리지 않는다([[lesson-vat-bizno-matching]] 3항).
4. 위키 브랜치는 PR 미생성 상태 — main 반영 시 PR 필요.

## 관련 문서

- 방법: [[skill-python-hwpx-cloud-generate]] — 템플릿 없이 python-hwpx로 hwpx 생성 + linesegarray 보강 + 재포장
- 레슨: [[lesson-cloud-session-repo-mapping]] — 클라우드 세션에서 "폴더에 저장해" 요청을 저장소로 번역하기
- 선행: [[handover-vat-2026h1-bizno-match]], [[handover-sunvillage-network-merge]], [[skill-hwpx-generate-from-template]]
