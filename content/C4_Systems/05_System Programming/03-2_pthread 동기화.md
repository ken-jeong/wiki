---
aliases: []
type: Lecture
tags:
  - 3-2/UNIX시스템프로그래밍
draft: false
date: 2026-10-12
prev:
  - "[[03-1_pthread]]"
next:
  - "[[04_시간 관리]]"
---
## 1. 먼저 전체 구조 잡기
스레드 동기화는 크게 두 가지 문제를 해결한다.
1. **공유 데이터 보호**: 여러 스레드가 같은 데이터를 동시에 건드려 결과가 망가지지 않게 한다.
2. **실행 순서 조정**: 다른 스레드의 종료, 데이터 준비, 자원 확보 등을 기다린다.

| 도구             | 핵심 목적                 | 기억할 표현           | 대표 상황               |
| -------------- | --------------------- | ---------------- | ------------------- |
| `pthread_join` | 다른 스레드의 종료 대기         | “네가 끝나면 진행할게”    | 작업 스레드가 끝난 후 결과 수집  |
| **Mutex**          | 공유 데이터에 대한 배타적 접근     | “한 번에 한 명”       | 공유 카운터 갱신           |
| **조건 변수**          | 공유 상태가 조건을 만족할 때까지 대기 | “준비되면 알려줘”       | 버퍼에 데이터가 들어올 때까지 대기 |
| **읽기/쓰기 락**        | 읽기는 동시 허용, 쓰기는 독점     | “여럿이 읽되, 쓰기는 혼자” | 읽기가 많은 공유 자료구조      |
| **세마포어**           | 사용 가능한 자원·허가 수 관리     | “허가증이 있어야 진행”    | 동시에 사용할 수 있는 자원이 N개 |

## 2. 왜 동기화가 필요한가?
### 2.1. 상호 배제와 임계 구역
- **상호 배제(Mutual Exclusion)**: 특정 공유 자원에 한 번에 하나의 스레드만 접근하도록 제한하는 것
- **임계 구역(Critical Section)**: 공유 데이터를 읽거나 변경해서, 다른 스레드의 접근과 충돌할 수 있는 코드 구간
- **원자성(Atomicity)**: 다른 스레드 관점에서 연산이 중간에 나뉘지 않고 하나의 동작처럼 수행되는 성질

**핵심**: C 코드 한 줄이라고 해서 원자적 연산은 아니다.

### 2.2. 보충 예제: `count++`의 경쟁 상태
`count`가 0일 때 두 스레드가 동시에 `count++`를 실행한다고 하자.
이해를 위해 연산을 읽기·계산·쓰기 단계로 나누면 다음 상황이 가능하다.

| 순서 | 스레드 A | 스레드 B | 공유 `count` |
|---|---|---|---|
| 1 | 0을 읽음 | | 0 |
| 2 | | 0을 읽음 | 0 |
| 3 | 1을 계산하고 저장 | | 1 |
| 4 | | 1을 계산하고 저장 | 1 |

두 번 증가시켰지만 하나의 갱신이 사라졌다.
이러한 실행 순서 의존 문제를 **경쟁 상태(Race Condition)**라고 한다.

> 이 표는 문제를 설명하기 위한 모형이다.
> 실제 C에서 일반 변수에 동기화 없이 충돌하는 접근을 하면 데이터 레이스가 발생하므로, 결과가 반드시 1 또는 2로만 제한되는 것도 아니다.

### 2.3. `pthread_join()`의 역할
```c
pthread_join(worker, NULL);
```

호출한 스레드는 `worker`가 종료할 때까지 기다린다.
하지만 작업 스레드들이 실행 중에 공유 데이터를 동시에 변경하는 문제는 별도의 동기화로 해결해야 한다.

### cf.
**블로킹 락** (Blocking Lock)
- 락을 얻지 못하면 스레드를 잠들게 하고, 락이 풀리면 깨워서 다시 시도한다.
- 기다리는 동안 CPU를 적게 쓰지만, 스레드를 재우고 깨우는 비용이 있다.

**논블로킹 락** (Non-blocking Lock)
- 즉시 실패를 반환한다. (`tryLock` 등)
- 기다리지 않고 다른 작업을 하거나 나중에 재시도할 수 있다.

**스핀 락** (Spin Lock)
- 락을 얻을 때까지 반복해서 확인한다.
- CPU를 계속 사용하므로 대기 시간이 매우 짧을 때 적합하다.

==스핀락과 처리기 환경== #3-2/UNIX시스템프로그래밍/출제
- 단일 처리기: 동시 실행이 불가능하므로, 락 해제를 못 해 CPU 사이클이 낭비되거나 교착 상태가 발생한다.
- 다중 처리기: 동시 실행이 가능하므로, 다른 처리기에서 짧은 시간 안에 락 해제를 해준다면 효율성이 매우 높아진다.

## 3. 뮤텍스: `pthread_mutex_*`
> **뮤텍스 (Mutex; Mutual Exclusion)**
> 공유 데이터에 잠금 걸기

### 3.1. 동작 흐름
```text
초기화 → lock → 임계 구역 수행 → unlock → 사용 종료 후 destroy
```

다른 스레드가 뮤텍스를 소유하고 있으면 `pthread_mutex_lock()`을 호출한 스레드는 잠금이 풀릴 때까지 대기한다.

**보호가 성립하는 조건**
- 같은 공유 데이터에 접근하는 모든 스레드가 같은 뮤텍스를 사용해야 한다.
- 일부 스레드만 잠금을 지키면 데이터가 보호되지 않는다.

### 3.2. 선언과 초기화
```c
#include <pthread.h>

// 방법 1: 정적 초기화
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// 방법 2: 동적 초기화
pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL);  // 기본 속성으로 초기화
```

- 사용 전에 반드시 초기화한다.
- 기본적으로 처음에는 잠기지 않은 상태다.
- 같은 객체를 두 방법으로 중복 초기화하지 않는다.
- 원자료의 `THREAD_MUTEX_INITIALIZER`는 **`PTHREAD_MUTEX_INITIALIZER`**로 고쳐야 한다.

### 3.3. 주요 함수
```c
pthread_mutex_lock(&m);  // 잠금 획득, 다른 스레드가 소유 중이면 대기
pthread_mutex_trylock(&m);  // 기다리지 않고 획득 시도, 사용 중이면 EBUSY 반환

/* 임계 구역 */

pthread_mutex_unlock(&m);  // 자신이 소유한 잠금 해제

pthread_mutex_destroy(&m);  // 더 이상 사용하지 않는 Mutex 정리
```

### 3.4. 예제: 공유 카운터 보호
```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int count = 0;  // 공유 변수

void increment(void) {
    pthread_mutex_lock(&mutex);
    count++;  // 임계 구역
    pthread_mutex_unlock(&mutex);
}
```

`count`를 읽는 코드도 다른 스레드의 쓰기와 동시에 실행될 수 있다면 같은 Mutex로 보호한다.
잠금은 `count++` 자체를 특별한 기계 명령으로 바꾸는 것이 아니라, 잠금 규칙을 지키는 스레드들 사이의 접근을 직렬화한다.

### 3.5. `lock`과 `trylock` 비교
| 구분           | `lock`                    | `trylock`                |
| ------------ | ------------------------- | ------------------------ |
| 잠금 획득 가능     | 획득하고 진행                   | 획득하고 진행                  |
| 다른 스레드가 소유 중 | 대기                        | 즉시 `EBUSY` 반환            |
| 적절한 활용       | 자원이 필요하므로 기다려야 함          | 실패 시 다른 일을 하거나 나중에 다시 시도 |
| 주의점          | 여러 락의 획득 순서가 꼬이면 교착 상태 가능 | 반복 호출하면 폴링 비용과 기아 가능     |

- **폴링(Polling)**: 상태가 바뀌었는지 반복해서 확인하는 것
- **기아(Starvation)**: 다른 스레드들은 진행하지만 특정 스레드는 계속 기회를 얻지 못하는 것
- **교착 상태(Deadlock)**: 서로 상대가 가진 자원을 기다려 아무도 진행하지 못하는 것
- **우선순위 역전(Priority Inversion)**: 높은 우선순위 스레드가 낮은 우선순위 스레드의 잠금 때문에 진행하지 못하는 현상

`trylock`의 활용
- 실시간 상태 확인, 교착 상태·우선순위 역전 대응
- 단, `trylock`만 호출한다고 이 문제들이 자동 해결되지는 않는다.
- 실패 시 잠금을 반납하거나 다른 작업을 하는 등 별도의 전략이 필요하다.

**보충**
- 임계 구역은 데이터의 일관성을 지키기에 충분한 범위를 포함해야 한다.
- 잠금을 오래 유지하는 불필요한 작업은 줄이되, 보호해야 할 연산을 분리해서는 안 된다.
- Mutex를 파괴할 때는 잠금 소유자나 대기자 등 사용 중인 스레드가 없어야 한다.

## 4. 조건 변수: `pthread_cond_*`
> **조건 변수 (Condition Variable)**
> 조건이 만족될 때까지 기다리기

### 4.1. Mutex와 역할이 다르다
- **Mutex**: 공유 상태를 안전하게 확인·변경하도록 보호한다.
- **조건 변수**: 그 상태가 원하는 조건을 만족할 때까지 스레드를 대기시킨다.

예:
- `count > 0`이 실제 조건이고, 조건 변수는 “데이터가 준비되었을 수 있으니 다시 확인하라”는 알림을 전달한다.
- 조건 변수 자체가 `count` 값이나 참·거짓 상태를 보관하는 것은 아니다.

### 4.2. 선언과 초기화
```c
// 방법 1: 정적 초기화
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

// 방법 2: 동적 초기화
pthread_cond_t cond;
pthread_cond_init(&cond, NULL);
```

### 4.3. 주요 함수
```c
pthread_cond_wait(&cond, &mutex);  // Mutex를 놓고 대기, 깨어난 뒤 Mutex를 다시 획득
pthread_cond_timedwait(&cond, &mutex, &time);  // 지정한 절대 시각까지 대기 가능

pthread_cond_signal(&cond);  // 하나의 대기자를 깨운다.
pthread_cond_broadcast(&cond);  // 모든 대기자를 깨운다.

pthread_cond_destroy(&cond);  // 사용 종료 후, 대기자가 없는 상태에서 정리
```

> **보충**
> `timedwait`의 세 번째 인자는 단순한 대기 시간(초)이 아니라, 조건 변수에 설정된 시계 기준의 절대 마감 시각이다.
> 시간 초과 시 `ETIMEDOUT`을 반환한다.
> [POSIX 조건 대기 명세](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3p.html)

### 4.4. `pthread_cond_wait()`의 핵심 흐름
```text
호출 전: 호출 스레드가 Mutex 소유
    ↓
Mutex 해제와 조건 변수 대기를 원자적으로 수행
    ↓
알림 등으로 대기에서 풀림
    ↓
Mutex 재획득 시도 - 다른 스레드가 소유 중이면 추가 대기
    ↓
정상 반환: 호출 스레드가 Mutex를 다시 소유
```

따라서 **깨워졌다는 것과 즉시 임계 구역에 진입했다는 것은 다르다.**
`broadcast`로 여러 스레드를 깨워도, 같은 Mutex를 소유하는 것은 한 번에 하나의 스레드다.

### 4.5. 예제: 데이터 준비를 기다리는 기본 패턴
```c
pthread_mutex_t count_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t count_threshold_cv = PTHREAD_COND_INITIALIZER;
int count = 0;
```

```c
/* 값을 변경하는 스레드 */
pthread_mutex_lock(&count_mutex);

count++;
if (count == COUNT_LIMIT) {
    pthread_cond_signal(&count_threshold_cv);
}

pthread_mutex_unlock(&count_mutex);
```

```c
/* 조건을 기다리는 스레드 */
pthread_mutex_lock(&count_mutex);

while (count < COUNT_LIMIT) {  // while로 검사
    pthread_cond_wait(&count_threshold_cv, &count_mutex);
}

pthread_mutex_unlock(&count_mutex);
```

**왜 `if`가 아니라 `while`인가?**
1. **허위 깨움(Spurious Wakeup)**: 조건이 충족되지 않아도 대기에서 반환할 수 있다.
2. 다른 스레드가 먼저 Mutex를 획득해 데이터를 소비하면, 자신이 락을 얻을 때는 조건이 다시 거짓일 수 있다.

그러므로 반환 후에는 반드시 공유 상태를 다시 확인한다.
[POSIX 조건 대기 명세](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3p.html)

### 4.6. 알림은 쌓이지 않는다
대기자가 없을 때 `pthread_cond_signal()`을 호출해도 미래의 대기자를 위한 알림이 저장되지 않는다.

하지만 위 패턴에서는 `ready`라는 **공유 상태**가 남아 있다.
나중에 온 스레드가 Mutex 안에서 `ready`를 확인하고 이미 참이면 대기하지 않는다.

**암기**: 조건은 공유 변수에 저장하고, 조건 변수는 상태를 다시 확인할 계기를 준다.

> **정확성 보충**
> 강의는 `signal`도 반드시 Mutex를 잠근 상태에서 호출해야 한다고 설명한다.
> POSIX는 Mutex를 소유하지 않은 호출도 허용한다.
> 학습용 기본 패턴으로는 **같은 Mutex 아래에서 상태를 변경하고 알림까지 보내는 방식**을 익히면 좋다.
> 또한 깨우는 순서를 일반적인 FIFO로 가정하면 안 된다.
> [POSIX 알림 명세](https://pubs.opengroup.org/onlinepubs/000095399/functions/pthread_cond_signal.html)

> 2026-10-13

## 5. 읽기-쓰기 락: `pthread_rwlock_*`
==읽기-쓰기 락 (Read-Write Lock; RW Lock)== #3-2/UNIX시스템프로그래밍/출제
- 여러 읽기 또는 하나의 쓰기만 허용한다.
	- 읽기는 공유(shared)하고, 쓰기는 독점(exclusive)한다.
- 읽기 작업에 대한 높은 동시성(읽기 병렬성/처리량)을 제공한다.

### 5.1. 접근 규칙
| 현재 락 상태               | 새 읽기 요청          | 새 쓰기 요청 |
| --------------------- | ---------------- | ------- |
| 아무도 소유하지 않음           | 가능               | 가능      |
| **읽기 락**을 소유한 스레드가 있음 | **다른 읽기와 공존 가능** | **대기**  |
| **쓰기 락**을 소유한 스레드가 있음 | **대기**           | **대기**  |

- 여러 읽기 스레드는 동시에 접근할 수 있다.
- 쓰기 스레드는 다른 읽기·쓰기 스레드와 동시에 접근할 수 없다.
- 읽기 락을 획득한 코드에서 보호 대상 데이터를 수정하면 안 된다.

> **보충**
> 표는 락의 공존 가능성을 나타낸다.
> 이미 대기 중인 쓰기 스레드가 있을 때 새 읽기 요청을 허용할지는 구현과 스케줄링 정책에 영향을 받을 수 있다.
> [POSIX 읽기 락 명세](https://man7.org/linux/man-pages/man3/pthread_rwlock_rdlock.3p.html)

### 5.2. 선언과 초기화
```c
// 방법 1: 정적 초기화
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

// 방법 2: 동적 초기화
pthread_rwlock_t rwlock;
pthread_rwlock_init(&rwlock, NULL);
```

### 5.3. 주요 함수
```c
/* 읽기 락 요청 */
pthread_rwlock_rdlock(&rwlock);  // 블로킹
pthread_rwlock_tryrdlock(&rwlock);  // 논블로킹
pthread_rwlock_timedrdlock(&rwlock, &deadline);  // 시간 제한

/* 쓰기 락 요청 */
pthread_rwlock_wrlock(&rwlock);  // 블로킹
pthread_rwlock_trywrlock(&rwlock);  // 논블로킹
pthread_rwlock_timedwrlock(&rwlock, &deadline);  // 시간 제한

/* 읽기/쓰기 락 해제 */
pthread_rwlock_unlock(&rwlock);

/* 정리 */
pthread_rwlock_destroy(&rwlock);
```

### 5.4. 예제
```c
pthread_rwlock_wrlock(&rwlock);
/* 쓰기 작업 */
pthread_rwlock_unlock(&rwlock);
```

```c
pthread_rwlock_rdlock(&rwlock);
/* 읽기 작업 */
pthread_rwlock_unlock(&rwlock);
```

**기억할 차이**
- Mutex는 읽기끼리도 배타적으로 만들지만, 읽기/쓰기 락은 읽기끼리의 동시 접근을 허용한다.
- 다만 락 관리 비용이 있으므로 실제 성능이 항상 더 좋은 것은 아니다.

## 6. 세마포어: `sem_*`
> 허가증 개수로 접근 제어하기

### 6.1. 기본 개념
**세마포어 (Semaphore)**
- **사용 가능한 허가 수를 나타내는 정수 값**과 대기 관리 기능을 가진 동기화 도구다.

- **초기값 1**: 한 번에 하나의 허가를 제공하는 용도로 사용 가능
	- **이진 세마포어** (Binary Semaphore): 0 또는 1 *-> 상호 배제*
- **초기값 N**: N개의 자원에 대한 접근을 제한하는 용도로 사용 가능
	- **계수 세마포어** (Counting Semaphore): 0 ~ N *-> 자원 관리*
- **초기값 0**: 다른 스레드의 `post`를 기다리는 사건 알림 용도로 사용 가능

### 6.2. 선언과 초기화
```c
#include <semaphore.h>

/* 명명 세마포어 */
sem_t sem;
sem_init(&sem, 0, 3);  // 같은 프로세스의 스레드끼리 공유, 초기값 3

// 사용 종료 후, 대기자 등 사용 중인 스레드가 없을 때
sem_destroy(&sem);
```

```c
/* 익명 세마포어 */
sem_open(name, oflag);

sem_close();
```

| 인자 | 의미 |
|---|---|
| `sem` | 초기화할 세마포어의 주소 |
| `pshared == 0` | 같은 프로세스 내 스레드 간 공유 |
| `pshared != 0` | 프로세스 간 공유 |
| `value` | 초기 허가 수 |

> **보충**
> 프로세스 간 공유는 `pshared`만 바꾼다고 되는 것이 아니다.
> 세마포어 객체 자체가 공유 메모리처럼 참여 프로세스들이 함께 접근할 수 있는 영역에 있어야 한다.
> [Linux `sem_init` 문서](https://man7.org/linux/man-pages/man3/sem_init.3.html)

#### 이름 없는 세마포어와 이름 있는 세마포어
| 구분     | 이름 없음(Unnamed)               | 이름 있음(Named)           |
| ------ | ---------------------------- | ---------------------- |
| 접근 방식  | 메모리의 `sem_t` 객체 접근           | 같은 이름으로 열기             |
| 생성·초기화 | `sem_init()`                 | `sem_open()`           |
| 이름 제거  | 해당 없음                        | `sem_unlink()`         |
| 주요 활용  | 스레드 간, 공유 메모리를 통한 프로세스 간 동기화 | 서로 독립적인 프로세스들도 이름으로 연결 |

이름 없는 세마포어를 관련 프로세스 중심으로 설명한다.
핵심은 혈연 관계 자체보다 **같은 세마포어 메모리에 접근할 수 있는가**이다.

### 6.3. 주요 함수
```c
/* 대기 연산: 카운터 1 감소 */
sem_wait(&sem);  // 값이 양수면 1 감소하고 진행, 0이면 대기
sem_trywait(&sem);  // 값이 있으면 획득, 없으면 기다리지 않고 실패

/* 시그널 연산: 카운터 1 증가 */
sem_post(&sem);  // 값을 하나 제공해 대기자가 진행할 수 있게 함
```

예를 들어 자원이 3개라면 `sem_init(&sem, 0, 3)`으로 시작한다.
세 스레드가 허가를 얻은 뒤에는 네 번째 스레드가 기다린다.
누군가 `post`하면 다시 한 스레드가 진행할 수 있다.

**Mutex와의 차이**
- Mutex는 잠근 스레드가 해제하는 소유권 규칙이 있다.
- 세마포어는 한 스레드가 `wait`하고 다른 스레드가 `post`할 수 있어, 스레드 간 알림에도 사용할 수 있다.

### 6.4. 조건 변수와 비교
| 비교 항목        | 조건 변수                    | 세마포어                           |
| ------------ | ------------------------ | ------------------------------ |
| 대기자가 없을 때 알림 | `signal`이 저장되지 않음        | 성공한 `post`는 허가 수로 남을 수 있음      |
| 상태 표현        | 조건은 별도 공유 변수에 저장         | 허가 수를 세마포어가 관리                 |
| Mutex와의 관계   | 공유 조건을 보호하는 Mutex와 함께 사용 | 자원 수 조절 자체에는 별도 Mutex가 필수는 아님  |
| 공유 버퍼 보호     | 상태 확인·버퍼 접근을 Mutex로 보호   | 허가 수와 별개로 버퍼의 일관성 보호가 필요할 수 있음 |

> **정확성 보충**
> 세마포어가 항상 FIFO나 기아 방지를 보장하는 것은 아니다.
> 깨울 스레드 선택은 스케줄링 정책 등에 의존한다.
> [POSIX `sem_post` 명세](https://man7.org/linux/man-pages/man3/sem_post.3p.html)

### 6.5. 은행 잔액 보호 예제
```c
static sem_t bal_sem;
static int   balance = 0;

int initshared(int value) {
    if (sem_init(&bal_sem, 0, 1) == -1)  // 초기값 1
        return -1;
    balance = value;
    return 0;
}

int incshared(void) {
    while (sem_wait(&bal_sem) == -1) {
        if (errno != EINTR) return -1;
    }
    balance++;
    return sem_post(&bal_sem);
}
```

## 7. 생산자·소비자 문제
**구성 요소**
- **생산자**는 데이터를 공유 버퍼에 넣는다.
- 버퍼: 생산자와 소비자가 공유하는 저장 공간
- **소비자**는 버퍼에서 데이터를 꺼낸다.

**문제 상황**
1. **경쟁 상태** (Race Condition)
2. 유한 버퍼의 **버퍼 오버플로우** (Buffer Overflow): 생산자는 빈 공간이 없으면 기다려야 한다.
3. **버퍼 언더플로우** (Buffer Underflow): 소비자는 데이터가 없으면 기다려야 한다.

한 개의 Mutex만으로 경쟁 상태를 해결할 수 있지만, 버퍼 오버/언더플로우를 위한 대기 조건이나 자원 수 관리도 필요하다.

### 7.1. 원형 버퍼 예제
- `b[]`: 공유 버퍼
- `in`: 다음 데이터를 넣을 위치
- `out`: 다음 데이터를 꺼낼 위치
- `n`: 배열 크기
- `% n`: 끝에 도달하면 처음으로 돌아가는 원형 인덱스 계산

```c
// 비어 있음
in == out

// 가득 참
(in + 1) % n == out
```

이 방식은 빈 상태와 가득 찬 상태를 구분하려고 **한 칸을 비워두므로 실제 저장 용량은 `n - 1`개**다.

강의의 생산자·소비자 코드는 아래처럼 조건을 계속 확인한다.

```c
while ((in + 1) % n == out) {
    /* 생산자: 빈 공간이 생길 때까지 바쁜 대기 */
}

while (in == out) {
    /* 소비자: 데이터가 생길 때까지 바쁜 대기 */
}
```

**읽을 때 주의**
- 이 코드는 버퍼의 동작을 설명하는 의사코드다.
- 그대로 일반 C 공유 변수에 적용하면 안전한 Pthread 동기화가 완성되지 않는다.
- 반복 검사로 CPU를 쓰는 문제와 공유 상태의 데이터 레이스를 해결해야 한다.

### 7.2. 의사코드
> ==변수명 변형 출제== #3-2/UNIX시스템프로그래밍/출제
```c
/* 생산자-소비자 문제: 유한 버퍼 */
const int N = /* buffer size */;

semaphore e = N;  // 빈 슬롯 수 (empty slots)
semaphore s = 1;  // 상호 배제
semaphore n = 0;  // 아이템 수 (number of items)

void producer() {
	while (true) {
		produce();
		sem_wait(e);    // 카운터 1 감소
		sem_wait(s);
		append();
		sem_signal(s);  // 카운터 1 증가, sem_post()
		sem_signal(n);
	}
}

void consumer() {
	while (true) {
		sem_wait(n);    // 카운터 1 감소
		sem_wait(s);
		take();
		sem_signal(s);  // 카운터 1 증가, sem_post()
		sem_signal(e);
		consume();
	}
}

void main() {
	parbegin(producer, consumer);
}
```

**왜 세마포어 대기를 Mutex 획득 전에 하는가?**
- 버퍼가 가득 찬 상황에서 생산자가 Mutex부터 잠근 채 `wait(empty)`로 기다리면, 소비자는 버퍼에서 데이터를 꺼내고 빈 칸을 만들기 위해 필요한 Mutex를 얻을 수 없다. 생산자와 소비자가 서로 기다리는 교착 상태가 된다.

**역할 분담**
- 세마포어는 “지금 작업해도 되는 수량인가?”를 관리한다.
- Mutex는 “버퍼 상태를 안전하게 바꾸는가?”를 보장한다.
- 데이터 생성·소비 작업은 가능하면 버퍼 잠금 밖에서 수행한다.

### 7.3. 구현
```c
#include <semaphore.h>

#define MAX_SIZE 5
sem_t empty;  // 5
sem_t full;   // 0

void *producer(void *arg) {
    char *buf = arg;
    int   in  = 0;
    for (;;) {
        char item = getChar();
        sem_wait(&empty);
        buf[in] = item;
        in = (in + 1) % MAX_SIZE;
        sem_post(&full);
    }
}

void *consumer(void *arg) {
    char *buf = arg;
    int   out = 0;
    for (;;) {
        sem_wait(&full);
        char item = buf[out];
        out = (out + 1) % MAX_SIZE;
        sem_post(&empty);
        useChar(item);
    }
}

int main(void) {
    char buffer[MAX_SIZE];
    
    // 두 개의 계수 세마포어를 사용한다.
    sem_init(&empty, 0, MAX_SIZE);  // 스레드 간 공유, 5 시작
    sem_init(&full,  0, 0);         // 스레드 간 공유, 0 시작
    
    /* producer와 consumer 실행 */
}
```

> 여러 생산자 또는 여러 소비자를 사용하는 일반형에서는 `in`, `out`, 버퍼 변경을 보호할 mutex 또는 초기값 1의 추가 세마포어가 필요하다.
