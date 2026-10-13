# TimeLink Architecture

최종 수정: 2026-10-13

## 1. 시스템 개요

TimeLink = **디지털 파일 자산화 플랫폼**

- 파일을 "팔지" 않는다. **공유**한다.
- TL 파일 = **디지털 자산**
- TL이 없는 TL 파일 = **무용지물**

## 2. 5개 레포지토리

| 레포 | 역할 | 기술 |
|---|---|---|
| `timelink-docs` | **공식 문서** (Single Source of Truth) | Markdown |
| `timelink-frontend` | 웹 플랫폼 | Cloudflare Pages (정적 HTML/JS) |
| `timelink-backend` | API 서버 | Cloudflare Workers + Hono + D1 |
| `timelink-app` | 데스크톱 Player | Electron (Windows/macOS/Linux) |
| `timelink-score` | TLD + AI 분석 | Python + FastAPI |

## 3. 3가지 파일 타입

| 타입 | 대상 | 컨테이너 | 세그먼트 | 상태 |
|---|---|---|---|---|
| **TL3** | 음악 | TLNK v3 | 1초 | ✅ 작동 |
| **TLJ** | 이미지 | TLNK v3 | 1청크 | 🟡 개발 중 |
| **TLD** | 문서/악보 | TLNK v3 | 1페이지 | ✅ 검증 완료 |
| **TL4** | 영상 | TLNK v3 | 1초 | ⚪ 예정 |

**모두 동일한 TLNK 컨테이너를 사용.** 내용만 다름.

## 4. 인프라

| 항목 | 값 |
|---|---|
| **프론트 (Cloudflare Pages)** | `timelink-frontend.pages.dev` |
| **도메인** | `www.timelink.digital` |
| **API** | `api.timelink.digital` |
| **로컬 에이전트 터널** | `agent.timelink.digital` (8787, 오디오) |
| **TLD 서버 터널** | `tld.timelink.digital` (8000, TLD) |
| **백엔드 레포** | `Taolee-crypto/timelink-backend` |

## 5. 데이터 흐름

### 5.1 자산화 (제조)
창작자 PC
├─ 원본 파일 (mp3, jpg, pdf...)
├─ 로컬 에이전트 (PM2: timelink-agent, 8787)
│ ├─ 디코딩
│ ├─ 세그먼트 분할
│ ├─ AES-256-GCM 암호화 (스파인)
│ ├─ 타임토큰 체인 생성
│ └─ .tl3/.tlj/.tld/.tl4 조립
└─ 서버로 메타 + 토큰체인 등록
↓
Cloudflare Workers (api.timelink.digital)
└─ D1 저장 (메타데이터)

text

### 5.2 사용 (재생/열람)
사용자 (클라이언트)
↓ /api/v1/tl3/segment/:id?n=1&session_id=xxx
Cloudflare Workers
├─ TL 잔액 검증
├─ 세션 상태 검증
├─ T_{n-1} 확인
├─ 창작자 PC에서 세그먼트 fetch (Range)
├─ AES-256-GCM 복호화
├─ T_n 갱신 + D1 저장
├─ TL 차감
└─ 평문 반환
↓
사용자 (이어붙임 → 재생/표시)

text

## 6. 보안 (스파인 기능)

### 6.1 키 유도
lic = HKDF(master_secret, "tl3:lic:" + shareId) [서버만]
T_0 = SHA256(fid ‖ hash ‖ salt ‖ rights_hash)
k_n = SHA256(T_{n-1} ‖ lic ‖ "K" + n) [AES-GCM 키]
T_n = SHA256(T_{n-1} ‖ fid ‖ str(n) ‖ SHA256(ct_n))

text

### 6.2 방어 메커니즘

| 시도 | 결과 |
|---|---|
| 시간 역행 | T_n → T_{n-1} 역산 불가 (해시 일방향) |
| 건너뛰기 | T_0 → T_5 계산 불가 (중간 토큰 필요) |
| 재조립 | 세그먼트 순서 바꾸면 키 다름 → 복호화 실패 |
| 파일 복사 | 서버 없이는 lic 계산 불가 → 복호화 불가 |
| mp3 파괴 | 서버 복호화 없이는 쓰레기 바이트 |
| 저작권자 위조 | AAD 검증 실패 |

## 7. 배포

| 대상 | 방법 |
|---|---|
| **프론트** | git push → Cloudflare Pages 자동 (30초) |
| **백엔드** | `npx wrangler deploy` |
| **Player** | git tag vX.Y.Z → GitHub Actions 자동 빌드 |
| **TLD 서버** | Python FastAPI (포트 8000) + Cloudflare Tunnel |
| **docs** | git push → GitHub |

## 8. 참조

- 핵심 원칙: `CORE_PRINCIPLES.md`
- 용어집: `LEXICON.md`
- 화면 구조: `IA.md`
- TL3 파일 포맷: `architecture/TL3_FORMAT.md`
- 암호화: `architecture/CRYPTO.md`
- 스파인 코드: `architecture/SPINE_CODE.md`
- 시스템 구조: `architecture/SYSTEM.md`
- 특허: `PATENTS.md`

---

최초 작성: 2026-10-13
프로젝트: TimeLink
