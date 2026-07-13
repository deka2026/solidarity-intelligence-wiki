# 아카데미 사이트 핸드오버 v11 — 홈 AI 대화 실연결용 중간 릴레이 서버 구축

> 작성일: 2026-07-13 | 작성: 데카(김일영) + Claude Code
> 사이트: https://deka2026.github.io/academy-site/
> 저장소: https://github.com/deka2026/academy-site

## 한 줄 요약

v10의 남은 작업 #1(홈 AI 대화 → Claude API 연결)을 착수했다. "사교원 팀계정에 교육생을 멤버로 추가"하는 방식은 비용 합산·통제 곤란으로 부적합하다고 판단해, **키 하나를 중간 릴레이 서버(Supabase Edge Function)에 보관하고 교육생은 그 서버를 통해 호출**하는 구조(B안)로 결정하고 코드를 완성했다. 결제·배포·SQL 실행 4단계는 사용자 몫으로 남았다.

## 배경 — 왜 팀 멤버 추가가 아니라 릴레이인가

학습가이드 7단계 "Anthropic API 키 설정"에서, 사교원 팀계정 사용자가 연결하려 하면 콘솔이 "관리자 승인 필요"를 띄운다. 이는 사이트가 아니라 console.anthropic.com 조직(멤버) 승인이다. 관리자가 Settings → Members에서 승인/초대할 수 있지만:

- 교육생을 조직 멤버로 넣으면 **각자 발급 키의 사용량·비용이 전부 사교원 청구서로 합산**되고 통제가 어렵다.
- v10 메모대로 "실연결 = 중간 서버 필요"가 맞는 방향이다.

→ 소수(스태프)만 팀에 넣는 A안이 아니라, 교육생 전체가 쓰는 **B안(릴레이 서버)** 채택.

## 이번 버전에서 완료된 작업 (커밋 전, academy-site 작업트리)

| 파일 | 작업 |
|------|------|
| `supabase/functions/claude-proxy/index.ts` | **릴레이 Edge Function 신규** — 팀 키를 secret으로만 보관, 로그인+승인(role user/admin) 검증, 1인 1일 30회 제한, Haiku 4.5 호출 |
| `supabase/claude-proxy-usage.sql` | 사용량 집계 테이블 `ai_usage`(email,day,count) + RLS on(브라우저 접근 차단) |
| `index.html` | 홈 "AI 대화" `sendChat()`를 `claude-proxy` 호출로 교체. 게스트/오류 시 기존 규칙기반 `findReply`로 자동 폴백. `aiChatHistory`로 대화 맥락 유지 |

- 검증: http-server 프리뷰에서 로드 콘솔 오류 0, 게스트 상태 질문 → 규칙 응답 정상 렌더 확인(실제 Claude는 결제·배포 후 확인 가능).
- ⚠️ **academy-site 저장소에는 아직 커밋/푸시 안 함** — 백엔드(아래 4단계) 배포와 함께 커밋하는 것을 권장(프런트만 먼저 올려도 폴백 덕에 무해하긴 함).

## 아키텍처

```
브라우저(index.html, 로그인 교육생)
   └─ sb.functions.invoke('claude-proxy', { messages })   ← Supabase 세션 토큰 첨부
        └─ Edge Function claude-proxy (Deno)
             ├─ getUser()로 로그인 확인
             ├─ profiles.role ∈ {user,admin} 확인 (pending/suspended 차단)
             ├─ ai_usage 1일 한도(30) 확인 (service_role)
             ├─ ANTHROPIC_API_KEY(secret)로 api.anthropic.com/v1/messages 호출 (Haiku 4.5)
             └─ ai_usage +1 기록 → { reply, remaining }
```
- API 키는 브라우저에 절대 노출되지 않음(secret 전용).
- 모델 `claude-haiku-4-5`, max_tokens 1024, 시스템 프롬프트는 아카데미 학습 도우미로 고정.
- 정책값(`DAILY_LIMIT`, `MAX_TOKENS`, `MODEL`)은 index.ts 상단에서 조정.

## 남은 작업 — 사용자 4단계(백엔드)

1. **Anthropic API 종량제 결제** (v10부터 막혀 있던 "카드 이슈") — console.anthropic.com → Billing에 카드 등록+크레딧 충전(⚠️ 팀 구독과 별개인 종량제 지갑) → API keys에서 `sk-ant-...` 발급
2. `supabase secrets set ANTHROPIC_API_KEY=sk-ant-...` (설정할 secret은 이것 하나뿐. URL/키는 Edge Function에 자동 주입)
3. `supabase functions deploy claude-proxy`
4. Supabase SQL Editor에 `supabase/claude-proxy-usage.sql` 실행

이후 academy-site 코드 커밋·푸시 → GitHub Pages 반영.

## 그 외 남은 작업(v10에서 이월)

- 검증 게임 월별 갱신 — 위키 새 문서 등록 후 실행 (현재 358문항, GAME_POOL_MONTH='2026-07')

---

## 스킬

### 1. Supabase Edge Function을 Claude API 릴레이로 쓰는 패턴
다수 사용자에게 개별 API 키를 나눠주지 않고 서버 하나가 대신 호출하는 구조. 핵심: (a) `ANTHROPIC_API_KEY`는 `supabase secrets set`으로만 보관—브라우저·코드에 노출 금지, (b) 호출자 인증은 anon 클라이언트 `getUser()` + `profiles.role` 확인, (c) `service_role` 클라이언트로 `ai_usage` 테이블에 1일 한도 집계, (d) `fetch('https://api.anthropic.com/v1/messages', { headers: {'x-api-key', 'anthropic-version':'2023-06-01'} })`로 프록시. URL/SERVICE_ROLE/ANON 키는 Edge Function 런타임에 자동 주입되므로 추가 secret은 API 키 하나뿐.

### 2. 기존 함수의 인증 패턴 재사용
`approve-applicant`가 이미 쓰던 "anon 클라이언트로 getUser → profiles.role 확인 → service_role로 쓰기" 3단계를 그대로 복제하면 새 함수도 일관되게 안전하다. 새 플랫폼(Cloudflare 등) 도입보다 기존 배포 워크플로(`supabase functions deploy`) 재사용이 마찰이 적다.

### 3. 정확한 모델 ID·엔드포인트는 claude-api 스킬로 확인
Anthropic/Claude 코드를 쓸 땐 기억에 의존하지 말고 `claude-api` 스킬을 먼저 읽는다. 이번엔 Haiku 4.5 = `claude-haiku-4-5`, Messages 엔드포인트/헤더(`anthropic-version: 2023-06-01`)를 스킬에서 확정했다.

## 레슨

### 1. 팀(구독) 계정 ≠ 종량제 API 지갑
사교원 팀계정 관리자 승인으로 교육생을 조직 멤버에 넣을 수는 있으나, 그러면 전원의 사용량이 팀 청구서로 합산돼 통제가 어렵다. "다수가 API를 쓴다"면 멤버 추가가 아니라 **키 하나 + 릴레이 서버**가 정석. 팀 구독(claude.ai)과 종량제 API(console) 결제가 별개라는 점도 매번 짚어줘야 한다.

### 2. 실연결 전이라도 프런트는 폴백으로 안전하게
릴레이 미배포·한도초과·비로그인 상황에서 기존 규칙기반 `findReply`로 폴백하도록 만들면, 백엔드 결제/배포가 끝나기 전에 프런트를 올려도 사용자 경험이 깨지지 않는다. 실제 응답 검증은 배포 후로 미루더라도 JS 무결성·폴백 경로는 프리뷰에서 즉시 검증할 수 있다.

### 3. 비용 통제 장치는 처음부터 코드에 넣는다
B안의 목적 자체가 비용 통제이므로 1인 1일 한도(`ai_usage` 테이블)와 저비용 모델(Haiku)·max_tokens 상한을 릴레이에 기본 탑재했다. "나중에 추가"가 아니라 첫 구현에 포함해야 폭주를 막는다.
