# 핸드오버: 사교원 연대지능활동가 아카데미 사이트 (v6)

**작성일:** 2026-06-29  
**작성자:** Claude (데카 요청)  
**이전 핸드오버:** `docs/handover-academy-site-v5.md` (2026-06-29)

---

## 이번 세션 작업 요약

### 완료된 작업 — 2단계 (Supabase 백엔드 통합)

1. **Supabase 데이터 동기화 레이어**
   - Supabase JS SDK 연동 (`@supabase/supabase-js@2` CDN)
   - 프로젝트: `solidarity intelligence` / URL: `https://gklecgujcoznxyvywnyu.supabase.co`
   - `sbSync()`: 페이지 로드 시 feedback, verify_records, github_map, wiki_contrib 4개 테이블 → localStorage 동기화
   - `sbSaveFeedback()`, `sbAddVerifyRecord()`, `sbSaveGithubMap()`, `sbSaveWikiContrib()`: 비동기 Supabase 저장
   - localStorage는 캐시로 유지 (오프라인 호환)

2. **Supabase 테이블 5개 + RLS**
   - `profiles` (id→auth.users, email, name, role, created_at) — RLS: `FOR ALL USING(true) WITH CHECK(true)`
   - `feedback` (id, author, email, category, content, date, likes, replies)
   - `verify_records` (id, email, date, answered, pts, details)
   - `github_map` (email PK, github_user)
   - `wiki_contrib` (github_user PK, docs, merged, reviews, improvements, links, verify_pts, prs)
   - feedback, verify_records, github_map, wiki_contrib: 모두 `FOR ALL USING(true) WITH CHECK(true)`

3. **Supabase Auth 인증 전환 (부분 완료)**
   - `requestAccess()`: `sb.auth.signUp()` 사용 (⚠️ 이메일 전송 문제로 현재 동작 안 함)
   - `doLogin()`: `sb.auth.signInWithPassword()` + profiles 역할 확인
   - `doAdminLogin()`: 관리자PW `2026` 폴백 유지
   - `logout()`: `sb.auth.signOut()` + localStorage 클리어
   - `sbCheckAuth()`: 페이지 로드 시 Supabase 세션 자동 복원
   - `renderPending()`, `approveUser()`, `rejectUser()`, `renderApproved()`: Supabase profiles 테이블 기반으로 전환 완료

4. **관리자 계정 생성 완료**
   - `sakyowon@sakyowon.co.kr` / role: admin / Supabase Auth + profiles 등록
   - Supabase 대시보드 > Add user > Auto Confirm으로 생성

5. **품질 리뷰 주간 리포트 Word 내보내기**
   - `exportWeeklyReport()`: 주간 피드백을 카테고리별로 정리 → Claude 수정 프롬프트 포함 → .doc 다운로드

6. **GitHub Pages 배포 완료**
   - 커밋: `2b55ddf` — "2단계: Supabase 백엔드 통합 — 데이터 동기화 + 인증 전환"

### ⚠️ 미완료 — 반드시 이어야 할 것

1. **교육생 가입(signUp) 이메일 인증 문제 해결**
   - 현상: `sb.auth.signUp()` → 500 에러 (트리거 충돌) 또는 429 에러 (이메일 전송 한도 초과)
   - 원인: Supabase 무료 플랜 이메일 전송 제한 (2회/시간) + "Confirm email" 비활성화 토글을 UI에서 찾지 못함
   - 트리거 `on_auth_user_created`는 현재 제거된 상태 (트리거가 signUp 500 에러의 원인이었음)
   - **해결 방안** (택 1):
     - A) Supabase 대시보드에서 "Confirm email" 비활성화 찾기 → 트리거 재생성 (ON CONFLICT DO NOTHING 포함)
     - B) signUp 대신 `applications` 테이블에 신청 저장 → 관리자가 대시보드에서 Auth 사용자 수동 생성 (Add user + Auto Confirm)
     - C) Supabase Edge Function으로 auto-confirm signUp 구현

2. **트리거 재설정** (방안 A 선택 시)
   ```sql
   CREATE OR REPLACE FUNCTION handle_new_user()
   RETURNS trigger AS $$
   BEGIN
     INSERT INTO profiles (id, email, name, role)
     VALUES (NEW.id, NEW.email, COALESCE(NEW.raw_user_meta_data->>'name', ''), 'pending')
     ON CONFLICT DO NOTHING;
     RETURN NEW;
   END;
   $$ LANGUAGE plpgsql SECURITY DEFINER;

   CREATE TRIGGER on_auth_user_created
     AFTER INSERT ON auth.users
     FOR EACH ROW EXECUTE FUNCTION handle_new_user();
   ```

3. **requestAccess() 에러 메시지 개선**
   - 현재 `escHtml(res.error.message)` → 빈 객체 `{}` 표시됨
   - `res.error.message || res.error.code || JSON.stringify(res.error)` 로 변경 필요

---

## 기술 상세

### 호스팅 + Supabase
| 항목 | 값 |
|---|---|
| GitHub 레포 | `deka2026/academy-site` |
| 공개 URL | https://deka2026.github.io/academy-site/ |
| 로컬 소스 | `C:\Users\Admin\Desktop\인공지능 작업실\academy-site\index.html` |
| 최신 커밋 | `2b55ddf` — "2단계: Supabase 백엔드 통합" |
| Supabase 프로젝트 | `solidarity intelligence` |
| Supabase URL | `https://gklecgujcoznxyvywnyu.supabase.co` |
| Supabase Anon Key | `sb_publishable_gedQpfHqmnd1hlGNnrglLw_n5b3f3YN` |
| 관리자 계정 | `sakyowon@sakyowon.co.kr` (role: admin) |
| 관리자PW 폴백 | `2026` |
| Preview 경로 | `C:\Users\Admin\academy-site-link` (junction → 한글 경로 우회) |

### Supabase 현재 상태
| 항목 | 상태 |
|---|---|
| 데이터 동기화 (4테이블) | ✅ 정상 동작 |
| Auth 로그인/로그아웃 | ✅ 정상 동작 |
| Auth 세션 복원 | ✅ 정상 동작 |
| Auth 가입 (signUp) | ❌ 이메일 인증 문제 |
| 트리거 (on_auth_user_created) | ❌ 제거된 상태 |
| profiles RLS | ✅ 전체 허용 (FOR ALL) |

### localStorage 키 (기존 호환 유지)
| 키 | 용도 |
|---|---|
| `academy_pending` | 교육 신청 대기 (로컬 폴백) |
| `academy_users` | 사용자 DB (로컬 폴백) |
| `academy_session` | 현재 로그인 세션 |
| `academy_wiki_contrib` | 위키 기여 데이터 |
| `academy_github_map` | 이메일↔GitHub 매핑 |
| `academy_verify_game` | 검증 게임 기록 |
| `academy_feedback` | 의견 게시판 데이터 |

---

## 다음 세션 작업 순서 (권장)

1. **signUp 이메일 인증 문제 해결** (위 해결 방안 A/B/C 중 택 1)
2. **requestAccess() 에러 표시 수정** (`res.error.message` → 폴백 추가)
3. **테스트 사용자 가입 → 승인 → 로그인 전체 플로우 검증**
4. 커스텀 도메인 연결 (DNS CNAME 설정)
5. 활동가 테스트 검수 결과 반영
