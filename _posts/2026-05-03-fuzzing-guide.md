---
layout: post
title: "퍼징(Fuzzing) 완전 정복 — AFL++, Jackalope, libFuzzer 비교 분석"
date: 2026-05-03
category: BugBounty
author: yejunkim2000
tags: [퍼징, Fuzzing, AFL++, Jackalope, libFuzzer, 버그바운티, 취약점분석]
---

> **예상 읽기 시간:** 약 15분
> **관련 링크:** [AFL++](https://github.com/AFLplusplus/AFLplusplus) · [Jackalope](https://github.com/googleprojectzero/Jackalope) · [Google Fuzzing](https://github.com/google/fuzzing)

---

## 목차

1. [퍼징이란 무엇인가?](#1-퍼징이란-무엇인가)
2. [퍼징의 종류](#2-퍼징의-종류)
3. [퍼징의 동작 원리](#3-퍼징의-동작-원리)
4. [주요 퍼저 비교](#4-주요-퍼저-비교)
5. [AFL++ 상세](#5-afl-상세)
6. [Jackalope 상세](#6-jackalope-상세)
7. [libFuzzer & Google Fuzzing](#7-libfuzzer--google-fuzzing)
8. [퍼저 선택 가이드](#8-퍼저-선택-가이드)
9. [실전 팁 & 마무리](#9-실전-팁--마무리)

---

## 1. 퍼징이란 무엇인가?

퍼징(Fuzzing)은 프로그램에 **무작위 혹은 반자동으로 생성된 입력값**을 대량 주입해 크래시, 메모리 오류, 예외 동작 등의 버그를 찾아내는 자동화된 소프트웨어 테스트 기법이다.

> **역사:** 1988년 위스콘신대학의 Barton Miller 교수가 Unix 유틸리티들이 랜덤 입력에 얼마나 충돌하는지 실험한 것이 시초다. 당시 Unix 유틸리티의 **25~33%** 가 랜덤 입력으로 크래시됐다.

퍼징이 강력한 이유는 사람이 직접 작성하는 테스트 케이스로는 발견하기 어려운 **경계 조건(edge case)** 이나 예상치 못한 입력 조합을 자동으로 탐색하기 때문이다. 실제로 Google의 OSS-Fuzz 프로젝트는 수천 개의 오픈소스 프로젝트에서 **10,000개 이상의 버그**를 자동으로 발견했다.

### 퍼징이 찾는 버그 유형

| 유형 | 설명 | 예시 |
|------|------|------|
| **메모리 오류** | 메모리 손상 취약점 | 버퍼 오버플로, use-after-free, 힙 오버플로 |
| **미정의 동작** | C/C++ UB 범주의 버그 | 정수 오버플로, NULL 포인터 역참조, 범위 초과 접근 |
| **크래시 / DoS** | 프로그램 비정상 종료 | 무한 루프, 스택 오버플로, 분할 오류(segfault) |

---

## 2. 퍼징의 종류

### 입력 생성 방식에 따른 분류

**생성 기반 (Generation-based)**

입력 포맷의 문법·스펙을 정의하고 이를 기반으로 테스트케이스를 새로 생성한다. 구조화된 프로토콜(HTTP, TLS, DNS 등)에 효과적이다. 문법을 정의하는 초기 비용이 높지만, 유효한 입력 구조를 유지할 수 있다.

**변이 기반 (Mutation-based)**

기존의 유효한 입력(seed)을 조금씩 변형해 새 테스트케이스를 만든다. 현실적이고 구현이 간단해 가장 널리 쓰인다. AFL, AFL++, libFuzzer가 이 방식을 채택한다.

---

### 프로그램 분석 수준에 따른 분류

**블랙박스 (Blackbox)**

내부 구조를 전혀 모르는 채로 입력만 변형한다. 구현이 쉽지만 코드 커버리지가 낮다. 네트워크 프로토콜 퍼징(Boofuzz 등)에 주로 사용된다.

**그레이박스 (Greybox)**

코드 커버리지 정보를 수집해 새 경로를 탐색하는 입력을 우선시한다. AFL이 이 방식의 대표주자로, **현재 가장 널리 쓰이는 방식**이다. 빠른 실행 속도와 높은 커버리지를 동시에 달성한다.

**화이트박스 (Whitebox)**

소스코드 분석, 기호 실행(symbolic execution)으로 특정 분기 조건을 계산한다. 깊은 버그를 찾을 수 있지만 매우 느리다. KLEE, S2E, SAGE 등이 이 방식을 사용한다.

> **실무에서 가장 많이 쓰이는 방식:** **그레이박스 + 변이 기반**의 조합이다. AFL, AFL++, libFuzzer가 모두 이 방식을 채택한다. 커버리지 피드백으로 탐색 효율을 높이면서도 화이트박스처럼 무겁지 않아 대규모 자동화에 적합하다.

---

## 3. 퍼징의 동작 원리

커버리지 기반 그레이박스 퍼징(CGF)의 기본 루프는 다음과 같다.

```
1. Seed 준비
2. 입력 변이
3. 타겟 실행
4. 커버리지 측정
5. 크래시 분류
6. 큐 업데이트
→ 2번으로 반복
```

### 커버리지 계측 (Instrumentation)

퍼저는 어느 코드 경로가 실행됐는지 알기 위해 계측(instrumentation)을 삽입한다. AFL/AFL++은 컴파일 시점에 `afl-clang-fast`로 브랜치 커버리지를 기록하며, 바이너리 전용 모드에서는 QEMU나 Frida로 런타임 계측을 한다.

**계측 방식 비교:**

| 방식 | 설명 | 속도 | 소스 필요 |
|------|------|------|-----------|
| 컴파일 타임 계측 | 빌드 시 브랜치 카운터 삽입 | 매우 빠름 | O |
| QEMU 모드 | QEMU 에뮬레이션으로 런타임 계측 | 약 1/10 속도 | X |
| Frida 모드 | Frida 훅으로 런타임 계측 | QEMU보다 빠름 | X |
| 하드웨어 PT | Intel PT로 하드웨어 수준 트레이싱 | 거의 네이티브 | X |

### AFL++ 기본 사용 예시

```bash
# 소스코드가 있는 경우 — 컴파일 계측
CC=afl-clang-fast ./configure && make

# 퍼징 실행
afl-fuzz -i ./seeds/ -o ./findings/ -- ./target_binary @@

# @@는 AFL이 생성한 파일 경로로 자동 치환됨
```

---

## 4. 주요 퍼저 비교

| 퍼저 | 방식 | 타겟 | 계측 | 성숙도 |
|------|------|------|------|--------|
| **AFL++** | 그레이박스 | Linux 바이너리 | 컴파일 / QEMU / Frida | ***** |
| **Jackalope** | 그레이박스 | Windows / macOS / Linux | TinyInst (바이너리) | *** |
| **libFuzzer** | 그레이박스 | 라이브러리 함수 | LLVM SanitizerCoverage | ***** |
| **Honggfuzz** | 그레이박스 | Linux / macOS / Android | 컴파일 / 하드웨어 PT | **** |
| **Syzkaller** | 그레이박스 | OS 커널 시스콜 | 커버리지 커널 모듈 | ***** |
| **Boofuzz** | 블랙박스 | 네트워크 프로토콜 | 없음 | *** |

---

## 5. AFL++ 상세

**AFL++ (American Fuzzy Lop Plus Plus)** 는 Google의 AFL을 커뮤니티가 포크해 대폭 개선한 업계 표준 퍼저다.

- **개발사:** 커뮤니티 (AFLplusplus)
- **라이선스:** Apache 2.0
- **GitHub Stars:** 17k+
- **저장소:** [AFLplusplus/AFLplusplus](https://github.com/AFLplusplus/AFLplusplus)

### AFL++의 주요 개선점

**CMPLOG**
비교 연산자의 입력값을 추적해 매직 바이트 / 체크섬을 자동으로 우회한다.

**MOpt / 커스텀 뮤테이터**
변이 전략을 동적으로 최적화하거나 직접 구현할 수 있다.

**Frida 모드**
소스 없는 바이너리를 Frida로 런타임 계측한다.

**Persistent 모드**
fork() 오버헤드 없이 루프 안에서 직접 퍼징한다. (속도 수백배↑)

**병렬 퍼징**
Master/Slave 구조로 멀티코어를 활용한다.

### Persistent 모드 하네스 예시

```c
#include <stdint.h>
#include <stddef.h>

// AFL++ persistent mode entry point
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    // 타겟 함수 직접 호출
    parse_input(data, size);
    return 0;  // 0이면 정상, -1이면 이 케이스를 큐에서 제외
}
```

```bash
# AddressSanitizer와 함께 빌드
afl-clang-fast -fsanitize=address -o fuzz_target harness.c target.c

# 병렬 퍼징 (Master 1개 + Slave 3개)
afl-fuzz -M master -i seeds/ -o out/ -- ./fuzz_target @@
afl-fuzz -S slave1 -i seeds/ -o out/ -- ./fuzz_target @@
afl-fuzz -S slave2 -i seeds/ -o out/ -- ./fuzz_target @@
afl-fuzz -S slave3 -i seeds/ -o out/ -- ./fuzz_target @@
```

### 장단점

**장점**
- 뛰어난 커버리지 탐색 효율
- 다양한 계측 모드 (소스/QEMU/Frida)
- 광범위한 커뮤니티와 문서
- libFuzzer 하네스와 호환
- 멀티코어 병렬 퍼징

**단점**
- Windows 미지원 (WSL 필요)
- QEMU 모드는 속도 10x 저하
- 초기 설정 학습 곡선 존재

---

## 6. Jackalope 상세

**Jackalope**는 Google Project Zero가 개발한 바이너리 전용 크로스플랫폼 퍼저다.

- **개발사:** Google Project Zero
- **라이선스:** Apache 2.0
- **지원 OS:** Windows / macOS / Linux
- **계측 도구:** TinyInst
- **저장소:** [googleprojectzero/Jackalope](https://github.com/googleprojectzero/Jackalope)

Jackalope의 핵심은 **TinyInst**라는 자체 동적 계측 라이브러리다. TinyInst는 DynamoRIO나 Pin보다 훨씬 가볍고, 특히 **Windows와 macOS**에서 소스코드 없이도 높은 성능으로 커버리지를 수집할 수 있다.

### Jackalope의 특징

- **바이너리 전용:** 소스코드 불필요 — 클로즈드소스 타겟에 적합하다.
- **크로스플랫폼:** Windows DLL, macOS dylib, Linux .so 모두 지원한다.
- **AFL 변이 엔진:** 내부적으로 AFL 방식의 변이 알고리즘을 사용한다.
- **확장성:** 커스텀 뮤테이터, 샘플 드라이버를 구현할 수 있다.

### Jackalope 사용 예시 (Windows)

```powershell
# Windows에서 PDF 파서 퍼징
fuzzer.exe -in seeds\ -out findings\ -t 5000
           -delivery samplecode -target_module pdfparser.dll
           -target_method ParseDocument -nargs 2
           -iterations 10000 -persist -loop
```

> **언제 Jackalope를 선택할까?** Windows 바이너리나 macOS 바이너리를 소스코드 없이 퍼징해야 할 때. 예: 벤더가 제공하는 폰트 파서, PDF 리더, 드라이버 등의 클로즈드소스 바이너리.

### 장단점

**장점**
- Windows/macOS 네이티브 지원
- 소스코드 불필요
- TinyInst로 낮은 오버헤드
- Google Project Zero의 신뢰성

**단점**
- AFL++보다 커뮤니티가 작음
- 소스 계측 불가 (바이너리 전용)
- 문서가 상대적으로 부족

---

## 7. libFuzzer & Google Fuzzing

**libFuzzer**는 LLVM에 내장된 인프로세스(in-process) 커버리지 기반 퍼징 엔진이다. 별도 바이너리가 아닌 **컴파일러 플래그 하나**로 활성화된다.

- **문서:** [llvm.org/docs/LibFuzzer](https://llvm.org/docs/LibFuzzer.html)
- **Google Fuzzing 가이드:** [google/fuzzing](https://github.com/google/fuzzing)

### libFuzzer 빌드 및 실행

```bash
# AddressSanitizer + libFuzzer 함께 사용
clang -fsanitize=address,fuzzer target.c -o fuzz_target

# 실행 (코어 수, 최대 입력 크기 등 옵션)
./fuzz_target -jobs=4 -max_len=4096 corpus/

# Rust — cargo-fuzz 사용
cargo fuzz add fuzz_target_1
cargo fuzz run fuzz_target_1
```

### libFuzzer 하네스 구조

```c
// libFuzzer 엔트리포인트 — 항상 이 시그니처를 사용
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    MyParser parser;
    parser.parse(data, size);
    return 0;
}
```

### AFL++ ↔ libFuzzer 호환

libFuzzer 형식의 하네스(`LLVMFuzzerTestOneInput`)는 AFL++의 `-fsanitize=fuzzer-no-link` 빌드로도 바로 사용 가능하다. **하나의 하네스로 두 퍼저 모두 활용할 수 있어 효율적이다.**

```bash
# 동일한 하네스를 AFL++로 빌드
afl-clang-fast -fsanitize=address -fsanitize=fuzzer-no-link \
    harness.c target.c -o afl_fuzz_target

# libFuzzer로 빌드
clang -fsanitize=address,fuzzer \
    harness.c target.c -o libfuzz_target
```

### Google OSS-Fuzz

Google의 [OSS-Fuzz](https://google.github.io/oss-fuzz/)는 libFuzzer + AFL++ + Honggfuzz를 동시에 실행해 오픈소스 프로젝트를 24/7 퍼징한다. 발견된 버그는 90일 후 자동으로 공개된다.

---

## 8. 퍼저 선택 가이드

| 상황 | 추천 퍼저 | 이유 |
|------|-----------|------|
| 소스코드 있는 Linux C/C++ 라이브러리 | AFL++ + libFuzzer | 최고의 커버리지, OSS-Fuzz 호환 |
| Windows 클로즈드소스 바이너리 | Jackalope | TinyInst 기반 Windows 네이티브 지원 |
| macOS 바이너리 (dylib, 앱) | Jackalope | macOS 지원 퍼저 중 가장 성숙 |
| 소스 없는 Linux 바이너리 | AFL++ (QEMU/Frida 모드) | 바이너리 계측 모드 내장 |
| Linux 커널 / 시스콜 인터페이스 | Syzkaller | 시스콜 문법 명세 기반 생성 퍼징 |
| 네트워크 프로토콜 / IoT 장비 | Boofuzz | 프로토콜 모델 기반 블랙박스 퍼징 |
| Rust / Go 라이브러리 | libFuzzer (cargo-fuzz) | 언어 생태계 통합, 언어별 새니타이저 |

---

## 9. 실전 팁 & 마무리

### 효과적인 퍼징을 위한 체크리스트

**1. Sanitizer 함께 사용**

ASan, UBSan, MSan을 퍼저와 함께 빌드하면 크래시 없이 넘어가는 메모리 오류도 잡을 수 있다.

```bash
# AddressSanitizer + UndefinedBehaviorSanitizer 조합
afl-clang-fast -fsanitize=address,undefined -o target target.c
```

**2. 고품질 Seed 준비**

실제 프로그램이 처리하는 유효한 입력 파일(PDF, PNG, XML 등)을 seed로 사용하면 초기 커버리지가 크게 높아진다.

**3. 코퍼스 최소화**

```bash
# afl-cmin으로 중복 커버리지 seed 제거
afl-cmin -i ./corpus/ -o ./corpus_min/ -- ./target @@
```

**4. Persistent 모드 활용**

fork() 오버헤드를 없애 초당 실행 횟수를 10~100배 높일 수 있다.

**5. 크래시 트리아지**

```bash
# afl-triage로 유니크 크래시 분류
afl-triage -i ./findings/crashes/ -o ./unique/ -- ./target @@
```

### 버그바운티에서의 퍼징 전략

> 공개된 오픈소스 라이브러리를 대상으로 AFL++을 돌리는 것은 가장 접근하기 쉬운 시작점이다. GitHub의 취약점 히스토리를 보고 비슷한 파서 코드를 집중 퍼징하는 **타겟팅 전략**이 효과적이다.

1. 타겟 프로그램의 **입력 파서** 부분을 식별한다.
2. 관련 **CVE 히스토리**를 분석해 취약한 코드 패턴을 파악한다.
3. 해당 코드 경로를 집중적으로 실행하는 **Seed 코퍼스**를 구성한다.
4. ASan + AFL++으로 **장기 퍼징**을 돌린다 (최소 24시간 이상).
5. 크래시 발생 시 **익스플로잇 가능성**을 분석해 보고서를 작성한다.

---

## 참고 자료

- [AFLplusplus/AFLplusplus — GitHub](https://github.com/AFLplusplus/AFLplusplus)
- [googleprojectzero/Jackalope — GitHub](https://github.com/googleprojectzero/Jackalope)
- [google/fuzzing — Fuzzing 가이드 & 튜토리얼](https://github.com/google/fuzzing)
- [libFuzzer 공식 문서 — llvm.org](https://llvm.org/docs/LibFuzzer.html)
- [OSS-Fuzz — Google 오픈소스 퍼징 플랫폼](https://google.github.io/oss-fuzz/)
- [Honggfuzz — GitHub](https://github.com/google/honggfuzz)
- [Syzkaller — GitHub](https://github.com/google/syzkaller)
