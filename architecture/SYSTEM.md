# TimeLink 시스템 구조 (SYSTEM)

> 전체 아키텍처. 백엔드, 프론트, Player, 로컬 에이전트.

---

## 1. 전체 구조도

┌─────────────────────────────────────────────────────┐
│                    사용자 (청취자)                    │
│                                                     │
│   [브라우저]              [TimeLink Player]         │
│   - shareplace.html       - Windows/Mac/Linux       │
│   - track.html            - .tl3 재생              │
│   - tl3-player.html                                 │
└──────────┬──────────────────────┬──────────────────┘
           │ HTTPS                │ HTTPS
           ↓                      ↓
┌─────────────────────────────────────────────────────┐
│              Cloudflare Workers (백엔드)             │
│                                                     │
│   api.timelink.digital                              │
│   - /api/v1/tl3/*  TL3 스트리밍                     │
│   - /api/v1/auth/* 인증                             │
│   - /api/shares/*  공유                             │
│   - /api/eco/*     경제                             │
│                                                     │
│   + D1 Database (메타데이터, 회계)                   │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP (Range fetch)
                       ↓
┌─────────────────────────────────────────────────────┐
│         창작자 PC (로컬 에이전트)                    │
│                                                     │
│   agent.timelink.digital (Cloudflare Tunnel)        │
│   - 원본 .tl3 파일 저장                             │
│   - Range 요청 처리                                 │
│   - PM2: timelink-agent, port 8787                 │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│              Cloudflare Pages (프론트)               │
│                                                     │
│   www.timelink.digital                              │
│   - 정적 HTML/JS/CSS                                │
│   - tl3.js (TL3 클라이언트)                         │
│   - 프로젝트: timelink-frontend                     │
└─────────────────────────────────────────────────────┘

## 2. 컴포넌트별 역할

### 2.1 백엔드 (Cloudflare Worker)

**레포:** Taolee-crypto/timelink-backend  
**URL:** https://api.timelink.digital  
**기술:** Hono + D1

**주요 라우트:**
- /api/v1/tl3/header/:id — 헤더 (T_0~T_N 스파인)
- /api/v1/tl3/segment/:id — ciphertext (복호화 X)
- /api/v1/tl3/code/:id — lic_n (TL 차감)
- /api/v1/tl3/segment/confirm/:id — 확정
- /api/v1/auth/* — 인증
- /api/shares/* — 공유
- /api/eco/* — 경제

**주요 파일:**
- src/index.ts — 메인 Worker
- src/tl3v3.ts — TL3 v3 빌더 (AES-GCM, 스파인)
- src/tl3_crypto.ts — 암호 유틸 (HKDF, HMAC, AES-GCM)
- src/tl_format.ts — TLNK 컨테이너 유틸
- src/routes/tl3.ts — TL3 API

**D1 테이블:**
- tl3_releases — 릴리스 메타
- tl3_segments — 세그먼트 offset/length
- tl3_tokens — 토큰 체인 (share_id, kind, segment_index, token)
- tl3_stream_sessions — 스트리밍 세션

### 2.2 프론트 (Cloudflare Pages)

**레포:** Taolee-crypto/timelink-frontend  
**URL:** https://www.timelink.digital  
**기술:** 순수 HTML/JS/CSS

**주요 페이지:**
- index.html — 랜딩 + 홈
- shareplace.html — 음악 공유
- creator.html — 창작자
- track.html — 재생
- dj-center.html — DJ 센터
- download.html — 앱 다운로드
- tl3-player.html — TL3 플레이어

**주요 JS:**
- public/tl3.js — TL3 클라이언트
- public/tl3-config.js — API base
- public/sidebar.js — 사이드바

### 2.3 Player (Electron 데스크톱)

**레포:** Taolee-crypto/timelink-app  
**기술:** Electron + Node.js

**주요 파일:**
- src/main.js — 메인 프로세스 (헤더 파싱, Worker 복호화)
- src/preload.js — IPC 브리지
- src/index.html — UI

**주요 기능:**
- .tl3 파일 오프라인 재생
- AES-256-GCM 복호화 (Worker Thread)
- TL 잔액 차감 (update-tl-balance)
- 재충전 (recharge-tl)
- 파일 연결 (.tl3 더블클릭)

**빌드:**
- GitHub Actions (.github/workflows/build.yml)
- v* 태그 push 시 자동 빌드
- Releases에 exe/dmg/AppImage 첨부

### 2.4 Android 앱

**레포:** Taolee-crypto/timelink-app-android  
**기술:** Kotlin/Java

## 3. 데이터 흐름

### 3.1 온라인 재생

1. 클라이언트 (track.html) → GET /api/v1/tl3/header/:id
2. 서버 → 헤더 + 스파인 반환
3. 클라이언트 → GET /api/v1/tl3/code/:id (TL 차감)
4. 서버 → lic_n 반환
5. 클라이언트 → GET /api/v1/tl3/segment/:id?n=1
6. 서버 → 창작자 PC에서 ciphertext fetch → 반환
7. 클라이언트 → 로컬 복호화 (AES-GCM)
8. MediaSource에 append → 재생
9. 반복 (n=1..N)

### 3.2 오프라인 재생 (Player)

1. 사용자가 .tl3 파일 다운로드 (track.html에서)
2. Player 실행 → .tl3 열기
3. Worker: 헤더 파싱 → T_0 검증 → 세그먼트 순회 → AES-GCM 복호화
4. mp3 조립 → 재생
5. TL 잔액 검증 (파일 헤더의 tl_balance)
6. 재생 시 TL 차감 (파일 저장)

### 3.3 파일 생성 (창작자)

1. 창작자 → creator.html 업로드
2. 클라이언트 → 로컬 에이전트 (agent.timelink.digital)
3. 로컬 에이전트: 원본 디코딩 → 세그먼트 분할 → AES-GCM 암호화 → .tl3 조립
4. 로컬 에이전트 → 서버: 메타 + 토큰체인 등록
5. 서버: D1 저장
6. 창작자 PC: 원본 .tl3 저장

## 4. 인프라

| 항목 | 값 |
|------|-----|
| 프론트 (Cloudflare Pages) | timelink-frontend.pages.dev |
| 도메인 (프론트) | www.timelink.digital |
| API (백엔드) | api.timelink.digital |
| 로컬 에이전트 터널 | agent.timelink.digital |
| 백엔드 Worker | timelink-backend.mununglee.workers.dev |
| D1 | timelink-db |
| Android | 별도 레포 |

## 5. 관련 레포

| 레포 | URL | 역할 |
|------|-----|------|
| timelink-app | github.com/Taolee-crypto/timelink-app | 데스크톱 Player |
| timelink-frontend | github.com/Taolee-crypto/timelink-frontend | 웹 플랫폼 |
| timelink-backend | github.com/Taolee-crypto/timelink-backend | API 서버 |
| timelink-app-android | github.com/Taolee-crypto/timelink-app-android | Android 앱 |
| timelink-docs | github.com/Taolee-crypto/timelink-docs | 통합 문서 (이 레포) |

## 6. 보안 계층

1. **TLS/HTTPS** — 전송 암호화
2. **JWT 인증** — 사용자 인증
3. **TL 회계** — 사용량 제한
4. **AES-256-GCM** — 파일 암호화
5. **SHA-256 체인** — 무결성 (스파인)
6. **HKDF** — 키 유도
7. **AAD (계획)** — 저작권자 정보 위조 방지

## 7. 확장성

### 7.1 타입 확장

- TL3: 음원 (현재)
- TL4: 영상 (계획)
- TLD: 문서 (계획)

**컨테이너 동일, 세그먼트 단위만 다름.**

### 7.2 로컬 스토리지 확장

- 현재: 창작자 PC (로컬 에이전트)
- 계획: 각자 서버 (사용자 소유)

### 7.3 경제 확장

- TL 충전 (결제 + 광고)
- 크리에이터 수익
- DJ 대리점
- 카페/인카 공간

---

## 8. 참조

- [핵심 원칙](../CORE_PRINCIPLES.md)
- [TL3 파일 포맷](TL3_FORMAT.md)
- [암호화](CRYPTO.md)
- [스파인 코드](SPINE_CODE.md)

---

**작성: 2026-10-11**  
**프로젝트: TimeLink**
