---
layout: post
title: "AFL 핵심 메커니즘 완전 분석 — Instrumentation, Coverage, Fork Server, Mutation"
date: 2026-05-04
category: BugBounty
author: yejunkim2000
tags: [AFL, Fuzzing, 퍼징, 취약점분석, BugBounty, 보안연구]
---

> **대상:** AFL (American Fuzzy Lop) 원본 — [google/AFL](https://github.com/google/AFL)
> **분석 파일:** `afl-gcc.c`, `afl-as.c`, `afl-as.h`, `afl-fuzz.c`, `docs/technical_details.txt`
> **예상 읽기 시간:** 약 20분

---

## 목차

1. [AFL이란 무엇인가 — 설계 철학](#1-afl이란-무엇인가--설계-철학)
2. [Instrumentation — 계측 도구가 심어지는 과정](#2-instrumentation--계측-도구가-심어지는-과정)
3. [Coverage — 커버리지를 측정하는 방법](#3-coverage--커버리지를-측정하는-방법)
4. [Fork Server — 왜 빠른가](#4-fork-server--왜-빠른가)
5. [Mutation — 입력을 어떻게 변이시키나](#5-mutation--입력을-어떻게-변이시키나)
6. [전체 흐름 종합](#6-전체-흐름-종합)

---

## 1. AFL이란 무엇인가 — 설계 철학

AFL의 technical_details.txt는 첫 문장부터 인상적이다.

> "American Fuzzy Lop does its best not to focus on any singular principle of operation... The tool can be thought of as a collection of hacks that have been tested in practice."

즉, AFL은 특정 이론의 증명이 아니라 **실제로 효과가 있음이 검증된 기법들의 조합**이다. 설계 원칙은 단 세 가지다: **속도(speed), 신뢰성(reliability), 사용 편의성(ease of use).**

AFL의 핵심 알고리즘을 한 줄로 요약하면 다음과 같다.

```
변이된 입력 → 타겟 실행 → 새 상태 전이 발견 → 큐에 추가 → 반복
```

공식 문서의 알고리즘 요약은 아래와 같다.

1. 사용자가 제공한 초기 테스트케이스를 큐에 로드
2. 큐에서 다음 입력 파일을 꺼냄
3. 측정된 동작을 바꾸지 않는 선에서 파일을 최소 크기로 트리밍
4. 다양한 변이 전략으로 파일을 반복 변이
5. 변이 결과가 새로운 상태 전이를 만들면 큐에 추가
6. 2번으로 돌아감

---

## 2. Instrumentation — 계측 도구가 심어지는 과정

AFL의 계측은 "LLVM 패스"가 아니다. 원본 AFL은 **어셈블러(assembler) 단계에서 계측을 삽입**한다. 이것이 AFL++과의 핵심 차이다.

### 2-1. 컴파일 파이프라인 전체 흐름

```
소스코드 (.c)
    ↓
[afl-gcc / afl-clang]  ← 실제 gcc/clang을 래핑한 wrapper
    ↓
전처리 (Preprocessor) → .i 파일
    ↓
컴파일 (Compiler) → .s 파일 (어셈블리)
    ↓
[afl-as]  ← ★ 이 단계에서 계측 코드가 삽입됨
    ↓
어셈블 (Assembler) → .o 파일 (오브젝트)
    ↓
링크 (Linker) → 최종 바이너리
```

### 2-2. afl-gcc의 역할 — Wrapper

`afl-gcc.c`를 보면, afl-gcc는 gcc 자체가 아니라 **gcc를 호출하는 래퍼(wrapper)** 다. 핵심은 `-B` 플래그다.

```c
// afl-gcc.c 중 핵심 부분
static void edit_params(u32 argc, char** argv) {

    // as_path에 afl-as가 있는 경로를 설정
    cc_params[cc_par_cnt++] = "-B";
    cc_params[cc_par_cnt++] = as_path;
    // ...
}
```

`-B` 플래그는 gcc에게 "어셈블러(as)를 이 경로에서 찾아라"고 지시한다. 즉, 시스템의 진짜 `as` 대신 **AFL의 `afl-as`를 어셈블러로 사용**하게 만드는 것이다.

컴파일 과정을 풀어쓰면 아래와 같다.

```bash
# 사용자가 입력하는 것
CC=afl-gcc ./configure && make

# 실제로 일어나는 것
afl-gcc (= gcc -B /path/to/afl/) source.c
    → gcc가 source.s (어셈블리) 생성
    → afl-as가 source.s를 가로채서 계측 코드 삽입
    → 계측된 source.o 생성
```

### 2-3. afl-as의 역할 — 어셈블리에 계측 코드 삽입

`afl-as.c`는 진짜 어셈블러(`as`)를 호출하기 전에 `.s` 파일을 **파싱해서 계측 코드를 삽입**한다.

삽입 타이밍은 **기본 블록(Basic Block)의 시작 지점**이다. 기본 블록이란 분기 없이 순차 실행되는 명령어들의 묶음이다.

```
기본 블록 예시:

  [블록 A]        [블록 B]        [블록 C]
  mov eax, 1  →  cmp eax, 0  →  je [블록 D]
  add ebx, 2     sub ecx, 1     mov edx, 5
                                jmp [블록 E]
```

AFL은 어셈블리 파일을 읽으면서 `jmp`, `je`, `call` 같은 분기 명령어 이후 위치를 감지하고, 그 직후(= 새 기본 블록 시작)에 계측 코드를 삽입한다.

`afl-as.h`에 삽입되는 실제 x86 어셈블리 코드가 정의되어 있다. 의사 코드로 표현하면 아래와 같다.

```asm
; 각 기본 블록 시작 지점에 삽입되는 코드
; cur_location은 컴파일 타임에 랜덤 생성된 이 블록의 ID

  mov rcx, <cur_location>          ; 현재 블록 ID를 레지스터에 로드
  call __afl_maybe_log             ; 커버리지 기록 함수 호출
```

`__afl_maybe_log` 함수가 실제 비트맵을 업데이트한다. 이 함수 역시 `afl-as.h`에 인라인 어셈블리로 정의되어 있으며, 핵심 로직은 아래와 같다.

```asm
__afl_maybe_log:
  ; shared_mem[cur_location XOR prev_location]++
  mov rdx, rcx               ; cur_location
  xor rdx, [prev_location]   ; XOR prev_location
  inc byte [shm_base + rdx]  ; 비트맵 카운터 증가

  ; prev_location = cur_location >> 1
  shr rcx, 1
  mov [prev_location], rcx
  ret
```

### 2-4. 왜 어셈블러 단계인가?

컴파일러마다 어셈블리 출력 형식이 조금씩 다르지만, **어셈블리 파일(.s)은 텍스트 형식**이라 파싱이 상대적으로 쉽다. 또한 gcc, clang, 심지어 gcj(Java!) 등 어떤 컴파일러로 빌드하든 어셈블리를 거치기 때문에, afl-as 하나만 교체하면 **모든 컴파일러에 투명하게 계측**을 적용할 수 있다.

---

## 3. Coverage — 커버리지를 측정하는 방법

### 3-1. 공유 메모리 비트맵 (Shared Memory Bitmap)

AFL의 커버리지 측정의 핵심은 **64KB 크기의 공유 메모리 배열**이다.

```c
// config.h
#define MAP_SIZE_POW2   16
#define MAP_SIZE        (1 << MAP_SIZE_POW2)  // = 65536 = 64KB
```

이 배열은 `afl-fuzz`(퍼저 프로세스)와 타겟 바이너리(자식 프로세스)가 `shmget()` / `shmat()`으로 **공유**한다. 타겟이 실행되는 동안 계측 코드가 이 배열을 직접 업데이트하고, 실행이 끝나면 퍼저가 배열을 읽어 분석한다.

```
[afl-fuzz 프로세스]          [타겟 바이너리 프로세스]
      |                              |
      |←── 공유 메모리 (64KB) ───→|
      |                              |
      |          실행 중에           |
      |                    bitmap[A^B]++
      |                    bitmap[B^C]++
      |                    bitmap[C^D]++
      |                              |
      |← 실행 종료 후 bitmap 읽기 ←|
```

### 3-2. 엣지 ID 계산 — XOR 공식의 의미

비트맵 인덱스를 계산하는 공식은 아래와 같다.

```
bitmap[cur_location XOR prev_location]++
prev_location = cur_location >> 1
```

여기서 `cur_location`은 컴파일 타임에 각 기본 블록에 **랜덤하게 배정된 16비트 정수**다.

**XOR을 쓰는 이유:** A→B 전이와 B→A 전이를 구별해야 하는데, 단순히 `A + B`나 `A * B`는 교환법칙이 성립해 구분이 불가능하다. XOR도 교환법칙이 성립하지만(`A^B == B^A`), `prev_location`을 `>> 1` 우측 시프트함으로써 방향성을 보존한다.

**시프트(`>> 1`)를 쓰는 이유:** `A^A`(자기 자신으로 돌아오는 루프)가 `0`이 되는 문제를 방지한다. `A >> 1`은 A가 아닌 값이므로, `A ^ (A >> 1) ≠ 0`이 된다. 덕분에 타이트 루프(A→A→A...)와 일반 루프를 구별할 수 있다.

**64KB(65536)가 적당한 이유:**

| 분기 수 | 충돌 확률 | 대표 타겟 |
|---------|-----------|-----------|
| 1,000개 | 0.75% | giflib, lzo |
| 2,000개 | 1.5% | zlib, tar, xz |
| 5,000개 | 3.5% | libpng, libwebp |
| 10,000개 | 7% | libxml |
| 20,000개 | 14% | sqlite |

대부분의 타겟(2k~10k 분기)에서 충돌률이 낮으면서, L2 캐시에 들어갈 만큼 작아 비교 연산이 마이크로초 단위로 빠르다.

### 3-3. 히트 카운트 버킷 — 세밀한 상태 구분

단순히 "이 엣지를 밟았는가/안 밟았는가"만 구분하면 정보가 너무 거칠다. AFL은 **히트 카운트를 버킷으로 분류**해 더 세밀하게 상태를 구분한다.

```
히트 카운트 버킷:
1 | 2 | 3 | 4-7 | 8-15 | 16-31 | 32-127 | 128+
```

예를 들어 어떤 코드 블록이 평소에 1번 실행되다가 갑자기 4번 실행된다면 버킷이 `1 → 4-7`로 바뀐다. 이것은 "새로운 동작"으로 간주되어 큐에 추가된다. 반면 47번에서 48번으로 바뀌는 것은 같은 버킷(`32-127`)이므로 무시된다.

이 방식 덕분에 루프를 2번 돌 때와 1번 돌 때를 구별할 수 있어, 오프-바이-원(off-by-one) 같은 미묘한 버그를 더 잘 찾아낸다.

### 3-4. 새로운 동작 감지 로직

```c
// afl-fuzz.c 중 — 새 커버리지 감지 (의사 코드)
u8 has_new_bits(u8* virgin_map) {
    u64* current = (u64*)trace_bits;   // 이번 실행 비트맵
    u64* virgin   = (u64*)virgin_map;  // 지금까지 본 적 없는 엣지 맵

    u32 i = MAP_SIZE / 8;
    u8  ret = 0;

    while (i--) {
        // 이번 실행에서 밟은 엣지 중에 virgin에 없는 게 있으면
        if (*current & *virgin) {
            // virgin 맵 업데이트
            *virgin &= ~*current;
            ret = 1;
        }
        current++;
        virgin++;
    }
    return ret;  // 1이면 "새 커버리지 발견"
}
```

64KB를 8바이트(u64) 단위로 처리하기 때문에, 전체 비트맵 비교가 단 8192번의 AND 연산으로 끝난다. 이것이 AFL이 마이크로초 단위로 커버리지를 판단할 수 있는 이유다.

---

## 4. Fork Server — 왜 빠른가

### 4-1. 문제: exec() 오버헤드

퍼징은 초당 수천 번 타겟을 실행해야 한다. 매번 처음부터 `execve()`로 프로세스를 시작하면 다음 비용이 매번 발생한다.

```
execve() 호출마다 발생하는 비용:
- 운영체제 프로세스 생성
- 동적 라이브러리 로드 및 링크 (.so 파일들)
- C 런타임 초기화 (libc, global constructors)
- 프로그램 자체 초기화 코드
```

큰 타겟(예: libxml, openssl 등)은 이 초기화만 수십 밀리초가 걸릴 수 있다. 초당 1000번 실행하려면 초기화를 1000번 해야 한다는 뜻이다.

### 4-2. 해결: Fork Server

AFL의 Fork Server는 이 문제를 `fork()`로 해결한다. `fork()`는 이미 실행 중인 프로세스를 복제하므로 초기화 비용이 들지 않는다.

```
[afl-fuzz]
    │
    │ (1) 타겟 최초 1회 실행
    ▼
[타겟 프로세스]
    │
    │ main() 진입 직전에 Fork Server 루프 진입
    │
    │◄──────── 파이프 ────────►[afl-fuzz]
    │
    │ 새 입력마다 fork()
    │
    ├── [자식 프로세스 1] → 입력 A 실행 → 종료
    ├── [자식 프로세스 2] → 입력 B 실행 → 종료
    └── [자식 프로세스 3] → 입력 C 실행 → 종료
```

타겟 바이너리의 모든 초기화(라이브러리 로드, 전역 변수 초기화 등)는 **처음 딱 한 번만** 일어난다. 이후 각 테스트케이스마다 이미 초기화된 상태에서 `fork()`만 호출한다.

### 4-3. Fork Server 구현 — 어디에 심어지나

Fork Server 코드는 계측 시 `afl-as.h`에 의해 **타겟 바이너리 안에 직접 삽입**된다. 별도 프로세스가 아니다.

실행 흐름은 아래와 같다.

```
타겟 바이너리 시작
    ↓
동적 링커 초기화 완료
    ↓
__afl_forkserver 함수 실행  ← afl-as.h가 삽입한 코드
    ↓
파이프를 통해 afl-fuzz에 "준비됐다" 신호 전송
    ↓
무한 루프 진입:
    ├── afl-fuzz로부터 "실행하라" 신호 수신
    ├── fork() 호출
    │     ├── [부모] 자식 PID를 afl-fuzz에 전송, waitpid() 대기
    │     └── [자식] Fork Server 루프 탈출 → main() 진입 → 실제 실행
    └── 자식 종료 시 상태코드 afl-fuzz에 전송, 다음 신호 대기
```

### 4-4. afl-fuzz와 Fork Server 간 통신

두 프로세스는 **파이프 2개**로 통신한다.

```c
// afl-fuzz.c 중 (의사 코드)

// 파이프 설정
int ctl_pipe[2];   // afl-fuzz → fork server (제어)
int st_pipe[2];    // fork server → afl-fuzz (상태)

// 실행 요청
write(ctl_pipe[1], &dummy, 4);    // "실행해" 신호 전송

// 자식 PID 수신
read(st_pipe[0], &child_pid, 4);

// 자식 종료 대기
read(st_pipe[0], &status, 4);    // 종료 상태 수신
```

파이프 메시지는 4바이트 정수 하나가 전부다. 오버헤드가 극히 작다.

### 4-5. 성능 효과

공식 문서에 따르면 Fork Server 방식은 단순 exec() 방식 대비 **수배~수십 배 빠르다.** 특히 초기화 비용이 큰 타겟일수록 효과가 크다.

---

## 5. Mutation — 입력을 어떻게 변이시키나

AFL의 변이 전략은 두 단계로 나뉜다: 결정론적(Deterministic)과 비결정론적(Non-deterministic). 각 seed는 이 단계들을 순서대로 거친다.

### 5-1. 결정론적 단계 (Deterministic Stages)

처음 seed를 처리할 때 **한 번씩 체계적으로** 실행된다. 재현 가능하고, 모든 위치를 빠짐없이 건드린다.

#### Bitflip — 비트 단위 뒤집기

```
원본:    01000001 01000010  (= "AB")

1비트씩: 11000001 01000010  (1비트 뒤집기)
         00000001 01000010
         ...

2비트씩: 11100001 01000010
         ...
```

1비트, 2비트, 4비트 단위로 슬라이딩하며 뒤집는다. 이 단계에서 AFL은 부수 효과로 **토큰을 자동 추출**한다. 특정 비트를 뒤집었을 때 커버리지가 크게 변하면, 그 위치가 "의미 있는 바이트"임을 알 수 있고, 이를 딕셔너리 토큰으로 기록한다.

#### Arithmetic — 정수 연산

각 바이트/워드/더블워드에 작은 정수를 더하거나 뺀다(`-35 ~ +35` 범위).

```
원본:  41 42 43 44

+1:    42 42 43 44
-1:    40 42 43 44
+35:   64 42 43 44
```

빅엔디안/리틀엔디안 양쪽 모두 시도한다. 정수 파서의 오버플로 버그를 노린다.

#### Interesting Values — 경계값 삽입

프로그래머가 자주 실수하는 **경계값들**을 직접 삽입한다.

```c
// afl-fuzz.c의 interesting values 정의
static s8  interesting_8[]  = { -128, -1, 0, 1, 16, 32, 64, 100, 127 };
static s16 interesting_16[] = { -32768, -129, 128, 255, 256, 512,
                                  1000, 1024, 4096, 32767 };
static s32 interesting_32[] = { -2147483648LL, -100663046, -32769,
                                  32768, 65535, 65536, 100663045,
                                  2147483647 };
```

`INT_MAX`, `UINT_MAX`, `0`, `-1` 같은 값들은 버퍼 크기 계산에서 오버플로를 자주 유발한다.

#### Dictionary / Token Insertion — 사전 삽입

`-x` 옵션으로 제공된 사전(예: HTML 태그, SQL 키워드, PNG 매직 바이트)을 입력 곳곳에 삽입하거나 교체한다.

```
사전: ["<script>", "SELECT", "PNG\x89"]

원본:  41 42 43 44
→      3C 73 63 72 69 70 74 3E 43 44  (앞에 "<script>" 삽입)
```

### 5-2. 비결정론적 단계 — Havoc

결정론적 단계가 끝나면 **Havoc** 단계로 진입한다. Havoc는 위의 변이들을 **무작위 순서로, 무작위 횟수로 조합**해서 적용한다.

```
Havoc 예시 (한 라운드):
원본: 41 42 43 44 45 46

1. 3번째 바이트에 interesting value 삽입 → 41 42 FF 44 45 46
2. 1-4번 바이트 삭제                     → 45 46
3. 앞에 랜덤 블록 삽입                   → 00 00 45 46
4. 첫 바이트 bitflip                     → FF 00 45 46
```

실제로 Havoc 단계에서 대부분의 버그가 발견된다. 결정론적 단계가 "체계적으로 모든 경우를 탐색"하는 준비 과정이라면, Havoc는 "예상치 못한 조합"을 만들어내는 창의적 단계다.

### 5-3. Splicing — 두 입력의 결합

큐에 여러 개의 seed가 쌓이면 AFL은 **두 입력을 이어붙이는** Splicing 단계를 수행한다.

```
seed A:  41 42 43 44 | 45 46 47 48
seed B:  61 62 63 64 | 65 66 67 68

Splicing (중간 지점에서 결합):
결과:    41 42 43 44 | 65 66 67 68
```

서로 다른 코드 경로를 탐색했던 두 입력을 합치면, 두 경로를 모두 탐색할 수 있는 입력이 탄생할 수 있다.

### 5-4. 큐 선택 — 어떤 seed를 먼저 변이하나

AFL은 모든 seed를 동등하게 처리하지 않는다. **성능 점수(performance score)** 를 계산해 각 seed에 변이 횟수를 차등 배분한다.

```c
// afl-fuzz.c 중 — 성능 점수 계산 (의사 코드)
u32 calculate_score(struct queue_entry* q) {

    u32 perf_score = 100;  // 기본 점수

    // 실행 속도가 평균보다 빠르면 점수 높이기
    if (q->exec_us * 0.1 > avg_exec_us) perf_score = 10;
    else if (q->exec_us * 2 < avg_exec_us) perf_score = 300;
    else if (q->exec_us * 3 < avg_exec_us) perf_score = 200;
    else if (q->exec_us * 4 < avg_exec_us) perf_score = 150;

    // 파일 크기가 작으면 점수 높이기
    if (q->len * 4 < avg_len) perf_score = 300;
    else if (q->len * 2 < avg_len) perf_score = 200;

    return perf_score;
}
```

점수가 높은 seed는 Havoc 단계에서 더 많은 변이 횟수를 할당받는다. **빠르고 작은 seed가 더 많이 변이된다** — 직관적으로도 합리적인 전략이다.

---

## 6. 전체 흐름 종합

지금까지 분석한 4개의 메커니즘이 실제로 어떻게 맞물려 돌아가는지 종합 흐름도로 정리한다.

```
━━━━━━━━━━━━━━━━━━━━ 컴파일 타임 ━━━━━━━━━━━━━━━━━━━━

소스코드
    └→ [afl-gcc] (wrapper)
           └→ gcc로 .s 파일 생성
           └→ [afl-as] (계측 어셈블러)
                  └→ 각 기본 블록 앞에 __afl_maybe_log() 삽입
                  └→ Fork Server 코드 삽입 (main() 직전)
                  └→ 계측된 바이너리 생성

━━━━━━━━━━━━━━━━━━━━ 실행 타임 ━━━━━━━━━━━━━━━━━━━━━━

[afl-fuzz 시작]
    │
    ├─ 공유 메모리(64KB bitmap) 생성
    ├─ 파이프 2개 생성 (ctl_pipe, st_pipe)
    ├─ 타겟 바이너리 최초 실행 → Fork Server 활성화
    │
    └─ 메인 퍼징 루프:
          │
          ├─ 1. 큐에서 seed 선택 (성능 점수 기반)
          ├─ 2. 변이 (결정론적 → Havoc → Splicing)
          ├─ 3. Fork Server에 실행 요청
          ├─ 4. fork() → 자식이 변이 입력으로 타겟 실행
          │       └─ 계측 코드가 bitmap[cur ^ prev]++ 업데이트
          ├─ 5. 자식 종료 → 상태 수신
          ├─ 6. 커버리지 분석
          │       ├─ 새 엣지 or 새 히트 버킷? → 큐에 추가 ✓
          │       ├─ 크래시? → crashes/ 에 저장
          │       └─ 변화 없음? → 버림
          └─ 7. 1번으로 돌아감
```

### 4개 메커니즘 요약표

| 메커니즘 | 관련 파일 | 핵심 역할 |
|----------|-----------|-----------|
| **Instrumentation** | `afl-gcc.c`, `afl-as.c`, `afl-as.h` | 어셈블러 단계에서 기본 블록마다 `bitmap[A^B]++` 삽입 |
| **Coverage** | `afl-fuzz.c` (`has_new_bits`) | 64KB SHM 비트맵으로 엣지 커버리지 + 히트 버킷 추적 |
| **Fork Server** | `afl-as.h` (삽입), `afl-fuzz.c` (제어) | 파이프 기반 부모-자식 통신으로 exec() 오버헤드 제거 |
| **Mutation** | `afl-fuzz.c` | 결정론적 단계 → Havoc → Splicing 순서로 진행 |

---

## 참고 자료

- [google/AFL — GitHub](https://github.com/google/AFL)
- [AFL technical_details.txt](https://github.com/google/AFL/blob/master/docs/technical_details.txt)
- [AFL afl-gcc.c](https://github.com/google/AFL/blob/master/afl-gcc.c)
- [AFL afl-as.c](https://github.com/google/AFL/blob/master/afl-as.c)
- [AFL afl-fuzz.c](https://github.com/google/AFL/blob/master/afl-fuzz.c)
- [AFL afl-as.h](https://github.com/google/AFL/blob/master/afl-as.h)
