#CS/Python #Status/완료 
## 한 줄 정의

**"CPython이 한 번에 하나의 스레드만 Python 코드를 실행하도록 거는 전역 잠금"**

---

## Problem First — GIL이 없었을 때 문제

Python은 메모리 관리를 **Reference Counting**으로 합니다.

```python
import sys

a = [1, 2, 3]
b = a           # 같은 객체를 가리킴
print(sys.getrefcount(a))  # 3 (a, b, getrefcount 인자)
```

객체마다 "나를 가리키는 참조가 몇 개인가"를 카운트합니다. 0이 되면 메모리 해제합니다.

**멀티스레드에서 이게 왜 문제인가:**

```
스레드 1: refcount 읽음 → 2
스레드 2: refcount 읽음 → 2       ← 동시에 읽음
스레드 1: refcount - 1 → 1로 씀
스레드 2: refcount - 1 → 1로 씀   ← 둘 다 1로 씀

실제 참조는 0개인데 refcount는 1
→ 메모리 해제 안 됨 → 메모리 누수

반대 경우:
실제 참조는 1개인데 refcount가 0
→ 살아있는 객체가 해제됨 → 프로그램 크래시
```

**근본 문제: refcount 읽기/쓰기가 원자적이지 않습니다.**

---

## GIL이 하는 것

```
GIL = 인터프리터 전체에 거는 뮤텍스(mutex) 하나

스레드 1: GIL 획득 → Python 코드 실행
스레드 2: GIL 대기 ...
스레드 3: GIL 대기 ...

스레드 1: GIL 반납
스레드 2: GIL 획득 → Python 코드 실행
```

한 번에 하나의 스레드만 실행되니 refcount 동시 접근 문제가 사라집니다.

---

## 실질적 영향

### CPU bound — GIL이 치명적

```python
import threading

def count():
    n = 0
    for _ in range(100_000_000):
        n += 1

# 단일 스레드
# 소요: 4초

# 멀티 스레드 2개
t1 = threading.Thread(target=count)
t2 = threading.Thread(target=count)
# 소요: 4초 (빨라지지 않음)
# 심지어 GIL 경합 오버헤드로 더 느릴 수 있음
```

**코어가 8개여도 Python 코드는 1개 코어만 씁니다.**

---

### I/O bound — GIL이 문제 없음

```python
import threading
import requests

def fetch(url):
    response = requests.get(url)  # I/O 대기 중 GIL 반납
    return response

# I/O 대기 중에는 GIL을 놓아줌
# → 다른 스레드가 실행 가능
# → 멀티스레드 효과 있음
```

```
I/O 대기 (네트워크, 파일) 중에는
GIL을 자동으로 반납합니다.
→ 웹 크롤링, API 호출 등은 멀티스레드 효과 있음
```

---

## Python의 우회 방법

```python
# CPU bound 해결책 — 멀티프로세스
from multiprocessing import Pool

# 스레드가 아닌 프로세스로 분리
# 프로세스마다 독립적인 GIL → 병렬 실행 가능
# 단점: 프로세스 생성 비용, 메모리 공유 불가

with Pool(4) as p:
    p.map(count, range(4))

# I/O bound 해결책 — asyncio
import asyncio

async def fetch(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()
# 단일 스레드로 비동기 처리
# GIL 문제 자체를 우회
```

---

## Java와 비교

```
Java:
└── GIL 없음
    스레드마다 진짜 병렬 실행
    멀티코어 CPU 100% 활용 가능
    단, 개발자가 직접 동기화 책임
    (synchronized, Lock, volatile 등)

Python:
└── GIL 있음
    멀티스레드여도 Python 코드는 순차 실행
    단, 개발자가 동기화 신경 덜 씀
    refcount 안전성은 보장
```

---

## 정리

```
GIL이 생긴 이유:
    Reference Counting 기반 메모리 관리를
    멀티스레드에서 안전하게 하기 위해

GIL의 대가:
    CPU bound 작업에서 멀티코어 활용 불가

영향 없는 곳:
    I/O bound (네트워크, 파일)
    → 대기 중 GIL 반납하기 때문

우회:
    CPU bound → multiprocessing
    I/O bound → asyncio
```

**GIL은 Python이 단순함을 택한 대가입니다.** 메모리 안전성을 언어가 책임지는 대신, 멀티코어 병렬성을 개발자가 포기한 트레이드오프입니다.