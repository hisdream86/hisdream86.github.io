---
layout: post
title: 파이썬에서의 멀티스레딩 (Multi-threading) vs. 비동기 프로그래밍 (Asynchronous Programming)
parent: Python
tags: [python, async, asyncio, coroutine, future, task]
nav_order: 1
---

# 파이썬에서의 멀티스레딩 (Multi-threading) vs. 비동기 프로그래밍 (Asynchronous Programming)

우리가 개발을 하다보면 단일 스레드 기반의 동기식 (Synchronous) 프로그래밍으로는 쉽게 한계에 봉착하게 된다. 

예를 들어 아래와 같은 어플리케이션을 개발한다고 가정해보자.

- ① HTTP 요청을 수신해서
- ② Database 에 복잡한 쿼리를 요청하고 (i.e. 다소 시간이 걸리는)
- ③ 쿼리 결과를 응답한다

만약 이를 단일 스레드 기반의 동기식 프로그램으로 개발한다면 ② 의 완료를 기다리는 동안 어떠한 다른 작업도 수행하지 못한다 (그리고 당연스럽게도, 이 동안에는 CPU 리소스는 거의 사용되지 않는다).

클라이언트가 하나라면 별로 문제를 느끼지 못할수도 있다. 하지만 매우 많은 클라이언트들이 동시에 요청을 전송하면 어떻게 될까? 아마 가장 마지막에 요청을 전송한 클라이언트는 응답을 수신하기 위해 엄청난 시간을 대기해야 할지도 모른다 (컴퓨팅 자원은 남아도는데 말이다).

이러한 문제를 해결하기 위해서, 우리는 `멀티스레딩` 또는 `비동기 프로그래밍`를 활용할 수 있다.

## 멀티스레딩 (Multi-threading)
`스레드`는 프로세스로 부터 생성되는 명령 실행 단위이다. 기본적으로 프로세스보다 가벼우며, Context Switching 또한 빠르다. 그리고 하나의 프로세스에서 생성된 스레드들은 Stack 영역은 개별적으로 관리하되 Heap, Static, Code 영역은 서로 공유가 가능하다.

만약 하나의 프로세스가 여러 개의 스레드 (즉, 멀티스레드) 기반으로 동작한다면, 각각의 스레드들은 반복적으로 짧은 시간의 CPU 시간을 할당받으면서 동작하게 되고, 이는 마치 여러 개의 스레드가 동시에 동작하는것 처럼 보이게 해준다. 그리고 만약 Multi-Core CPU 의 경우, 코어 개수만큼의 스레드들이 실제 동시에 동작도 가능하다.

이러한 멀티스레딩을 도입하면 위에서 설명한 문제는 손쉽게 해결된다. 이제 새로운 요청이 올 때 마다 새로운 스레드를 생성하여 요청을 처리해보자. 하나의 스레드가 (2) 에서 대기하더라도, 다른 스레드는 자신의 차례에 주어진 CPU 시간을 활용하면서 요청을 처리하기 때문에 얼마든지 새로운 요청을 병렬적으로 처리 가능하다.

하지만 멀티스레딩에는 일부 단점이 존재한다. 그리고 이것이 바로 비동기 프로그램을 사용하는 이유이다.

**(1) 복잡성**

멀티스레드 프로그램은 기본적으로 고민을 해야 할 부분들이 다소 많으며 (ex. Race condition 을 해결하기 위한 동기화), 문제가 발생할 경우 디버깅이 매우 어렵다. 즉, 멀티스레드 프로그램은 복잡한 개발 및 운영을 요구할 가능성이 높다.

**(2) Context Switching 비용**

프로세스 보다는 가볍긴 하지만, 스레드의 Context Switching 으로 인한 비용도 무시할 수 없다.

매우 많은 개수의 스레드가 (ex. 10,000 개) 단순한 I/O 작업을 처리하는 상황을 가정해보자. 대부분의 스레드가 I/O 작업의 완료를 대기하고 있을 뿐인데, 개별 스레드들의 I/O 작업 완료를 확인하기 위해서 반복적으로 Context Switching 이 발생해야 한다. 그리고, 단순히 I/O 완료를 대기중인 스레드들에게 할당하는 CPU 자원의 낭비도 무시하기 힘들다 (비동기 프로그래밍의 경우, I/O 작업이 완료되지 않으면 해당 Task 에 CPU 자원을 할당하지 않기 때문에 보다 효율적이다).

당연하지만, 이러한 비용은 높은 병렬성이 요구될수록 더욱 증가한다.

**(3) GIL (Global Interpreter Lock) 문제**

앞서 설명했던 것 처럼, Mult-Core CPU 환경에서는 동시에 최대 코어 개수만큼의 스레드가 동시에 동작될 수 있어야 한다. (하이퍼스레딩이 적용되면 얘기가 좀 다르긴 한데, 여기서는 넘어간다)

하지만, 파이썬에만 해당되는 문제이긴 한데  `GIL` 이라는 녀석 덕분에 파이썬 멀티스레드는 우리가 원하는 방식대로 동작하지 않는다. 

(아래는 [파이썬 Wiki](https://wiki.python.org/moin/GlobalInterpreterLock) 에서 설명하고 있는 GIL)
> In CPython, the global interpreter lock, or GIL, is a mutex that protects access to Python objects, preventing multiple threads from executing Python bytecodes at once. The GIL prevents race conditions and ensures thread safety. A nice explanation of how the Python GIL helps in these areas can be found here. In short, this mutex is necessary mainly because CPython's memory management is not thread-safe.

위의 글에서 GIL 은 파이썬 객체를 보호하기 위해 멀티 스레드들이 동시에 파이썬 바이드코드를 실행하는 것을 방지하는 `뮤텍스 (Mutex)` 라고 설명하고 있다. 쉽게 말하면, CPU 코어가 몇개던지 관계 없이 한 번에 하나의 스레드만 실행될 수 있도록 동기화시키는 `락 (Lock)` 이다.

그렇다면 이게 왜 필요한걸까? 짧게만 설명하면, 파이썬은 내부적으로 생성된 모든 객체들에 대한 `레퍼런스 카운트 (Reference Count)` 를 `refcount` 라는 객체에서 관리하도록 구현되어 있다. `refcount` 값은 각 객체가 참조되거나 해제되는 시점에 갱신되며, 레퍼런스 카운트가 0이 된 객체는 `파이썬 GC` (Garbage Collector) 메모리에서 해제한다.

(아래는 파이썬 코드 내에서 객체의 레퍼런스 카운트를 확인할 수 있는 간단한 예제)
```python
import sys
a = 1
print(sys.getrefcount(a))
```

파이썬 객체가 멀티 스레드 사이에서 동기화되지 않는다면 Race Condition 으로 인해 레퍼런스 카운트가 정상적으로 관리되지 못하고, 이는 프로그램을 버그투성이로 만들것이다 (갑자기 객체가 메모리에서 해제되거나, 아니면 영영 해제되지 않거나). 그렇다고 해서 파이썬 객체에 접근 시 마다 락을 걸게 되면 이건 또 너무나도 비효율적이다. 이러한 연유로, 파이썬에는 한 번에 하나의 스레드만 실행되도록 보장하는 GIL 을 도입되게 되었다.

사실 어플리케이션이 I/O 위주라면 GIL 으로 인한 영향은 다소 적다고 볼 수 있다 (I/O 작업 자체는 파이썬 외부에서 병렬적으로 동작 가능하므로). 하지만, 어쨋든 파이썬 멀티스레드 프로그램 작성하기 위해서는 반드시 GIL을 고려해야 함은 변함이 없다.

## 비동기 프로그래밍 (Asynchronous Programming)

`비동기 프로그래밍`은 멀티스레드 기반의 구현에서 오는 문제들을 극복하기 위해 대두된 개념으로, 일련의 `비동기 태스크(Asynchronous Task)` 들을 동시에 처리할 수 있도록 지원하여 단일 스레드로도 동시성을 확보할 수 있도록 하는 프로그래밍 방법이다.

처음에 얘기했던 예제 어플리케이션을 다시 가져와보자.

- ① HTTP 요청을 수신해서
- ② Database 에 복잡한 쿼리를 요청하고 (i.e. 다소 시간이 걸리는)
- ③ 쿼리 결과를 응답한다

비동기 프로그래밍에서는 클라이언트들의 요청을 동시에 처리하기 위해 각각의 별도로 스레드를 생성하지 않는다. 대신 각각의 요청마다 ② 를 처리하기 위한 비동기 태스크를 생성하는데, 이는 함수 요청 시 리턴을 기다리면 결과를 받는 방식과는 다르게 **처리 결과를 비동기식으로 전달**하는 방식을 사용한다. 비동기 태스크들은 `이벤트 루프 (Event Loop)` 에 스케줄링시켜서 동작시킬 수 있는데, 바로 이 이벤트 루프가 단일 스레드 기반으로 `동시성 (Concurrency)` 을 확보하는 것의 핵심이라고 볼 수 있다.

이벤트 루프는 그 이름에서 알 수 있듯이, 루프를 돌면서 스케줄링된 비동기 태스크들을 수행하는 역할을 수행한다. 이 때 각각의 태스크들은 일련의 작업을 수행하면서 (e.x. ② 와 같이 오래 걸리는 I/O 작업) 자신의 완료와 관계없이 제어권을 다시 이벤트 루프로 즉시 넘긴다. 제어권을 넘겨받은 이벤트 루프는 이전 태스크의 완료 유무와 관계없이 다른 태스크를 수행시킬 수 있다. 

아래의 코드 예제를 통해 좀 더 쉽게 동작을 이해를 해보자.

```python
import asyncio
from datetime import datetime


async def main(): # (1)
    start = datetime.now()
    results = await asyncio.gather(task_1(), task_2()) # (3)
    end = datetime.now()
    
    print("execution time: ", end - start)
    print("results: ", results)


async def task_1():    
    await asyncio.sleep(1) # 1초가 걸리는 I/O 작업을 가정함
    return "task_1()"


async def task_2():    
    await asyncio.sleep(2) # 2초가 걸리는 I/O 작업을 가정함
    return "task_2()"

asyncio.run(main()) # (2)

# 결과
# execution time:  0:00:02.001574
# results:  ['task_1()', 'task_2()']
```

`asyncio` 는 파이썬에서 3.5 부터 추가된 비동기 라이브러리로, 이에 대한 내용은 별도로 작성해볼 예정이므로 본 문서에서 자세한 내용은 담지 않는다.

**(1) async def main()**

`async` 를 함수 선언 앞에 추가할 경우, 해당 함수는 코루틴 함수가 (Coroutine function) 된다. 코루틴 함수를 호출할 경우 함수가 실행되어 결과가 리턴되는 것이 아니라 해당 함수를 기반으로 한 코루틴 객체 (Coroutine object) 라는 것이 생성되어 리턴된다. 이 코루틴 객체는 이벤트 루프에 스케줄링하여 실행시킬 수 있다. (함수 task_1() 과 task_2() 도 마찬가지이다.)

**(2) asyncio.run(main())**

`asyncio.run()` 함수는 Awaitable 객체 (비동기 태스크, 총 3가지 종류가 있음: Coroutine, Future, Task) 를 스케줄링해주는 상위 레벨의 함수이다. 스레드에 이벤트 루프가 없으면 이를 생성해주고 태스크의 완료를 보장해주는 등, 편리하게 비동기 태스크를 실행할 수 있도록 기능을 제공한다. 

**(3) await asyncio.gather(task_1(), task_2())**

`asyncio.gather()` 함수는 인자로 전달받은 n 개의 Awaitable 들을 루프에 스케줄링하고, 모든 태스크가 완료될 때 까지 기다린 뒤 결과를 List 형식으로 반환해주는 일을 대행해주는 Awaitable 을 만들어주는 상위 레벨의 함수이다.

본 예제에서는 이벤트 루프에 `task_1()` 과 `task_2()` 를 동시에 스케줄링하고, 그 결과를 results 변수에 담고 있다.

여기서 `await` 키워드를 확인할 수 있는데, 이는 Awaitable 객체 앞에 작성할 수 있는 키워드이다. 만약 이 키워드를 작성하면 해당 Awaitable 의 작업이 모두 완료된 결과를 보장받고 난 뒤 다음 코드 라인이 실행되도록 할 수 있다. (참고로, 인터프리터가 await 키워드를 만나게 되면 해당하는 Awaitable 을 루프에 스케줄링 한 뒤 제어권을 이벤트 루프로 넘긴다. 따라서, Awaitable 이 완료될 동안 다른 비동기 태스크들을 동작시킬 수 있다!)

**결과**

결과를 확인하면 이벤트 루프 내에서 `task_1()` 과 `task_2()` 가 I/O 작업을 동시에 수행하였고, 따라서 최종적으로 소요된 시간이 약 2초 임을 확인할 수 있다. 그리고 코드에서 참고할 수 있듯이 비동기 프로그래밍을 활용하면 매우 간단하게 동시성을 확보할 수 있음을 확인할 수 있다.

## 결론

비동기 프로그래밍을 활용하면 단일 스레드로 (이벤트 루프 기반) 손쉽게 동시성을 확보할 수 있음을 확인했다. 비동기 프로그래밍은 멀티스레딩에 비해 프로그램의 복잡성을 줄일 수 있으며, 스레드 Context Switching 에서 오는 낭비도 줄일 수 있음을 확인했다.

하지만, 그렇다고 해서 비동기 프로그래밍이 무조건 멀티스레딩보다 우수하다는 것은 아니다. 비동기 프로그래밍이 멀티스레딩에 비해 우위를 점할 수 있는 것은 어플리케이션이 **I/O Intensive** 한 경우 (i.e. I/O 위주의 작업을 많이 하는 경우) 이다. 만약 어플리케이션이 **CPU Intensive** 한 경우에는 멀티스레드 기반의 프로그램이 더 나은 성능을 보일 수 있다. 따라서 구현하고자 하는 어플리케이션의 성격을 토대로 어떤 기법이 적절할지 판단하는 과정이 반드시 필요하다고 볼 수 있겠다.