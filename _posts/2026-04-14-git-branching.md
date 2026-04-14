---
layout: post
title: "Learn Git Branching 완전 정복 — 명령어 총정리"
date: 2026-04-14
category: 블로그/기술문서
author: yejunkim2000
tags: [git, branch, rebase, cherry-pick, remote]
---

> [learngitbranching.js.org](https://learngitbranching.js.org/)를 풀면서 배운 Git 명령어를 정리했습니다.

---

## 목차

1. [기본 커밋 & 브랜치](#1-기본-커밋--브랜치)
2. [머지 & 리베이스](#2-머지--리베이스)
3. [HEAD & 상대적 참조](#3-head--상대적-참조)
4. [작업 되돌리기](#4-작업-되돌리기)
5. [커밋 조작하기](#5-커밋-조작하기)
6. [태그 & Describe](#6-태그--describe)
7. [리모트 기초](#7-리모트-기초)
8. [리모트 심화](#8-리모트-심화)

---

## 1. 기본 커밋 & 브랜치

### git commit

현재 변경사항을 스냅샷으로 저장합니다.

```bash
git commit
git commit -m "커밋 메시지"
git commit --amend   # 가장 최근 커밋 수정
```

### git branch

브랜치는 특정 커밋을 가리키는 포인터입니다.

```bash
git branch 브랜치명          # 브랜치 생성
git branch -f main HEAD~3   # 브랜치를 강제로 특정 커밋으로 이동
git branch -d 브랜치명       # 브랜치 삭제
```

### git checkout / git switch

브랜치 또는 커밋으로 이동합니다.

```bash
git checkout 브랜치명
git checkout -b 브랜치명     # 브랜치 생성 + 이동 (한 번에)
git checkout C4             # 특정 커밋으로 HEAD 이동 (detached HEAD)
```

---

## 2. 머지 & 리베이스

### git merge

두 브랜치의 작업을 하나의 커밋으로 합칩니다.

```bash
git merge 브랜치명
```

```bash
# 예시
git checkout main
git merge bugFix   # bugFix를 main에 머지
```

### git rebase

커밋들을 복사해서 다른 위치에 붙여넣습니다. 히스토리가 깔끔해집니다.

```bash
git rebase main          # 현재 브랜치를 main 위로 rebase
git rebase main bugFix   # bugFix를 main 위로 rebase (이동 후 rebase)
```

### git rebase -i (인터랙티브 리베이스)

커밋 순서 변경, 삭제, 수정이 가능합니다.

```bash
git rebase -i HEAD~3     # 최근 3개 커밋을 대화형으로 편집
```

인터랙티브 창에서 사용할 수 있는 옵션입니다.

- `pick` — 커밋 유지
- `drop` — 커밋 삭제
- 순서를 바꾸면 커밋 순서가 재배치됨

---

## 3. HEAD & 상대적 참조

**HEAD**는 현재 작업 위치를 가리키는 포인터입니다.

### ^ 연산자 (한 칸 위)

```bash
git checkout HEAD^      # 부모 커밋으로 이동
git checkout HEAD^^     # 두 칸 위로 이동
git checkout main^      # main의 부모 커밋
git checkout HEAD^2     # 머지 커밋에서 두 번째 부모 선택
```

### ~ 연산자 (여러 칸 위)

```bash
git checkout HEAD~3          # 3칸 위로 이동
git branch -f main HEAD~3    # main을 3칸 위로 강제 이동
```

### 체이닝 활용

```bash
git checkout HEAD~^2~
git branch bugWork main^^2^
```

---

## 4. 작업 되돌리기

### git reset (로컬 전용)

브랜치를 이전 커밋으로 되돌립니다. 히스토리를 덮어씁니다.

```bash
git reset HEAD~1        # 한 단계 되돌리기
git reset --hard o/main # 원격 브랜치 상태로 강제 초기화
```

> 원격 저장소에 push된 브랜치에는 사용하지 마세요.

### git revert (공유 브랜치용)

되돌리는 내용을 담은 새 커밋을 만듭니다. 히스토리가 보존됩니다.

```bash
git revert HEAD         # 최근 커밋을 되돌리는 커밋 생성
```

---

## 5. 커밋 조작하기

### git cherry-pick

원하는 커밋만 골라서 현재 위치에 복사합니다.

```bash
git cherry-pick C2 C3 C4      # 여러 커밋 선택
git cherry-pick C3 C4 C7      # 순서대로 복사
```

### commit --amend 활용 패턴

히스토리 중간 커밋을 수정하고 싶을 때 사용하는 패턴입니다.

```bash
# 1. rebase -i로 수정할 커밋을 최신으로 올리기
git rebase -i HEAD~2

# 2. 수정
git commit --amend

# 3. 다시 순서 복원
git rebase -i HEAD~2

# 4. 브랜치 이동
git branch -f main HEAD
```

---

## 6. 태그 & Describe

### git tag

특정 커밋에 영구적인 이름을 붙입니다.

```bash
git tag v1.0          # 현재 HEAD에 태그
git tag v0 C1         # 특정 커밋에 태그
```

### git describe

가장 가까운 태그를 기준으로 현재 위치를 설명합니다.

```bash
git describe main     # 출력: v1_2_gC3 (태그_거리_커밋해시)
git describe HEAD
```

---

## 7. 리모트 기초

### git clone

원격 저장소를 로컬로 복사합니다.

```bash
git clone
```

### git fetch

원격 변경사항을 로컬로 가져옵니다. merge는 하지 않습니다.

```bash
git fetch             # 모든 원격 변경사항 가져오기
git fetch origin main # 특정 브랜치만 가져오기
```

### git pull

fetch + merge를 한 번에 실행합니다.

```bash
git pull
git pull --rebase     # fetch + rebase (히스토리 깔끔)
```

### git push

로컬 커밋을 원격에 업로드합니다.

```bash
git push
git push origin main
```

### 엇갈린 히스토리 해결 패턴

```bash
git pull --rebase
git push
```

### 잠긴 main 우회 패턴

```bash
git reset --hard o/main
git checkout -b feature C2
git push
```

---

## 8. 리모트 심화

### push/fetch refspec (source:destination)

```bash
# push: 로컬 source → 원격 destination
git push origin foo:main
git push origin main~1:foo

# fetch: 원격 source → 로컬 destination
git fetch origin main~1:foo
git fetch origin foo:main
```

### 빈 source — 삭제 & 생성

```bash
git push origin :foo    # 원격 foo 브랜치 삭제
git fetch origin :bar   # 로컬 bar 브랜치 생성
```

### 트래킹 브랜치 설정

```bash
# 방법 1: 브랜치 생성 시 트래킹 지정
git checkout -b side o/main

# 방법 2: 기존 브랜치에 트래킹 설정
git branch -u o/main side
```

### pull 인자

```bash
git pull origin bar:foo
git pull origin main:side
```

---

## 자주 쓰는 패턴 모음

| 상황 | 명령어 |
|------|--------|
| 브랜치 만들고 이동 | `git checkout -b 브랜치명` |
| 원격 최신 반영 후 push | `git pull --rebase && git push` |
| 특정 커밋만 가져오기 | `git cherry-pick C2 C3` |
| 커밋 순서 재배치 | `git rebase -i HEAD~N` |
| 브랜치 강제 이동 | `git branch -f main HEAD~3` |
| 원격 브랜치 추적 설정 | `git checkout -b 브랜치명 o/main` |
| 원격 브랜치 삭제 | `git push origin :브랜치명` |

---

<img width="1581" height="1253" alt="image" src="https://github.com/user-attachments/assets/6ae39df0-6d90-49fb-9804-d1d1f495e564" />
https://cdn.discordapp.com/attachments/1265900286422159474/1493608867118972998/image.png?ex=69df9727&is=69de45a7&hm=0adef78b1e13c421b41a58305a1862f61af5289cb811cf4701cba8958f8ce6dd&
