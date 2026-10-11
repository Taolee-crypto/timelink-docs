# 스파인 코드 (Spine Code)

> TL3 파일에 내장된 타임토큰 체인 (T_0 ~ T_N).

---

## 1. 정의

**스파인 코드(Spine Code)**는 **TL3 파일에 내장된 타임토큰 체인(T_0 ~ T_N)** 이다.

파일의 "척추(spine)" 역할을 하며, 다음을 **파일 구조 자체로** 강제한다:

- 재생 순서 (T_0 → T_1 → ... → T_N)
- 파일 무결성 (각 토큰이 이전 토큰 해시 포함)
- 역방향/우회 방지

**스파인 코드는 파일에 있고, 플레이어는 그걸 순차 진행시키는 로직만 가진다.**

## 2. 3요소 구조

| 요소 | 위치 | 역할 |
|------|------|------|
| **스파인 코드 (T_0 ~ T_N)** | **파일 (.tl3)** | **순서/무결성 강제 (파일의 척추)** |
| 진행 로직 | 플레이어 (tl3.js / main.js) | 파일의 체인을 순차 진행 |
| 음원 코드 (lic) | 서버 | 라이선스 키 (TL 차감 후 제공) |

## 3. 체인 구조

T_0 = SHA256(fid ‖ hash_mp3 ‖ salt)
T_1 = SHA256(T_0 ‖ fid ‖ "1" ‖ SHA256(ct_1))
T_2 = SHA256(T_1 ‖ fid ‖ "2" ‖ SHA256(ct_2))
...
T_N = SHA256(T_{N-1} ‖ fid ‖ str(N) ‖ SHA256(ct_N))

**파일 헤더의 mp3_tokens:**
[
  "T0_hex_64자",
  "T1_hex_64자",
  "T2_hex_64자",
  ...
  "TN_hex_64자"
]

총 N+1개 (T_0 ~ T_N).

## 4. 검증 로직

### T_0 검증

const T0c = await sha256(fid, hashMp3, salt);   // 계산
const T0f = unhex(tokens[0]);                    // 파일에 저장된 값
if (hex(T0c) !== hex(T0f)) {
  throw new Error('T_0 검증 실패');
}

### T_n 검증 (n >= 1)

const hct = await sha256(ct);                    // 암호문 해시
const Tnc = await sha256(Tprev, fid, enc(String(n)), hct);
const Tnf = tokens[n] ? unhex(tokens[n]) : null;
if (Tnf && hex(Tnc) !== hex(Tnf)) {
  throw new Error('T_' + n + ' 검증 실패');
}

**의미:**
- 파일이 변조되면 → ct 변경 → SHA256(ct) 변경 → T_n 계산값 불일치
- 순서가 바뀌면 → Tprev 불일치 → T_n 계산 실패

## 5. 스파인 코드와 서버 검증

### 온라인 재생

서버가 매 세그먼트 요청마다:
1. TL 잔액 검증
2. 세션 상태 검증
3. **T_{n-1} 확인** (D1의 이전 토큰)
4. 세그먼트 fetch
5. 복호화
6. **T_n 갱신 + D1 저장**
7. TL 차감

→ **Server Spine**: 파일을 복사해도 서버 검증을 우회할 수 없음.

### 오프라인 재생

파일의 mp3_tokens 배열로 클라이언트가 자체 검증:
- T_0 ~ T_N이 모두 파일에 있음
- 순차적으로 검증 + 복호화
- 서버 없이도 무결성 보장

**보안 트레이드오프:**
- 온라인: 강력한 Server Spine (TL 회계 정확)
- 오프라인: 파일 자급자족 (TL 회계 우회 가능)

→ **lic 포함 시 오프라인 재생 가능** (현재 방식)

## 6. 방어 메커니즘

| 시도 | 결과 |
|------|------|
| 시간 역행 | T_n → T_{n-1} 역산 불가 (해시 일방향) |
| 건너뛰기 | T_0 → T_5 계산 불가 (중간 토큰 필요) |
| 재조립 | 세그먼트 순서 바꾸면 키 다름 → 복호화 실패 |
| 파일 복사 | 서버 없이는 lic 계산 불가 (오프라인은 lic 포함) |
| mp3 파괴 | 서버 복호화 없이는 쓰레기 바이트 |
| 저작권자 위조 | AAD 검증 실패 |

## 7. 구현 예시 (Player main.js)

const WORKER_CODE = 
const { workerData, parentPort } = require('worker_threads');
const fs = require('fs');
const crypto = require('crypto');

const sha256 = (...parts) => {
  const h = crypto.createHash('sha256');
  for (const p of parts) {
    if (Buffer.isBuffer(p)) h.update(p);
    else if (p instanceof Uint8Array) h.update(Buffer.from(p));
    else h.update(Buffer.from(String(p), 'utf8'));
  }
  return h.digest();
};

const unhex = (s) => Buffer.from(s, 'hex');

const ivFor = (n) => {
  const iv = Buffer.alloc(12);
  iv.writeBigUInt64BE(BigInt(n), 4);
  return iv;
};

try {
  const buf = fs.readFileSync(workerData.fp);
  const data = new Uint8Array(buf);
  const hl = ((data[5]<<24)|(data[6]<<16)|(data[7]<<8)|data[8])>>>0;
  const hdr = JSON.parse(Buffer.from(data.slice(9,9+hl)).toString('utf8'));

  const fid = unhex(hdr.fid);
  const salt = unhex(hdr.salt);
  const hashMp3 = unhex(hdr.hash_mp3);
  const lic = unhex(hdr.lic);
  const tokens = hdr.mp3_tokens;
  const N = hdr.mp3.N;
  const segLens = hdr.mp3_seg_lens;

  let off = 9 + hl;
  const parts = [];

  for (let n = 1; n <= N; n++) {
    const ctLen = segLens[n-1];
    const ct = data.slice(off, off + ctLen);
    off += ctLen;

    // Tprev
    const Tprev = (n === 1) ? sha256(fid, hashMp3, salt) : unhex(tokens[n-1]);

    // T_n 검증 (선택)
    // const hct = sha256(ct);
    // const Tnc = sha256(Tprev, fid, Buffer.from(String(n)), hct);
    // if (hex(Tnc) !== tokens[n]) throw new Error('T_' + n + ' 검증 실패');

    // K_n
    const K_n = sha256(Tprev, lic, Buffer.from('K' + n));

    // 복호화
    const iv = ivFor(n);
    const tag = ct.slice(ct.length - 16);
    const ctOnly = ct.slice(0, ct.length - 16);
    const decipher = crypto.createDecipheriv('aes-256-gcm', K_n, iv);
    decipher.setAuthTag(tag);
    const plain = Buffer.concat([decipher.update(ctOnly), decipher.final()]);

    parts.push(plain);
  }

  const mp3 = Buffer.concat(parts);
  parentPort.postMessage({ok:true, b64: mp3.toString('base64')});
} catch(e) {
  parentPort.postMessage({ok:false, error: e.message});
}
;

## 8. 성능 고려사항

- **검증 생략 옵션:** Player는 T_n 검증을 생략할 수 있음 (성능 vs 보안)
- **GCM 인증으로 대체:** AES-GCM 자체가 무결성을 검증하므로, T_n 검증이 이중 안전장치
- **오프라인:** T_n 검증 없이도 GCM 인증으로 변조 감지

## 9. 파일 검증 결과 예시

T0c (계산):  153ee3b8853eddf2f8ad1d9fa253ea5c...
tokens[0]:   153ee3b8853eddf2f8ad1d9fa253ea5c...
일치? True ✅

AES-GCM 복호화: 성공 ✅
plain 첫 8바이트: fffb540000000000 (MP3 프레임 싱크)

---

## 10. 참조

- [TL3 파일 포맷](TL3_FORMAT.md)
- [암호화](CRYPTO.md)
- [핵심 원칙](../CORE_PRINCIPLES.md)

---

**작성: 2026-10-11**  
**프로젝트: TimeLink**
