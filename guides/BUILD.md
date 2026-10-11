# 빌드 가이드 (BUILD)

> Player 앱 로컬 빌드 + GitHub Actions.

---

## 1. 사전 요구사항

| 도구 | 버전 | 용도 |
|------|------|------|
| Node.js | 20+ | 빌드 환경 |
| npm | 10+ | 패키지 관리 |
| Git | 최신 | 버전 관리 |

## 2. 로컬 빌드

### 2.1 의존성 설치

cd C:\Users\win11\Desktop\timelink-app
npm install

### 2.2 개발 모드 (빠른 테스트)

npm start

→ Electron 앱 실행 (src/main.js 직접 사용)

### 2.3 프로덕션 빌드

**Windows:**
npm run build:win

**macOS:**
npm run build:mac

**Linux:**
npm run build:linux

**결과:**
dist/
├── win-unpacked/
│   ├── TimeLink Player.exe
│   └── resources/app.asar
├── TimeLink-Player-Setup-1.1.2.exe
└── ...

## 3. package.json 스크립트

{
  "scripts": {
    "start": "electron .",
    "build:win": "electron-builder --win",
    "build:mac": "electron-builder --mac",
    "build:linux": "electron-builder --linux"
  }
}

## 4. electron-builder 설정

package.json의 build 키:

"build": {
  "appId": "digital.timelink.player",
  "productName": "TimeLink Player",
  "fileAssociations": [
    { "ext": "tl3", "name": "TimeLink Music" },
    { "ext": "tlj", "name": "TimeLink Digital Asset" }
  ],
  "win": { "target": "nsis" },
  "mac": { "target": "dmg" },
  "linux": { "target": "AppImage" }
}

## 5. GitHub Actions 자동 빌드

### 5.1 트리거

- **v* 태그 push** — 자동 빌드
- **workflow_dispatch** — 수동 트리거

### 5.2 build.yml (요약)

name: Build TimeLink Player

on:
  push:
    tags: ["v*"]
  workflow_dispatch:

jobs:
  build-win:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm install
      - run: npm run build:win
      - uses: softprops/action-gh-release@v2
        with:
          files: dist/*.exe

  build-mac:
    runs-on: macos-latest
    # ...

  build-linux:
    runs-on: ubuntu-latest
    # ...

### 5.3 배포 절차

cd C:\Users\win11\Desktop\timelink-app

# 1. 버전 업
# package.json: "version": "1.1.3"

# 2. 커밋
git add -A
git commit -m "feat: ..."

# 3. 태그
git tag v1.1.3

# 4. push
git push origin main
git push origin v1.1.3

### 5.4 결과 확인

**Actions:** https://github.com/Taolee-crypto/timelink-app/actions  
**Releases:** https://github.com/Taolee-crypto/timelink-app/releases

## 6. 빌드 결과물

### 6.1 Windows

- **TimeLink-Player-Setup-1.1.2.exe** — NSIS 인스톨러
- **win-unpacked/** — 포터블 실행 파일
- **.blockmap** — 자동 업데이트용

### 6.2 macOS

- **TimeLink-Player-1.1.2-arm64.dmg** — Apple Silicon
- **TimeLink-Player-1.1.2-x64.dmg** — Intel (설정 시)

### 6.3 Linux

- **TimeLink-Player-1.1.2.AppImage** — 범용

## 7. asar (앱 패키징)

### 7.1 asar란?

Electron 앱의 소스 코드를 묶는 아카이브.  
dist/win-unpacked/resources/app.asar

### 7.2 asar 내용 확인

npx @electron/asar list "dist\win-unpacked\resources\app.asar"

### 7.3 asar 추출

npx @electron/asar extract "dist\win-unpacked\resources\app.asar" "temp-app"

### 7.4 asar 재팩 (긴급 수정)

# 1. 추출
npx @electron/asar extract "dist\win-unpacked\resources\app.asar" "temp-app"

# 2. main.js 교체
Copy-Item "src\main.js" "temp-app\main.js" -Force

# 3. 재팩
npx @electron/asar pack "temp-app" "dist\win-unpacked\resources\app.asar"

# 4. 정리
Remove-Item "temp-app" -Recurse -Force

## 8. 트러블슈팅

### 빌드 실패: EBUSY

**원인:** 실행 중인 TimeLink Player 프로세스가 파일 잠금  
**해결:**
Get-Process | Where-Object { .ProcessName -like "*TimeLink*" } | Stop-Process -Force

### 빌드 실패: Cannot find module

**원인:** 의존성 미설치  
**해결:** npm install

### asar가 예전 코드

**원인:** src 수정 후 재빌드 안 함  
**해결:** npm run build:win 재실행

### GitHub Actions 실패

**원인:** 다수  
**해결:** Actions 로그 확인 (https://github.com/Taolee-crypto/timelink-app/actions)

---

## 9. 참조

- [핵심 원칙](../CORE_PRINCIPLES.md)
- [배포 가이드](DEPLOYMENT.md)
- [개발 환경](DEVELOPMENT.md)

---

**작성: 2026-10-11**  
**프로젝트: TimeLink**
