# TimeLink 핵심 원칙 (Core Principles)

> **"파일을 파는 것이 아니라, 사용할 권리를 판매합니다."**
> — TimeLink, 특허 출원 10-2025-0167813

---

## 1. TL 파일 철학

**시간 충전형 디지털 파일.** 파일은 완성된 실체가 아니라, TL(TimeLink Token)이 있는 동안만 "실체화"되는 잠재 상태입니다. TL을 다 쓰면 파일은 무력화됩니다.

**적용 대상:** 음원(TL3), 영상(TL4), 문서(TLD), 이미지 등 모든 디지털 콘텐츠.

---

## 2. 6대 원칙 (파일 제조)

### 원칙 1: TL이 있는 만큼만 사용 가능

- 파일은 완성된 실체가 아니다
- TL이 있는 동안만 "실체화"된다
- TL 소진 = 파일 무력화 = 사용 불가
- 사용자가 mp3/mp4로 파괴해도, 서버 없이는 쓰레기 바이트만 나온다

### 원칙 2: 스파인 기능 (Spine Function)

- 순방향(정상 사용) = 매끄러움
- 역방향/우회 = 손상 (뚫어도 쓸모없게)
- **Time Spine:** T_0 → T_1 → ... → T_n (역산 불가)
- **Order Spine:** 세그먼트 순서 강제
- **Server Spine:** 매 요청 서버 검증
- **Lic Spine:** master_secret 없이는 키 유도 불가

### 원칙 3: LP 및 확장 기능은 별도 모듈

- TL 파일 = 순수 원본 (mp3, mp4, pdf 등)
- 확장 기능(LP 음질, 필터, 뷰어) = 별도 JS/TS 모듈
- 재생/열람 시 선택적 호출, 파일 저장 X
- 음원: lp.js / 영상: video-ext.js / 문서: doc-viewer.js

### 원칙 4: 저장 위치 = 창작자 PC

- TL 파일 원본 = 창작자 PC (로컬 에이전트)
- 서버 = 중계자 (복호화 + TL 검증)
- D1 = 메타데이터
- 창작자가 원본 소유, 서버는 접근만 중계

### 원칙 5: TLNK 컨테이너 (모든 타입 공통)

[0..3]   "TLNK" (magic 0x54 0x4C 0x4E 0x4B)
[4]      version (0x03)
[5..8]   headerLen (uint32 BE)
[9..]    header JSON (메타)
[세그먼트 × N]  (AES-256-GCM, 세그먼트별 다른 키)

- 음원(TL3), 영상(TL4), 문서(TLD) 모두 동일 컨테이너
- 컨테이너 동일, 내용만 다름
- 세그먼트 단위만 타입별로 다름 (음원=1초, 영상=1초, 문서=1페이지)

### 원칙 6: 저작권자 정보 삽입 (Creator Attribution)

- 모든 TL 파일은 저작권자 정보를 암호학적으로 포함
- 삽입 위치:
  1. 헤더 (평문) — 법적 요구사항
  2. AAD (인증) — 위조 방지
  3. 타임토큰 체인 (스파인) — T_0에 종속
  4. 워터마크 (재생) — 캡처 추적
- 구성:
  - 창작자 ID, 이름, TON 지갑
  - 라이선스 타입, 조건
  - 국제 표준 코드 (ISRC/ISBN/ISAN)
  - 공동 창작자 (composer, lyricist, producer...)

---

## 3. 개발 실무 원칙

### 3.1 mp3 재생 금지 (절대 원칙)

**우리 플랫폼에서는 mp3 파일을 절대 재생하지 않는다.**

- 업로드된 원본 mp3는 다운로드만 가능
- 플랫폼 내 재생은 오직 .tl3 파일만
- .mp3 URL을 audio/video에 직접 넣는 것 금지
- mp3는 크리에이터가 "원본 보관용"으로만 받아감

이유:
1. 저작권 보호 — mp3 원본 유통 방지
2. 과금 정확성 — mp3 직접 재생 = TL 차감 우회
3. v3 정체성 — TL3(AES-GCM + 타임토큰 체인)를 통해서만 재생

적용 범위: track.html, dj-center.html, player-host.html, shareplace.html, cafe-radio.html, tl3-player.html, 오프라인 플레이어(playFile)

예외: 없음. 무조건.

### 3.2 TL3 파일 원칙

- .tl3 = mp3 + 시간 + 저작권 + AES-256-GCM + 타임토큰 체인
- 재생은 반드시 클라이언트에서 복호화 후
- 서버는 ciphertext만 전달
- LP 효과는 lp.js가 실시간 처리 (파일 저장 X)

### 3.3 파일 저장 원칙

- 원본 파일: 로컬 PC (창작자), 메타: D1
- .tl3 파일: D1 object (tl3/releases/{fileId}.tl3)
- 세그먼트 위치: D1 tl3_segments (offset/length)
- 토큰: D1 tl3_tokens (share_id, kind='mp3', segment_index, token)

### 3.4 암호화 원칙

lic = HKDF(master, info="license:"+shareId)   [서버만]
T_0 = SHA256(fid + hash_mp3 + salt)
k_n = SHA256(T_{n-1} + lic + "K"+n)
iv_n = [4B 0][8B n] (big-endian)
T_n = SHA256(T_{n-1} + fid + str(n) + SHA256(ct_n))

- AES-256-GCM, tag 128bit
- 세그먼트 = 1초 (SEG_SECONDS=1)
- 매 초 다른 키

### 3.5 오프라인 원칙

- 오프라인 재생은 .tl3 파일 다운로드 후
- 파일에 lic, mp3_tokens, mp3_seg_lens, fid, salt, hash_mp3 포함 (자급자족)
- 온라인은 기존 스트리밍 (resolveUrl)
- mp3 파일로는 오프라인 재생 불가

### 3.6 UI 원칙

- UI 절대 안 바꿈 (JS 로직만)
- 기존 사용자 경험 유지
- 새 기능은 추가 버튼으로 (기존 버튼 변경 X)

### 3.7 개발 원칙

- 한 번에 하나씩 (백업 → 수정 → 검증)
- LF 유지 (.gitattributes)
- Python 스크립트로 수정 (PowerShell 이스케이프 회피)
- git push만 (wrangler pages deploy 안 씀 — 자동 배포)

---

## 4. 제조 프로세스 (모든 타입 공통)

1. 창작자가 파일 업로드 (mp3, mp4, pdf...)
2. 클라이언트 → 로컬 에이전트:
   a. 원본 디코딩 (PCM, 프레임, 페이지)
   b. 세그먼트 분할 (1초 단위)
   c. AES-256-GCM 암호화 (스파인)
   d. 타임토큰 체인 생성
   e. .tl3/.tl4/.tld 조립
3. 로컬 에이전트 → 서버:
   a. 메타 + 토큰체인 등록
   b. 저작권자 정보 등록
4. 서버: D1 저장 + 검증

## 5. 사용(재생/열람) 프로세스

1. 클라이언트: /api/v1/tl3/segment/:id?n=1&session_id=xxx
2. 서버:
   a. TL 잔액 검증
   b. 세션 상태 검증
   c. T_{n-1} 확인
   d. 창작자 PC에서 세그먼트 fetch (Range)
   e. AES-256-GCM 복호화
   f. T_n 갱신 + D1 저장
   g. TL 차감
   h. 평문 반환
3. 클라이언트: 이어붙임 (재생/표시)
4. (선택) 확장 모듈 호출 (lp.js 등)

---

## 6. 특허 출원

10-2025-0167813

> "시간 충전형 디지털 파일 기반 창작자 수익 창출, AI 예측 및 대출 연계 통합 시스템"

핵심 청구항:
1. 시간 기반 사용권 (TL)
2. 지속적 실체화 검증 (세그먼트별 서버 검증)
3. 스파인 기능 (순방향만 매끄러움)
4. 저작권자 암호학적 귀속
5. AI 예측 + 금융 연계

---

## 7. 참조

- TL3 파일 포맷: architecture/TL3_FORMAT.md
- 암호화 상세: architecture/CRYPTO.md
- 스파인 코드: architecture/SPINE_CODE.md
- 시스템 구조: architecture/SYSTEM.md

## 8. 공유와 기여의 언어 (원칙 7)

TimeLink는 "판매"와 "구매"의 언어를 쓰지 않는다.
우리는 "공유"와 "기여"의 언어를 쓴다.

- 파일을 "팔지" 않는다. "공유"한다.
- 돈을 "벌지" 않는다. "보상"받는다.
- 소비자는 "구매자"가 아니라 "기여자"다.
- 창작자는 "판매자"가 아니라 "공유자"다.

## 9. TL 무료 금지 — 디지털 자산 (원칙 8)

TL 파일은 절대 무료가 아니다.
TL이 없는 TL 파일은 무용지물이다.

- TL3, TLJ, TLD, TL4 — 모두 TL 필수
- "무료 재생"은 존재하지 않는다
- 무료는 "TL을 얻는 방법"에만 존재한다
  - 광고 시청 → TL
  - POC 기여 → TLP → TL
  - 이벤트 → TLA → TL

TL 파일 = 디지털 자산.
이 원칙이 TimeLink 보안성의 근본이다.

최초 작성: 2026-10-08
통합: 2026-10-11
프로젝트: TimeLink
