# 배포 가이드 (DEPLOYMENT)

> 백엔드/프론트/Player 배포 절차.

---

## 1. 개요

| 컴포넌트 | 배포 방식 | 명령 |
|---------|----------|------|
| 백엔드 | wrangler deploy | npx wrangler deploy |
| 프론트 | Cloudflare Pages | git push (자동) 또는 wrangler pages deploy |
| Player | GitHub Actions | git tag + push |

---

## 2. 백엔드 배포 (Cloudflare Worker)

### 2.1 사전 준비

cd C:\Users\win11\Desktop\timelink-backend
npm install

### 2.2 시크릿 설정 (최초 1회)

npx wrangler secret put JWT_SECRET
npx wrangler secret put TL3_MASTER_SECRET

### 2.3 배포

npx wrangler deploy

**출력 예시:**
Total Upload: 948.40 KiB / gzip: 162.00 KiB
Uploaded timelink-backend (4.53 sec)
Deployed timelink-backend triggers (1.34 sec)
  https://timelink-backend.mununglee.workers.dev
Current Version ID: 4b081c7d-9520-4faf-9d4a-80d61068e6ab

### 2.4 검증

curl.exe -s -o NUL -w "HTTP %{http_code}
" "https://api.timelink.digital/api/shares"

**기대:** HTTP 200

---

## 3. 프론트 배포 (Cloudflare Pages)

### 3.1 프로젝트 정보

- 프로젝트: timelink-frontend
- 계정: 68161bb226f69cb03ab342b036562f79
- URL: https://www.timelink.digital
- 미리보기: https://{hash}.timelink-frontend.pages.dev

### 3.2 방법 A: 자동 배포 (권장)

git push만 하면 Cloudflare Pages가 자동 감지 → 배포.

cd C:\Users\win11\Desktop\timelink-frontend
git add public/
git commit -m "fix: ..."
git push origin main

**30초~2분 후 자동 배포.**

### 3.3 방법 B: 수동 배포 (wrangler)

cd C:\Users\win11\Desktop\timelink-frontend
npx wrangler pages deploy public --project-name=timelink-frontend

**출력 예시:**
Success! Uploaded 107 files (171 already uploaded) (2.27 sec)
Deploying...
Deployment complete! Take a peek over at
  https://a446a15e.timelink-frontend.pages.dev

### 3.4 검증

curl.exe -s -o NUL -w "HTTP %{http_code}
" "https://www.timelink.digital/"

**기대:** HTTP 200

---

## 4. Player 배포 (GitHub Actions)

### 4.1 자동 빌드 활성화 (완료)

- .github/workflows/build.yml 배치됨
- v* 태그 push 시 자동 실행
- Releases에 exe/dmg/AppImage 자동 첨부

### 4.2 배포 절차

cd C:\Users\win11\Desktop\timelink-app

# 1. package.json 버전 업
# version: 1.1.3

# 2. 커밋
git add -A
git commit -m "feat: ..."

# 3. 태그
git tag v1.1.3

# 4. push (커밋 + 태그)
git push origin main
git push origin v1.1.3

### 4.3 GitHub Actions 결과

**URL:** https://github.com/Taolee-crypto/timelink-app/actions

**3~5분 후:**
- build-win ✅
- build-mac ✅
- build-linux ⚠️ (선택)

### 4.4 Releases 확인

**URL:** https://github.com/Taolee-crypto/timelink-app/releases

**Assets 예시 (v1.1.2):**
- TimeLink-Player-Setup-1.1.2.exe (76MB)
- TimeLink-Player-1.1.2-arm64.dmg (93MB)
- TimeLink-Player-1.1.2.AppImage (104MB)

---

## 5. 로컬 에이전트 (창작자 PC)

### 5.1 실행

cd C:\Users\win11\Desktop\timelink-frontend\local-agent
npm start

### 5.2 PM2로 관리

pm2 start local-agent/server.js --name timelink-agent
pm2 save

### 5.3 상태 확인

pm2 list
curl.exe -s "http://127.0.0.1:8787/health"

### 5.4 Cloudflare Tunnel

- 터널: agent.timelink.digital → 127.0.0.1:8787
- 설정: Cloudflare Zero Trust

---

## 6. D1 데이터베이스

### 6.1 마이그레이션 적용

cd C:\Users\win11\Desktop\timelink-backend
npx wrangler d1 migrations apply timelink-db

### 6.2 원격 쿼리

npx wrangler d1 execute timelink-db --remote --command="SELECT name FROM sqlite_master WHERE type='table'"

### 6.3 주요 테이블

- tl3_releases — 릴리스 메타
- tl3_segments — 세그먼트 offset/length
- tl3_tokens — 토큰 체인
- tl3_stream_sessions — 스트리밍 세션

---

## 7. 배포 체크리스트

### 백엔드 배포 시

- [ ] npm install
- [ ] npx wrangler deploy
- [ ] Version ID 확인
- [ ] curl 검증 (api.timelink.digital)

### 프론트 배포 시

- [ ] git status (의도한 파일만)
- [ ] git add + commit
- [ ] git push
- [ ] 30초 후 배포 URL 확인
- [ ] 브라우저에서 실제 반영 확인

### Player 배포 시

- [ ] package.json 버전 업
- [ ] git commit
- [ ] git tag vX.Y.Z
- [ ] git push origin main + 태그
- [ ] GitHub Actions 성공 확인
- [ ] Releases에서 exe 다운로드 테스트

---

## 8. 트러블슈팅

### 프론트 배포 안 됨

1. Cloudflare Pages 대시보드 → 최근 배포 확인
2. GitHub 연동 여부 확인
3. 수동: npx wrangler pages deploy public --project-name=timelink-frontend

### Player 빌드 실패

1. GitHub Actions 로그 확인
2. 로컬에서 npm run build:win 테스트
3. 빌드.yml 확인 (.github/workflows/)

### 백엔드 오류

1. npx wrangler tail (실시간 로그)
2. D1 쿼리 확인
3. 시크릿 확인 (npx wrangler secret list)

---

## 9. 참조

- [핵심 원칙](../CORE_PRINCIPLES.md)
- [빌드 가이드](BUILD.md)
- [개발 환경](DEVELOPMENT.md)

---

**작성: 2026-10-11**  
**프로젝트: TimeLink**
