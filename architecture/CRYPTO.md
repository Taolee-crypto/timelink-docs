# 암호화 상세 (CRYPTO)

> TL3 v3의 키 유도, AES-GCM, 스파인 체인 상세.

---

## 1. 개요

TL3 v3는 **AES-256-GCM** + **SHA-256 체인** + **HKDF** 기반.

| 요소 | 알고리즘 | 용도 |
|------|---------|------|
| 세션 키 유도 | HKDF-SHA-256 | lic 유도 (서버) |
| 세그먼트 키 | SHA-256 체인 | K_n = SHA256(T_{n-1} ‖ lic ‖ "K"+n) |
| 암호화 | AES-256-GCM | 세그먼트별 암호화 |
| 태그 | GCM 128bit | 무결성 검증 |
| IV | 12바이트 | [4B 0][8B n (BE)] |

## 2. lic 유도 (서버만)

### HKDF-SHA-256

const master = await importMasterSecret(masterSecret);
const hkdfBits = await crypto.subtle.deriveBits(
  {
    name: 'HKDF',
    hash: 'SHA-256',
    salt: enc('tl3:lic:salt:' + shareId),
    info: enc('tl3:lic:' + shareId)
  },
  master,
  256   // 32바이트
);
const lic = new Uint8Array(hkdfBits);

**중요:** master_secret은 서버만 알고 있습니다.

### master_secret 설정

npx wrangler secret put TL3_MASTER_SECRET
# 32바이트 이상 랜덤 입력

## 3. 토큰 체인 (스파인)

### 초기 토큰 (T_0)

T_0 = SHA256(fid ‖ hash_mp3 ‖ salt)

- fid: 파일 ID (16바이트 랜덤)
- hash_mp3: 원본 mp3의 SHA256
- salt: 16바이트 랜덤

### 다음 토큰 (T_n)

T_n = SHA256(T_{n-1} ‖ fid ‖ str(n) ‖ SHA256(ct_n))

- T_{n-1}: 이전 토큰 (32바이트)
- fid: 파일 ID
- str(n): 세그먼트 번호 (10진수 문자열)
- SHA256(ct_n): 암호문의 SHA256

### 체인 예시

T_0 = SHA256(fid, hash_mp3, salt)
T_1 = SHA256(T_0, fid, "1", SHA256(ct_1))
T_2 = SHA256(T_1, fid, "2", SHA256(ct_2))
...
T_N = SHA256(T_{N-1}, fid, str(N), SHA256(ct_N))

**특성:**
- 역산 불가 (SHA-256 일방향)
- 순서 강제 (T_n은 T_{n-1} 필요)
- 무결성 검증 (변조 시 T_n 불일치)

## 4. 세그먼트 키 (K_n)

K_n = SHA256(T_{n-1} ‖ lic ‖ "K"+n)

- T_{n-1}: 이전 토큰
- lic: HKDF 유도 라이선스 (32바이트)
- "K"+n: 문자열 (예: "K1", "K42")

**결과:** 32바이트 AES-256 키

## 5. IV (초기화 벡터)

function ivFor(n) {
  const v = new Uint8Array(12);
  new DataView(v.buffer).setBigUint64(4, BigInt(n), false);
  return v;
}

**구조:**
[4B: 0x00000000] [8B: n (big-endian)]

예 (n=1):
00 00 00 00 00 00 00 00 00 00 00 01

예 (n=42):
00 00 00 00 00 00 00 00 00 00 00 2A

## 6. AES-256-GCM 암호화

### 암호화 (백엔드)

const ctMp3 = new Uint8Array(await crypto.subtle.encrypt(
  { name: 'AES-GCM', iv: ivFor(n), tagLength: 128 },
  kMp3,     // CryptoKey
  segBytes  // 평문
));

**결과:** ctMp3 = ciphertext + tag(16B)

### 복호화 (프론트/Player)

**브라우저 (Web Crypto):**

const plain = new Uint8Array(await crypto.subtle.decrypt(
  { name: 'AES-GCM', iv: ivFor(n), tagLength: 128 },
  key,
  ct   // ciphertext + tag 전체
));

**Node.js (Player Worker):**

const tag = ct.slice(ct.length - 16);
const ctOnly = ct.slice(0, ct.length - 16);
const decipher = crypto.createDecipheriv('aes-256-gcm', K_n, iv);
decipher.setAuthTag(tag);
const plain = Buffer.concat([
  decipher.update(ctOnly),
  decipher.final()
]);

**중요:** Node.js는 tag를 수동 분리해야 함.

## 7. 공격 시나리오와 방어

| 공격 | 방어 |
|------|------|
| 시간 역행 | T_n → T_{n-1} 역산 불가 |
| 세그먼트 건너뛰기 | 중간 토큰 필요 (T_0 → T_5 불가) |
| 세그먼트 순서 변경 | K_n이 달라져 복호화 실패 |
| 파일 복사 | lic 없으면 K_n 유도 불가 |
| mp3 파괴 | 서버 복호화 없이는 쓰레기 |
| 저작권자 위조 | AAD 검증 실패 |
| 중간자 공격 | TLS + GCM 인증 |

## 8. AAD (Additional Authenticated Data)

**저작권자 정보 AAD 포함 (계획):**

const aad = enc(JSON.stringify({
  creator_id: header.rights.creator_id,
  creator_name: header.rights.creator_name,
  license: header.rights.license
}));

const ct = await crypto.subtle.encrypt(
  { name: 'AES-GCM', iv: ivFor(n), additionalData: aad, tagLength: 128 },
  key,
  segBytes
);

→ 저작권자 정보 변조 시 복호화 실패.

**⚠️ 현재 구현에서는 AAD 미포함 (v1.1.2). 향후 추가 예정.**

---

## 9. 참조

- [TL3 파일 포맷](TL3_FORMAT.md)
- [스파인 코드](SPINE_CODE.md)
- [핵심 원칙](../CORE_PRINCIPLES.md)

---

**작성: 2026-10-11**  
**프로젝트: TimeLink**
