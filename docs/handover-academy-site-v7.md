# 핸드오버: 사교원 연대지능활동가 아카데미 사이트 (v7)

**작성일:** 2026-06-29  
**작성자:** Claude (데카 요청)  
**이전 핸드오버:** `docs/handover-academy-site-v6.md` (2026-06-29)

---

## 이번 세션 작업 요약

### 완료된 작업 — signUp 이메일 문제 해결

1. **교육 신청 방식 전환: `sb.auth.signUp()` → `applications` 테이블**
   - 문제: Supabase 무료 플랜 이메일 전송 한도(2회/시간) + "Confirm email" 비활성화 토글 UI에서 찾지 못함 + 트리거 충돌로 signUp 500 에러
   - 해결: Auth signUp을 완전히 제거하고, 신청 데이터를 `applications` 테이블에 직접 저장
   - 비밀번호 필드 제거 (Auth 계정은 관리자가 Supabase 대시보드에서 생성)

2. **Supabase `applications` 테이블 생성**
   - 구조: `id` (bigint, auto), `email` (unique), `name`, `region`, `phone`, `site`, `created_at`
   - RLS: `FOR ALL USING(true) WITH CHECK(true)`

3. **교육생 등록 플로우 (최종)**
   - 교육생: `#apply` 페이지에서 이름/지역/전화번호/이메일 입력 → `applications` 테이블 저장
   - 관리자: 교육생 관리 페이지에서 "승인" → `applications`에서 삭제 + Supabase 대시보드 안내
   - 관리자: Supabase Dashboard > Authentication > Add user > Auto Confirm ✅ 로 계정 생성
   - 교육생: 관리자가 설정한 비밀번호로 로그인

4. **수정된 함수**
   - `requestAccess()`: applications 테이블 INSERT + 중복 확인 (applications, profiles)
   - `renderPending()`: applications 테이블에서 조회 + localStorage 호환
   - `approveUser()`: applications 삭제 + 대시보드 안내 alert
   - `rejectUser()`: applications 삭제

5. **배포 완료**
   - 커밋: `32770ad` — "signUp 이메일 문제 우회: applications 테이블 기반 교육 신청으로 전환"

### Supabase 전체 현황 (최종)
| 항목 | 상태 |
|---|---|
| 데이터 동기화 (4테이블) | ✅ 정상 |
| Auth 로그인/로그아웃 | ✅ 정상 |
| Auth 세션 복원 | ✅ 정상 |
| 교육 신청 (applications) | ✅ 정상 |
| 교육생 승인/거절 | ✅ 정상 |
| 관리자 계정 | ✅ sakyowon@sakyowon.co.kr (admin) |
| 트리거 (on_auth_user_created) | ❌ 제거됨 (불필요) |

---

## 기술 상세

### 호스팅 + Supabase
| 항목 | 값 |
|---|---|
| GitHub 레포 | `deka2026/academy-site` |
| 공개 URL | https://deka2026.github.io/academy-site/ |
| 로컬 소스 | `C:\Users\Admin\Desktop\인공지능 작업실\academy-site\index.html` |
| 최신 커밋 | `32770ad` |
| Supabase 프로젝트 | `solidarity intelligence` |
| Supabase URL | `https://gklecgujcoznxyvywnyu.supabase.co` |
| Supabase Anon Key | `sb_publishable_gedQpfHqmnd1hlGNnrglLw_n5b3f3YN` |
| 관리자 계정 | `sakyowon@sakyowon.co.kr` (role: admin) |
| 관리자PW 폴백 | `2026` |
| Preview junction | `C:\Users\Admin\academy-site-link` |

### Supabase 테이블 (6개)
| 테이블 | 용도 | RLS |
|---|---|---|
| `profiles` | 사용자 프로필 (auth.users FK) | FOR ALL permissive |
| `applications` | 교육 신청 대기 | FOR ALL permissive |
| `feedback` | 의견 게시판 | FOR ALL permissive |
| `verify_records` | 검증 게임 기록 | FOR ALL permissive |
| `github_map` | 이메일↔GitHub 매핑 | FOR ALL permissive |
| `wiki_contrib` | 위키 기여 데이터 | FOR ALL permissive |

---

## 미완료 (다음 세션)

1. **커스텀 도메인 연결** — DNS CNAME 설정 (사용자 직접)
2. **활동가 테스트 검수 결과 반영** — 엑셀 전달 후 진행
3. **RLS 정책 강화** — 현재 전체 허용 상태, 프로덕션 전에 역할별 제한 필요
4. **교육생 승인 자동화** — 현재 관리자가 Supabase 대시보드에서 수동 Auth 생성, Edge Function으로 자동화 가능
