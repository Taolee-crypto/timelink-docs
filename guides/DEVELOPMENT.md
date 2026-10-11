# 개발 환경 (DEVELOPMENT)

> 로컬 개발 세팅, 규칙, 워크플로우.

---

## 1. 개발 환경 요구사항

| 도구 | 버전 | 필수 |
|------|------|------|
| Node.js | 20+ | ✅ |
| npm | 10+ | ✅ |
| Git | 최신 | ✅ |
| VS Code | 최신 | 권장 |
| PowerShell | 7+ | ✅ (Windows) |
| Python | 3.10+ | 선택 (스크립트) |

## 2. 저장소 클론

cd C:\Users\win11\Desktop

git clone https://github.com/Taolee-crypto/timelink-app.git
git clone https://github.com/Taolee-crypto/timelink-frontend.git
git clone https://github.com/Taolee-crypto/timelink-backend.git
git clone https://github.com/Taolee-crypto/timelink-docs.git

## 3. 각 레포 개발

### 3.1 앱 (timelink-app)

cd timelink-app
npm install
npm start   # 개발 모드

**주요 파일:**
- src/main.js — Electron 메인
- src/preload.js — IPC
- src/index.html — UI

### 3.2 프론트 (timelink-frontend)

순수 HTML/JS/CSS. 별도 빌드 없음.

cd timelink-frontend

**로컬 서버 (선택):**
npx serve public

**주요 파일:**
- public/tl3.js — TL3 클라이언트
- public/*.html — 페이지

### 3.3 백엔드 (timelink-backend)

cd timelink-backend
npm install

**로컬 개발:**
npm run dev

**배포:**
npx wrangler deploy

**주요 파일:**
- src/index.ts — 메인 Worker
- src/tl3v3.ts — TL3 빌더

## 4. 개발 규칙

### 4.1 파일 포맷 관련

- **변경 시 3자 동시 수정 + 검증**
  - 백엔드 (tl3v3.ts)
  - 프론트 (tl3.js)
  - Player (main.js)
- **first_offset은 계산하지 말고 9+hlen 사용**
- **엔디안은 big-endian (BE) 통일**
- **헤더 파서는 data[5..8] + slice(9)**

### 4.2 암호화 관련

- **lic은 서버만 유도** (HKDF)
- **T_n 체인은 파일에 내장** (스파인)
- **AES-GCM tag는 ciphertext 뒤 16바이트**
- **IV = [4B 0][8B n BE]**

### 4.3 UI 관련

- **UI 절대 안 바꿈** (JS 로직만)
- **새 기능은 추가 버튼으로**
- **기존 사용자 경험 유지**

### 4.4 백업/검증

- **수정 전 백업:** main.js.bak-YYYYMMDD-HHMMSS
- **한 번에 하나씩** (백업 → 수정 → 검증)
- **각 단계 검증 후 다음**

### 4.5 Git

- **커밋 메시지:** fix:, feat:, docs:, chore:
- **LF 유지** (.gitattributes)
- **push만** (자동 배포)

## 5. 자주 쓰는 명령

### 5.1 앱 (Player)

# 개발 실행
cd C:\Users\win11\Desktop\timelink-app
npm start

# 프로세스 정리
Get-Process | Where-Object { .ProcessName -like "*TimeLink*" } | Stop-Process -Force

# 빌드
npm run build:win

# 커밋 + 태그 + push
git add -A
git commit -m "feat: ..."
git tag v1.1.x
git push origin main
git push origin v1.1.x

### 5.2 프론트

cd C:\Users\win11\Desktop\timelink-frontend
git add public/
git commit -m "fix: ..."
git push origin main   # 30초 후 자동 배포

### 5.3 백엔드

cd C:\Users\win11\Desktop\timelink-backend
npx wrangler deploy

## 6. 개발 워크플로우 (일반)

### 6.1 버그 수정

1. **재현:** 버그 확인
2. **백업:** main.js.bak-* 등
3. **수정:** 코드 변경 (한 번에 하나)
4. **검증:** npm start로 테스트
5. **커밋:** git add + commit
6. **푸시:** git push
7. **빌드:** (필요시) git tag vX.Y.Z + push
8. **문서:** 세션 로그 (timelink-docs)

### 6.2 새 기능

1. **설계:** 어떤 원칙에 맞는지 (CORE_PRINCIPLES.md)
2. **검토:** 3자 동시 수정 필요한지
3. **구현:** 순차 진행
4. **검증:** 각 컴포넌트
5. **문서:** 아키텍처 변경 시 architecture/ 업데이트

### 6.3 세션 로그 작성

**매 세션 후:**

cd C:\Users\win11\Desktop\timelink-docs
notepad sessions\YYYY-MM-DD.md

**템플릿:**
# YYYY-MM-DD 작업 세션

## 요약
- 주요 성과 1
- 주요 성과 2

## 상세
### 1. ...
### 2. ...

## 커밋
- repo: commit_hash

## 다음 TODO
- [ ] ...

## 7. 백업 정책

### 7.1 코드 백업

- **수정 전:** main.js.bak-YYYYMMDD-HHMMSS
- **Git:** 모든 커밋이 백업
- **GitHub:** 원격 백업

### 7.2 백업 파일 정리

**월 1회:**

cd C:\Users\win11\Desktop\timelink-frontend\public
Get-ChildItem "*.bak-*" | Sort-Object LastWriteTime -Descending | Select-Object -Skip 3 | Remove-Item -Force

### 7.3 D1 백업

npx wrangler d1 export timelink-db --remote --output backup-YYYYMMDD.sql

## 8. 디버깅 팁

### 8.1 Player 문제

**개발자도구:**
- Ctrl+Shift+I (Player 창에서)

**Worker 로그:**
- npm start 실행 창에서 확인
- console.error 확인

### 8.2 프론트 문제

**F12 개발자도구:**
- Console: JS 에러
- Network: API 호출
- Application: localStorage, 캐시

### 8.3 백엔드 문제

npx wrangler tail   # 실시간 로그

---

## 9. 참조

- [핵심 원칙](../CORE_PRINCIPLES.md)
- [배포 가이드](DEPLOYMENT.md)
- [빌드 가이드](BUILD.md)
- [시스템 구조](../architecture/SYSTEM.md)

---

**작성: 2026-10-11**  
**프로젝트: TimeLink**
