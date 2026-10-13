arkdown

\# TimeLink IA (Information Architecture) v2



최종 수정: 2026-10-13



\## 철학



\- TimeLink = \*\*디지털 파일 자산화 플랫폼\*\*

\- TL 파일 = \*\*디지털 자산\*\* (무료 없음)

\- TL이 없는 TL 파일 = \*\*무용지물\*\*

\- 모든 회원 = 크리에이터 (AI 시대)

\- 역할은 겹친다 (사장 + DJ + 청취자)

\- SharePlace = 자산 거래의 중심

\- TL3 = 시간토큰(TL) 없으면 재생 불가

\- 광고 시청 = TL 획득 수단 (하루 20회: 오전 10 / 오후 10)

\- TL은 절대 무료가 아니다

\- DJ 센터 = DJ 대리점

\- 마이카페 = 카페 방송국 + 마이카 라디오

\- index.html = 랜딩(비로그인) + 홈(로그인)



\## 언어 원칙



\- 판매 X → \*\*공유\*\* O

\- 구매 X → \*\*기여\*\* O

\- 돈을 번다 X → \*\*보상받는다\*\* O

\- 자세한 내용은 `LEXICON.md` 참조



\## 전체 IA 구조

\[로그인] → \[메인 홈]

↓

┌─────────┼─────────┬─────────┬─────────┬─────────┐

│ │ │ │ │ │

DIGITAL EARN CONSUME MINING 지갑 마이카페 DJ센터

ASSET



text



\## 화면별 상세



\### 1. 로그인 (`login.html`)



\- 이메일/비밀번호

\- Google 로그인

\- TON 지갑 로그인

\- 회원가입



\### 2. 메인 홈 (`index.html`)



\- Hero: "당신의 파일은 아직 자산이 아닙니다"

\- 실시간 티커 (보상 흐름)

\- 지금 인기 있는 자산 (카드)

\- 작동 방식 (4단계)

\- 창작자 되기 CTA

\- 우측: 롤 스타일 보상 피드

\- 하단: 미니 플레이어



\### 3. DIGITAL ASSET



\#### 3-1. 자산화 (`assetize.html`)



\- 파일 선택 (MP3, JPEG, PDF, MP4)

\- 자동 TL 변환

\- 최근 자산화 목록



\#### 3-2. 내 자산 (`my-assets.html`)



\- 보유 자산 목록

\- 각 자산별 보상/사용 수

\- 자산 관리



\#### 3-3. SharePlace (`shareplace.html`)



\- 전체/최신/인기/추천/장르별

\- 카드 그리드 (카드 = 버튼)

\- 즉시 재생

\- Tip TL



\### 4. EARN (보상) (`earn.html`)



\- 이번 주 보상

\- 보상 추이 그래프

\- 창작권 인증 상태

\- 최근 보상

\- 환전하기



\### 5. CONSUME (사용) (`consume.html`)



\- 음악 (TL3) O

\- 이미지 (TLJ) X 준비 중

\- 문서 (TLD) △ 부분 지원

\- 영상 (TL4) X 준비 중



\### 6. MINING (채굴) (`mining.html`)



\- 기여도 점수

\- 오늘 채굴된 TLC

\- 기여 내역 (콘텐츠/추천/큐레이션/커뮤니티)

\- TON 지갑 연결



\### 7. 지갑 (`wallet.html`)



\- 내 TL

\- 내 TLC

\- 받은 TL / 채굴 TLC 토글

\- 거래 내역

\- 환전하기 (창작권 인증 시)



\### 8. 마이카페 (`mycafe.html`)



\- 카페 방송국 (상업용, 월 50,000 TL)

\- 마이카 라디오 (개인용, TL 소모만)

\- 두 공간의 차이 명확히



\### 9. DJ 센터 (`dj-center.html`)



\- DJ 대리점

\- 라이브 방송

\- 편성표

\- 청취자 통계



\## 사이드바 구조

\[상단 토글: 듣기 | 만들기]



━━━━━━━━━━━━━━━━━━━━━━━━━━━

듣기 모드

━━━━━━━━━━━━━━━━━━━━━━━━━━━

홈

SharePlace

음악

차트

지갑

채굴



━━━━━━━━━━━━━━━━━━━━━━━━━━━

만들기 모드

━━━━━━━━━━━━━━━━━━━━━━━━━━━

대시보드

자산화

내 자산

보상

PoC 채굴

창작권 관리



━━━━━━━━━━━━━━━━━━━━━━━━━━━

\[공간]

━━━━━━━━━━━━━━━━━━━━━━━━━━━

마이카페

DJ 센터



━━━━━━━━━━━━━━━━━━━━━━━━━━━

\[내 TL 잔액]

12,450 TL

\[충전]

\[마이]



text



\## URL 매핑



| URL | 파일 | 상태 |

|-----|------|------|

| / | index.html | O |

| /login | login.html | O |

| /assetize | assetize.html | 신규 |

| /my-assets | my-assets.html | 신규 |

| /shareplace | shareplace.html | O |

| /earn | earn.html | 신규 |

| /consume | consume.html | 신규 |

| /mining | mining.html | 신규 |

| /wallet | wallet.html | O |

| /mycafe | mycafe.html | 신규 |

| /dj-center | dj-center.html | O |

| /radio | radio.html | O |

| /track | track.html | O |

| /ads | ads.html | O |



\## 파일 타입 상태



| 타입 | 대상 | 상태 |

|------|------|------|

| TL3 | 음악 | O 작동 |

| TLJ | 이미지 | X 준비 중 |

| TLD | 문서/악보 | △ 부분 지원 |

| TL4 | 영상 | X 준비 중 |



\## 경제 시스템



\### TL (사용권)

소비자 → TL 소비 → 콘텐츠 사용

↓

7:3 분배

↓

창작자 70% / 플랫폼 30%

↓

창작자: 즉시 TL 적립

↓

창작권 증명 시 → 환전 가능



text



\### TLC (기여 보상)

기여 (POC) → 채굴 → TLC → TON 지갑



text



\## 향후 작업



\- assetize.html, my-assets.html, earn.html, consume.html, mining.html, mycafe.html 신규 생성

\- sidebar.js 전면 재작성 (2모드)

\- \_redirects 생성

\- /api/me/roles 엔드포인트

\- mp3 표현 전면 제거 (원칙 8)

\- "공유/기여/보상" 언어 적용



\---



최초 작성: 2026-10-05

v2 재작성: 2026-10-13

프로젝트: TimeLink

