Phaser 웹게임 플랫폼 기획서 (SvelteKit 기반)

목적: SvelteKit + Phaser로 단일 개발자(너)의 게임들을 배포/실행/랭킹/분석까지 제공하는 경량 플랫폼을 만든다. 결제/업로드/심사 없음, 오직 네가 만든 게임만 노출된다. 이 문서는 AGENTS.md 용으로, 역할별(아키텍트/프론트/백엔드/SDK/인프라/QA/콘텐츠)로 바로 실행 가능한 스펙/절차를 담는다.

0. MVP 범위 (1~2주 목표)

게임 허브: 네가 등록한 게임 리스트(카드/검색/태그)

게임 상세: 스크린샷, 설명, 조작법, 버전 로그

런처: iframe 샌드박스 실행 + 일시정지/재시작 + 전체화면 + 로딩 인디케이터

리더보드: 글로벌 Top/N + 내 기록, 기간 탭(전체/주간/일간)

세이브 데이터: 로컬 저장(IndexedDB) + 선택적 클라우드 동기화

프로필/설정(옵션): 게스트 플레이 + 닉네임, 언어/볼륨/키맵 저장

분석: 세션 시작/종료, 로딩지표(First Input까지), 스코어 이벤트

PWA: 홈 화면 추가, 오프라인 캐시(허브/런처 셸 + 핵심 에셋)

제외: 결제, 서드파티 개발자 업로드/심사, 멀티플레이(차후)

1. 기술 스택

App: SvelteKit(SSR + ISR, Vite) / TypeScript

Game Engine: Phaser 3 (3.7x 이상 권장)

Auth/DB/Storage: Supabase (Postgres + RLS, Auth, Storage) — 게스트 우선, 소셜은 선택

Analytics: PostHog(클라이언트 이벤트) + Vercel Analytics(웹)

Deploy: Vercel(웹) / Supabase(데이터) / 이미지 최적화는 Vercel Image Optimization

Styling/UI: Tailwind + Shadcn-svelte(또는 Skeleton) + Radix 아이디어 차용

Tooling: Biome/ESLint, Prettier, Vitest/Playwright, MSW(모킹)

2. 상위 아키텍처

[사용자] ─ SvelteKit(App) ─ (+server.ts API) ─ Supabase(Postgres/RLS)
                               │
                               ├─ Storage(스크린샷/포스터/정적 에셋)
                               └─ Realtime(리더보드 반영/Presence)

런처 분리: 게임은 iframe으로 샌드박스. 플랫폼↔게임은 postMessage 프로토콜(V1) 사용.

게임 정적 호스팅: /_games/{slug}/index.html 로 서빙(프로젝트 리포 내 서브모듈 또는 Storage 싱크)

보안: RLS로 공개 게임만 읽기, 점수 제출은 세션 토큰 필요. iframe은 sandbox 최소 권한.

3. 디렉토리 구조 (SvelteKit)

apps/web
  ├─ src
  │  ├─ routes
  │  │  ├─ +layout.svelte
  │  │  ├─ +layout.ts               # load hooks(프로필/설정 프리패치)
  │  │  ├─ +page.svelte             # 홈(피처드/최근 플레이)
  │  │  ├─ games/+page.svelte       # 허브(리스트/필터)
  │  │  ├─ games/[slug]/+page.svelte# 상세(SSG/ISR)
  │  │  ├─ play/[gameId]/+page.svelte  # 런처(iframe)
  │  │  ├─ api/games/+server.ts     # 목록/검색
  │  │  ├─ api/games/[slug]/+server.ts # 상세
  │  │  ├─ api/leaderboard/[gameId]/+server.ts  # GET/POST
  │  │  └─ api/session/+server.ts   # 플레이 세션 발급
  │  ├─ lib
  │  │  ├─ supabase.ts
  │  │  ├─ analytics.ts
  │  │  ├─ sdk/postmessage.ts       # 플랫폼 SDK(런처측)
  │  │  ├─ storage.ts
  │  │  └─ pwa/sw.ts                # Service Worker
  │  ├─ hooks.server.ts             # 쿠키/세션, locale
  │  ├─ app.d.ts
  │  └─ styles
  ├─ static/_games/                 # 네 게임 빌드물(서브모듈 또는 CI 동기화)
  ├─ package.json
  └─ svelte.config.js

packages/game-sdk    # 게임 측에서 import하는 경량 SDK(UMD/ESM)
packages/ui          # 컴포넌트 라이브러리
infra/               # IaC, CI/CD, DB 마이그레이션

4. 데이터 모델 (Supabase / Postgres)

profiles

id(uuid, pk = auth.uid), nickname, avatar_url, locale, settings(jsonb), created_at

games

id(uuid, pk), slug(unique), title, summary, description_md, version, status(enum: public|hidden),

poster_url, screenshots(jsonb), controls(jsonb), tags(text[]), storage_prefix, content_hash,

created_at, updated_at

leaderboards

id(uuid), game_id(fk), user_id(nullable: 게스트 null), score(int), meta(jsonb), period(enum: all|weekly|daily), created_at

인덱스: (game_id, period, score DESC, created_at ASC), (user_id, game_id)

sessions

id(uuid), game_id, user_id(null 가능), client_fingerprint, started_at, ended_at, device, country

changelogs (옵션)

id, game_id, version, notes_md, created_at

RLS

games.status = 'public'만 익명 읽기 허용

leaderboards: INSERT는 서버에서 서명 세션 토큰 검증 후 수행(Edge Function 또는 +server.ts)

5. API 설계 (+server.ts)

공용

GET /api/games?q=&tag=&page= → 게임 리스트(캐시 가능)

GET /api/games/[slug] → 상세

POST /api/session → 플레이 세션 토큰 발급 { gameId }

GET /api/leaderboard/[gameId]?period=all|weekly|daily&me=1 → 랭킹 조회

POST /api/leaderboard/[gameId] → 점수 제출 { score, meta } (헤더 X-Session-Token)

응답 공통

{ "ok": true, "data": { ... }, "error": null }

에러는 { code, message, hint } 포함.

6. 런처 & 메시지 프로토콜

6.1 Iframe 런처

경로: /play/[gameId]

속성: sandbox="allow-scripts allow-pointer-lock allow-same-origin", allowfullscreen

기능: 로딩 스켈레톤 → READY 수신 시 시작 / 포커스 아웃 시 일시정지(옵션) / 전체화면 토글

6.2 postMessage V1

// 플랫폼 → 게임
{ type: 'INIT', payload: { user: { id?, nickname? }, lang, saveData } }
{ type: 'PAUSE' | 'RESUME' }
{ type: 'SETTINGS', payload: { volume, keymap } }

// 게임 → 플랫폼
{ type: 'READY' }
{ type: 'LOADED', payload: { assetsMs } }
{ type: 'SCORE_SUBMIT', payload: { score: number, meta?: Record<string, any> } }
{ type: 'SAVE', payload: { key: string, value: any } }        // 선택적 클라우드 세이브 훅
{ type: 'ERROR', payload: { code, message } }

6.3 @platform/game-sdk (게임 측)

번들: UMD/ESM, < 4KB, 타입 정의 제공

API: init(), on(event, cb), emit(event, payload), submitScore(score, meta?), save(key, value)

설치: import { init, submitScore } from '@platform/game-sdk'

7. 빌드/배포 파이프라인

게임 리포에서 pnpm build → dist/ 산출

플랫폼 리포의 static/_games/{slug}/에 복사(CI 동기화 스텝)

games 테이블의 storage_prefix=/static/_games/{slug} 및 content_hash 갱신 스크립트 실행

Vercel 배포 시 정적 경로로 서빙, 캐시 무효화(서브패스 캐시 purge)

옵션: 게임 에셋을 Supabase Storage에 넣고 빌드시 static/으로 미러링

8. 리더보드 설계 & 안티치트

세션 토큰: /api/session에서 gameId별 토큰 발급. 제출 시 헤더 검증 + 만료(예: 2시간)

서버 기준 시간: 점수 제출 타임스탬프는 서버가 결정

이상치 탐지: score 상한, 빈도 제한, 분포 기반 아웃라이어 컷(상위 0.1%)

동점 처리: 먼저 달성한 기록을 상위로

신고 대신: 단일 개발자 운영이므로 관리 탭 없이 로그/알림으로 대체(PostHog/Slack Webhook)

9. PWA & 성능 전략

Service Worker: 허브/런처 셸 pre-cache, 게임 에셋은 runtime cache(최대용량/만료 정책)

Code-splitting: 런처/허브 분리, 이미지 lazy, 스크린샷 LQIP

폰트/이미지: next-gen 포맷(avif/webp), srcset

메트릭: TTFB/INP/FCP 추적, 게임 내 LOADED.assetsMs 수집

10. 페이지 & UX 플로우

홈: 피처드 + 최근 플레이(로컬 히스토리)

허브(games): 태그/검색/정렬, 카드에 플레이 버튼

상세(games/[slug]): 포스터/스크린샷, 조작법, 버전/업데이트, 플레이타임 통계

런처(play/[gameId]): 전체화면, 일시정지, 리더보드 사이드패널, 키맵/볼륨 퀵설정

오프라인: 허브/상세/최근 실행 게임 1개까지 오프라인 실행 허용(캐시 상태 표시)

11. 환경 변수

PUBLIC_POSTHOG_KEY=
PUBLIC_SUPABASE_URL=
SUPABASE_SERVICE_ROLE=        # 서버 사이드만 사용
PUBLIC_SUPABASE_ANON_KEY=
SESSION_SECRET=

12. 초기 스키마 마이그레이션(SQL 초안)

create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  nickname text,
  avatar_url text,
  locale text,
  settings jsonb default '{}'::jsonb,
  created_at timestamptz default now()
);

create table games (
  id uuid primary key default gen_random_uuid(),
  slug text unique not null,
  title text not null,
  summary text,
  description_md text,
  version text,
  status text check (status in ('public','hidden')) default 'public',
  poster_url text,
  screenshots jsonb default '[]'::jsonb,
  controls jsonb default '{}'::jsonb,
  tags text[],
  storage_prefix text,
  content_hash text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create table leaderboards (
  id uuid primary key default gen_random_uuid(),
  game_id uuid not null references games(id) on delete cascade,
  user_id uuid references auth.users(id) on delete set null,
  score int not null,
  meta jsonb default '{}'::jsonb,
  period text check (period in ('all','weekly','daily')) default 'all',
  created_at timestamptz default now()
);

create index on leaderboards (game_id, period, score desc, created_at asc);
create index on leaderboards (user_id, game_id);

create table sessions (
  id uuid primary key default gen_random_uuid(),
  game_id uuid references games(id) on delete cascade,
  user_id uuid references auth.users(id) on delete set null,
  client_fingerprint text,
  started_at timestamptz default now(),
  ended_at timestamptz
);

RLS 정책 개요

games: status='public' 읽기 허용(익명). 쓰기는 관리자(너)만

leaderboards: insert/update는 Edge/서버 라우트에서만(서명 토큰 검증 후) 수행

13. 서버 라우트 시그니처 (SvelteKit)

// GET /api/games
// query: q, tag, page
// cache: 60s ISR

// GET /api/games/[slug]
// returns detailed game meta

// POST /api/session
// body: { gameId }
// returns: { token, expiresAt }

// GET /api/leaderboard/[gameId]?period=&me=
// returns: { top: [...], me: {...}? }

// POST /api/leaderboard/[gameId]
// headers: { 'X-Session-Token': '...' }
// body: { score, meta }

14. 게임 SDK(게임 프로젝트에서 사용)

import { init, on, submitScore, save } from '@platform/game-sdk';

init({
  onReady: () => { /* 자산 로드 이후 READY 전송 */ },
  onPause: () => { /* 게임 일시정지 */ },
  onResume: () => { /* 게임 재개 */ },
  onSettings: (s) => { /* 볼륨/키맵 반영 */ },
});

submitScore(12345, { stage: 3, time: 42 });
save('slot1', { hp: 100, inv: ['sword'] });

번들 타깃: ESM + UMD(브라우저 <script> 삽입용), 타입 정의 포함.

15. CI/CD

게임 빌드 동기화: GitHub Actions에서 게임 리포 dist/를 플랫폼 리포 static/_games/{slug}로 복사 후 커밋

해시 계산: 변경 시 content_hash 업데이트 → 캐시 무효화

DB 마이그레이션: Supabase db push

배포: Vercel 프리뷰 → 테스트 통과 시 프로덕션 프로모트

16. QA 체크리스트



17. 롤아웃 계획

단일 게임(대표작) 1개로 퍼블릭 오픈

리더보드/분석 지표 안정화(1~2주)

추가 게임 1~2종 순차 등록(정적 싱크) → 허브 UX 보완

18. 작업 분배(AGENTS)

ARCH: 라우팅/데이터 모델/보안 정책 시드, PWA 셸 설계

FE: 허브/상세/런처 UI, Tailwind/컴포넌트, 접근성

BE: +server.ts API, 세션 토큰, 리더보드 쿼리/인덱스, RLS

SDK: packages/game-sdk 설계/번들/테스트, postMessage 규약 확정

OPS: CI 동기화(게임→static), 해시/캐시, Vercel/Supabase 환경

QA: 체감 성능/오프라인/안티치트 시나리오

CONTENT: 게임 메타/스크린샷/조작법/버전노트 정리

19. 향후 확장(참고)

멀티플레이(Colyseus) / 친구 랭킹 / 업적 & 배지 / 스냅샷 리플레이

간단한 모드(스킨/테마) 적용 시스템

20. 개발 착수 절차 (Day 0)

Supabase 프로젝트 및 테이블 생성(마이그레이션 적용)

SvelteKit 템플릿 부트스트랩 → Tailwind/BIOME 세팅

games 시드 데이터(대표작 1개) 삽입 + 정적 에셋 배치

런처 페이지 구현(iframe + postMessage V1)

리더보드 GET/POST 라우트, 세션 토큰 발급 라우트

PWA 셸 및 기본 캐시 전략 적용

PostHog 이벤트 스키마 정의 후 페이지/게임 이벤트 연동

## PR 기록
- 2025-09-08: apps/web 기본 SvelteKit 구조 추가
