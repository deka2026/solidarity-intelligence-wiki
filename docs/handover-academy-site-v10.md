# 아카데미 사이트 핸드오버 v10 — 교육신청 수정·사이드바 개편 + 사업계획서 PDF 편집

> 작성일: 2026-07-05 | 작성: 데카(김일영) + Claude Code
> 사이트: https://deka2026.github.io/academy-site/
> 저장소: https://github.com/deka2026/academy-site

## 한 줄 요약

교육 신청 접수 실패(프로필 저장 오류)의 원인 3가지를 진단·수정해 자동승인 가입을 정상화했고, 사이드바 메뉴를 색상·크기로 구분했으며, 별도로 전남광주 시민주권국 사업계획서를 원문박스+해설 구조 PDF로 편집했다.

## 이번 버전에서 완료된 작업 (커밋 순)

| 커밋 | 작업 |
|------|------|
| `a45cdc5` | 교육 신청 접수 실패 수정 — profiles 직접 upsert 제거 → create_user_profile RPC(SECURITY DEFINER), 상세정보는 applications에만 저장, 실제 오류 메시지 표시 |
| `5f02406` | RPC v2 — profiles.id(NOT NULL)에 auth.users.id를 조회해 채움 |
| `9b2cb83` | prompt() 비밀번호 팝업 제거 → 신청 폼 내 비밀번호 입력칸 (github.io 도메인 탓에 GitHub 비번창으로 오인되던 문제 해소) |
| `fc3b3c2` | 사이드바 메뉴 구분 — 섹션제목 검정 15px, 볼드 메뉴 파랑(#2563eb), 일반 메뉴 보라(#7c3aed) |
| `d15b3d9` | 연대지능네트워크 메뉴도 파랑(menu-bold) 적용 |

## 교육 신청 실패 — 원인 3가지와 해결 (중요)

1. **profiles 테이블에 phone/region/site 컬럼 없음** → PGRST204로 upsert 실패. 프로필에는 email/name/role만 저장, 상세정보는 applications 테이블로 분리
2. **이메일 인증(Confirm email) ON** → 가입 후 세션이 없어 RLS가 insert 차단 + 이후 로그인도 불가. 사용자가 대시보드에서 OFF 처리
3. **profiles.id NOT NULL** → RPC가 auth.users에서 이메일로 id를 조회해 채우도록 v2 수정
- 백엔드 작업: `supabase_create_user_profile.sql` 실행 완료(사용자), Confirm email OFF 완료(사용자)
- 진단 방법: 프리뷰 콘솔에서 sb 클라이언트로 직접 upsert/signUp을 실행해 실제 오류 코드 확인

## Supabase 프로젝트 정보

- 대시보드: https://supabase.com/dashboard/project/gklecgujcoznxyvywnyu
- 대시보드가 깨져 보이면 크롬 자동번역 OFF (알려진 이슈)

## 별도 작업 — 전남광주 시민주권국 사업계획서 PDF 편집

- 위치: `C:\Users\pc\sakyowon-ai\전남광주 공동체자산 계획수립\`
- v2.0: 4개 요청(이론배경·정량목표·추진방법/예산산출·추진체계 조직도) 통합 서술본 (5쪽)
- v3.0: **원문은 파란 박스〔원문〕, 추가 내용은 초록 세로줄〔해설〕**로 구분한 판 (7쪽) + 문서 상단 '읽는 방법' 범례
- 소스 HTML(v2.0/v3.0)이 같은 폴더에 있어 수치·문구 수정 후 재출력 가능

## 남은 작업

1. **홈 AI 대화 → Claude API 연결** — API 크레딧 결제 카드 이슈로 대기. 팀 구독≠API(종량제) 안내 완료. 결제되면 Cloudflare Workers + Haiku 4.5 연동
2. **검증 게임 월별 갱신** — 위키 새 문서 등록 후 실행 (현재 358문항, GAME_POOL_MONTH='2026-07')
3. 사업계획서 v3.0 조직도가 3쪽으로 넘어가 2쪽 하단 여백 발생 — 요청 시 배치 조정

---

## 스킬

### 1. Windows에서 PDF 읽기·쓰기 파이프라인 (poppler 없이)
- 읽기: 한글 PDF는 pdftotext가 커스텀 폰트 인코딩으로 깨짐 → node `pdf-to-img`로 페이지를 PNG 렌더링 후 Read 도구로 읽는 것이 확실
- 쓰기: HTML(원문 편집양식 재현) → `puppeteer-core` + 로컬 Chrome로 A4 PDF 인쇄(printBackground, footerTemplate로 페이지번호)
- 검증: 출력 PDF를 다시 pdf-to-img로 렌더링해 눈으로 확인
- 주의: 이 환경의 bash python은 스텁(실행 불가), /tmp는 실제로 %LOCALAPPDATA%\Temp — node 스크립트는 상대경로로 쓰기

### 2. Supabase 오류의 실전 진단 순서
화면의 일반 오류문구를 믿지 말고, 콘솔에서 sb 클라이언트로 같은 쿼리를 직접 실행해 error.code를 본다. PGRST204=컬럼 없음, RLS violation=정책 차단, id NOT NULL=키 미지정. 각각 해결책이 다르다(스키마 수정 / SECURITY DEFINER RPC / auth.users 조회).

### 3. 원문 보존형 문서 증보 패턴
발주자 원문은 손대지 않고 박스로 감싸 '원문' 태그를 달고, 추가 해설은 시각적으로 구분된 블록에 배치 + 문서 앞에 읽는 방법 범례. 원문 수치를 재조정할 때는 해설에 "총예산에 맞춰 재조정"이라고 명시해 왜 달라졌는지 드러낸다.

## 레슨

### 1. 오류 메시지를 구체화하면 다음 디버깅이 빨라진다
"프로필 저장 실패. 관리자 문의" 같은 일반 문구 대신 실제 error.message를 화면에 노출하도록 고치자, 두 번째 오류(id NOT NULL)를 사용자 캡처만으로 즉시 진단할 수 있었다.

### 2. github.io에서 prompt()는 GitHub 창으로 오인된다
브라우저 기본 prompt는 도메인명을 표시하므로 GitHub Pages 사이트에서는 "github.io 내용:"으로 떠 사용자가 GitHub 비밀번호 요구로 오해한다. 자격증명류 입력은 반드시 폼 필드로.

### 3. 데이터 없는 자동화는 실행하지 않는다 (재확인)
"월별 문제 갱신" 요청에도 위키에 새 문서가 없음을 먼저 확인·보고하고 사용자 선택(B+C)을 받았다. 트리거만으로 재생성했으면 낭비였다.
