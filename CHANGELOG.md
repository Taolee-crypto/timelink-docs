# Changelog

TimeLink 프로젝트 주요 변경 이력.

---

## [v1.1.2] — 2026-10-11

### Fixed
- **오프라인 .tl3 재생 복구** (핵심)
  - 헤더 파서: big-endian + slice(9)
  - Worker: AES-256-GCM + SHA256 체인 재작성
  - first_offset 대신 9+hl 사용 (18바이트 어긋남 해결)
- **백엔드 first_offset 안정화 루프** (tl3v3.ts)
- **프론트 tl3.js first_offset 무시** (9+hlen 사용)

### Added
- **GitHub Actions 자동 빌드** (v* 태그 push 시)
- **download.html GitHub API 자동 조회** (최신 릴리스)

### Changed
- version: 1.1.1 → 1.1.2
- build.yml → .github/workflows/ 이동

### Commits
- timelink-app: a59f2bb
- timelink-backend: 4b081c7d
- timelink-frontend: bc7715b

---

## [v1.1.1] — 2026-04-08

### Added
- 충전 버튼 항상 표시
- 충전 로직 수정

---

## [v1.1.0] — 2026-04-XX

### Added
- 플레이리스트
- 오디오/영상
- 전체화면
- TL 차감

---

## [v1.0.0] — 2026-03-19

### Added
- TimeLink Player 최초 출시
- .tl3 파일 재생
- TL 잔액 차감

---

## 문서 이력

### 2026-10-11
- **timelink-docs 레포 생성**
  - CORE_PRINCIPLES.md (통합 원칙)
  - architecture/ (TL3_FORMAT, CRYPTO, SPINE_CODE, SYSTEM)
  - guides/ (DEPLOYMENT, BUILD, DEVELOPMENT)
  - sessions/2026-10-11.md (세션 로그)

---

**프로젝트: TimeLink**  
**회사: 씨엠피 주식회사**
