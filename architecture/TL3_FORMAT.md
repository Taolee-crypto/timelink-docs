# TL3 파일 포맷 (v3)

> TimeLink v3 컨테이너. 음원(TL3), 영상(TL4), 문서(TLD) 공통.

---

## 1. 개요

TL3 파일은 **TLNK 컨테이너**에 AES-256-GCM 암호문 세그먼트를 담은 형태입니다.

- **매직:** TLNK
- **버전:** 0x03
- **암호:** AES-256-GCM (tag 128bit)
- **KDF:** HKDF-SHA-256 (lic), SHA-256 체인 (T_n)
- **세그먼트:** 1초 단위 (음원/영상), 1페이지 (문서)

## 2. 파일 레이아웃

[0..3]   "TLNK" (magic 0x54 0x4C 0x4E 0x4B)
[4]      version = 0x03
[5..8]   headerLen (uint32, big-endian)
[9..]    header JSON (UTF-8)
[9+hlen] mp3 세그먼트 × N (ciphertext + tag)

### 각 세그먼트

[ ciphertext (segLen - 16) ] [ tag (16) ]

- ciphertext: AES-256-GCM 암호문
- tag: GCM 인증 태그 (16바이트, 마지막)

## 3. 헤더 스키마 (JSON)

{
  "v": 3,
  "aead": "AES-256-GCM",
  "kdf": "HKDF-SHA-256",
  "title": "곡 제목",
  "artist": "아티스트",
  "cid": "creator id",
  "name": "파일명",
  "dur_sec": 42,
  "fs": 0,
  "ch": 0,
  "mp3": {
    "N": 42,
    "seg_bytes": 0,
    "last_bytes": 39568,
    "first_offset": 3936
  },
  "lp": null,
  "hash_mp3": "...",
  "hash_lp": null,
  "fid": "...",
  "salt": "...",
  "lic_id": "k_2026_01",
  "lic": "...",
  "token_latest": "...",
  "share_id": "sh_...",
  "track_id": 43,
  "mp3_tokens": ["T0_hex", "T1_hex", ...],
  "mp3_seg_lens": [117328, 117328, ...],
  "ts": "2026-10-11T...",
  "rights": {
    "creator_id": 2,
    "license": "TL3-v3",
    "ownership": "creator"
  }
}

## 4. 세그먼트 구조

### mp3 세그먼트 (음원)

- 1초 = 44100 sample × 2ch × 2byte = 176400 byte (PCM16)
- 암호화 후: 176400 + 16 (tag) = 176416 byte
- mp3 프레임 방식이면 실제 길이는 mp3_seg_lens에 저장

### mp3_seg_lens 배열

- 세그먼트별 **ciphertext + tag 총 길이**
- 마지막 세그먼트는 짧을 수 있음 (last_bytes)

## 5. first_offset 문제와 해결

### 문제

기존 백엔드는 2-pass offset 계산:

const headerJson1 = enc(JSON.stringify(header));
const headerTotal1 = 4 + 1 + 4 + headerJson1.length;
header.mp3.first_offset = headerTotal1;

const headerJson2 = enc(JSON.stringify(header));
const headerTotal2 = 4 + 1 + 4 + headerJson2.length;
if (headerTotal2 !== headerTotal1) {
  header.mp3.first_offset += (headerTotal2 - headerTotal1);
}

const headerJsonFinal = enc(JSON.stringify(header));
// ⚠️ 여기서 headerTotalFinal !== headerTotal2일 수 있음!

**결과:** first_offset이 실제 payload 시작과 18바이트 어긋남.

### 해결 (백엔드, 2026-10-11)

**안정화 루프:**

let prevTotal = -1;
let safety = 0;
while (safety++ < 10) {
  const json = enc(JSON.stringify(header));
  const total = 4 + 1 + 4 + json.length;
  if (total === prevTotal) break;
  header.mp3.first_offset = total;
  prevTotal = total;
}

### 해결 (프론트/Player)

**first_offset 무시, 9 + hlen 사용:**

const payloadStart = 9 + hlen;
let off = payloadStart;

## 6. 파싱 알고리즘 (3자 통일)

### 6.1 헤더 파싱

const buf = fs.readFileSync(fp);
const data = new Uint8Array(buf);

// 매직
if (data[0]!==0x54||data[1]!==0x4C||data[2]!==0x4E||data[3]!==0x4B)
  throw new Error('Not TLNK');

// 버전
if (data[4] !== 0x03) throw new Error('Not v3');

// 헤더 길이 (big-endian)
const hlen = ((data[5]<<24)|(data[6]<<16)|(data[7]<<8)|data[8])>>>0;

// 헤더 JSON
const header = JSON.parse(Buffer.from(data.slice(9,9+hlen)).toString('utf8'));

### 6.2 세그먼트 순회

const mp3 = header.mp3;
const N = mp3.N;
const segLens = header.mp3_seg_lens;
const tokens = header.mp3_tokens;
const fid = unhex(header.fid);
const salt = unhex(header.salt);
const hashMp3 = unhex(header.hash_mp3);
const lic = unhex(header.lic);

let off = 9 + hlen;   // first_offset 무시!
const parts = [];

for (let n = 1; n <= N; n++) {
  // ciphertext + tag
  const ctLen = segLens[n-1];
  const ct = data.slice(off, off + ctLen);
  off += ctLen;

  // T_{n-1}
  const Tprev = (n === 1)
    ? sha256(fid, hashMp3, salt)   // T_0
    : unhex(tokens[n-1]);

  // K_n = SHA256(Tprev, lic, "K"+n)
  const K_n = sha256(Tprev, lic, Buffer.from('K' + n));

  // IV = [4B 0][8B n BE]
  const iv = Buffer.alloc(12);
  iv.writeBigUInt64BE(BigInt(n), 4);

  // AES-GCM decrypt
  const tag = ct.slice(ct.length - 16);
  const ctOnly = ct.slice(0, ct.length - 16);
  const decipher = crypto.createDecipheriv('aes-256-gcm', K_n, iv);
  decipher.setAuthTag(tag);
  const plain = Buffer.concat([decipher.update(ctOnly), decipher.final()]);

  parts.push(plain);
}

const mp3Bytes = Buffer.concat(parts);

---

## 7. 버전 이력

### v3 (2026-08~현재)

- TLNK 컨테이너
- AES-256-GCM 세그먼트 암호화
- SHA-256 토큰 체인 (스파인)
- first_offset 자동 계산 (2026-10-11 안정화)

### v1 (2026-03, 폐기)

- TLNK 01 버전
- XOR 스트림 암호 (보안 취약)
- 토큰 체인 없음
- **더 이상 지원 안 함**

---

## 8. 참조

- [암호화 상세](CRYPTO.md)
- [스파인 코드](SPINE_CODE.md)
- [핵심 원칙](../CORE_PRINCIPLES.md)

---

**작성: 2026-10-11**  
**프로젝트: TimeLink**
