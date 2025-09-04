---
layout: single
title:  "[Python] Multiprocessing"
typora-root-url: ../
categories: Python
tag: [Python, Multiprocessing]
author_profile: false
sidebar:
    nav: 'counts'
search: true
use_math: false
redirect_from:
  - /python/multiprocessing
published: true
---

# Python Multiprocessing: Shared Memory, Race Condition, and Lock

파이썬은 기본적으로 <span style="color:red">**GIL(Global Interpreter Lock)**</span> 때문에 CPU-bound 연산에서는 멀티스레딩이 제대로 성능을 내지 못합니다. 따라서 CPU 코어를 병렬로 활용하기 위해서는 `multiprocessing` 모듈을 사용하는 것이 일반적입니다. 이 글에서는 파이썬을 이용해서 프로세스 간 데이터 공유 방식, Race Condition 문제, 그리고 Lock을 통한 동기화 기법을 정리합니다.

## 1. 프로세스 간 메모리 독립성

```python
import multiprocessing

data = []

def append_data():
    for i in range(10):
        data.append(i)

if __name__ == "__main__":
    ps = [multiprocessing.Process(target=append_data) for _ in range(4)]
    
    for p in ps:
        p.start()

    for p in ps:
        p.join()

    print(f"data 길이 : {len(data)}")
```

- 메인 프로세스와 서브 프로세스는 독립적인 메모리 공간을 사용합니다.
- 따라서 `data` 리스트는 공유되지 않으며, 최종 길이는 항상 `0`이 됩니다.


## 2. Manager를 활용한 프로세스 간 데이터 공유

```python
import multiprocessing

def append_data(shared_data):
    for i in range(10):
        shared_data.append(i)

if __name__ == "__main__":
    with multiprocessing.Manager() as manager:
        shared_data = manager.list()

        ps = [multiprocessing.Process(target=append_data, args=(shared_data,)) for _ in range(4)]

        for p in ps:
            p.start()

        for p in ps:
            p.join()

        print(f"data 길이 : {len(shared_data)}")
```

- `multiprocessing.Manager()`는 별도의 서버 프로세스를 백그라운드로 띄워서 데이터 객체를 관리합니다.
- 각 서브 프로세스는 Proxy 객체를 통해 공유 데이터에 접근합니다.
- 실행 결과: `data 길이 : 40`.

⚠️ 단점: IPC(Inter-Process Communication)를 통해 동작하기 때문에 성능 오버헤드가 있습니다.


## 3. 공유 메모리 Array

```python
import multiprocessing

def append_data(shared_data, idx):
    for i in range(10):
        shared_data[idx+i] = i

if __name__ == "__main__":
    shared_data = multiprocessing.Array('i', 40) # 'i'는 C 타입 int

    ps = [multiprocessing.Process(target=append_data, args=(shared_data, i*10)) for i in range(4)]

    for p in ps:
        p.start()

    for p in ps:
        p.join()

    print(f"data 길이: {len(shared_data)}")
```

- `multiprocessing.Array`는 저수준의 **공유 메모리(SHM)**를 직접 사용합니다.
- IPC 오버헤드가 적고 속도가 빠릅니다.
- 단, **고정 크기**만 지원.


## 4. Race Condition 상황

```python
import multiprocessing

def append_data(shared_data):
    for i in range(10):
        current = list(shared_data)
        current.append(i)
        shared_data[:] = current

if __name__ == "__main__":
    with multiprocessing.Manager() as manager:
        shared_data = manager.list()

        ps = [multiprocessing.Process(target=append_data, args=(shared_data,)) for _ in range(4)]

        for p in ps:
            p.start()

        for p in ps:
            p.join()

        print(f"data 길이 : {len(shared_data)}")
```

- 각 프로세스가 `shared_data`를 읽은 뒤 수정하고 다시 덮어쓰는 과정에서 **경쟁 상태(Race Condition)** 발생.
- 실행할 때마다 결과가 다르게 발생.


## 5. Lock으로 Race Condition 해결

```python
import multiprocessing

def append_data(shared_data, lock):
    for i in range(10):
        with lock:
            current = list(shared_data)
            current.append(i)
            shared_data[:] = current

if __name__ == "__main__":
    with multiprocessing.Manager() as manager:
        shared_data = manager.list()
        lock = multiprocessing.Lock()

        ps = [multiprocessing.Process(target=append_data, args=(shared_data, lock)) for _ in range(4)]

        for p in ps:
            p.start()

        for p in ps:
            p.join()

        print(f"data 길이 : {len(shared_data)}")
```

- `multiprocessing.Lock()`은 임계 구역(Critical Section)을 보호합니다.
- 하나의 프로세스만 안전하게 `shared_data`를 수정할 수 있습니다.
- 결과는 항상 `data 길이 : 40`.


## 요약
- 기본적으로 프로세스 간 메모리는 공유되지 않는다.
- `Manager` → 편리하지만 느림 (Proxy 기반 IPC).
- `Array` → 빠르지만 크기가 고정.
- 동시 접근 시 Race Condition이 발생할 수 있으며, `Lock`으로 동기화가 필요하다.

멀티프로세싱을 제대로 활용하려면 **작업의 특성(읽기 위주 vs 쓰기 동기화 필요)**에 따라 Manager, Array, Lock을 적절히 선택하는 것이 중요합니다.
