---
layout: default
title: "CH12) I/O 모델"
permalink: /OS/ch12-io-model/
---
# I/O 모델

## I/O가 병목이 되는 이유

CPU는 1나노초 단위로 연산하지만, 디스크에서 파일을 읽으면 수 밀리초가 걸린다. 네트워크로 데이터를 받으면 수십에서 수백 밀리초다. CPU 입장에서 I/O는 수백만 클록을 그냥 기다리는 시간이다.

이 대기 시간을 어떻게 처리하느냐가 서버 성능을 갈라놓는다. 같은 하드웨어에서 동시 접속 100명을 처리하는 서버와 10,000명을 처리하는 서버의 차이는 여기서 나온다.

I/O 모델은 두 개의 독립적인 축으로 정리된다.

- 기다리는 동안 스레드를 멈출 것인가 (Blocking vs Non-blocking)
- 결과를 누가 처리할 것인가 (Sync vs Async)

<br><br>

---

<br><br>

## Blocking vs Non-blocking

이 구분은 I/O 요청을 했을 때 스레드가 어떻게 되는지에 관한 것이다.

Blocking I/O는 결과가 나올 때까지 스레드를 Waiting 상태로 만든다. CPU 점유 없이 잠든다.

```python
data = socket.read()   # 데이터가 올 때까지 스레드 멈춤
process(data)          # 데이터가 온 후에만 실행
```

Non-blocking I/O는 데이터가 없으면 즉시 반환한다. 스레드는 계속 실행된다.

```python
data = socket.read()   # 데이터 없으면 즉시 None 반환
if data:
    process(data)
# 데이터가 없어도 이 줄이 바로 실행됨
```

차이는 단순하다. Blocking은 데이터가 올 때까지 스레드가 잠든다. Non-blocking은 스레드가 잠들지 않는다.

<br><br>

---

<br><br>

## Sync vs Async

이 구분은 I/O 완료 후 결과를 누가 가져다 처리하는지에 관한 것이다.

Synchronous는 내가 직접 결과를 받아서 처리한다.

```python
data = socket.read()   # 직접 결과를 받아서
process(data)          # 직접 처리
```

Asynchronous는 OS(또는 런타임)가 완료 시 내가 등록한 콜백을 호출해준다.

```python
socket.read_async(callback=process)  # "끝나면 process 불러줘"
# 여기서 기다리지 않고 다음 코드 실행
do_other_things()
# ... 나중에 I/O 완료되면 OS가 process() 호출
```

Blocking/Non-blocking이 "기다리냐"의 문제라면, Sync/Async는 "누가 결과를 챙기냐"의 문제다.

<br><br>

---

<br><br>

## 4가지 조합

두 축을 조합하면 네 가지 모델이 나온다.

<iframe src="/DEV_LOG/OS/assets/demo_io_model.html" width="100%" height="580" frameborder="0" style="border-radius:10px;border:1px solid #334155;display:block;" onload="this.style.height=(this.contentDocument||this.contentWindow.document).documentElement.scrollHeight+'px'"></iframe>

실제로 의미 있게 사용되는 조합은 두 가지다.

Sync + Blocking은 가장 직관적인 모델이다. 요청하고 결과 올 때까지 기다렸다가 처리한다. 코드가 단순하고 순서가 명확하다. 단일 스레드에서 순차 처리할 때 적합하다.

Async + Non-blocking은 고성능 서버의 기반이다. 요청 후 기다리지 않고 다른 일을 한다. OS가 완료 시 콜백을 호출한다. Node.js와 Nginx가 이 모델로 동작한다.

나머지 두 조합은 이론적으로 가능하지만 실용적이지 않다.

Sync + Non-blocking은 결과가 생길 때까지 직접 루프를 돌며 확인해야 한다. CPU가 계속 돌아가는 busy-wait 상태가 된다. 스레드는 자지 않지만 CPU를 의미 없이 낭비한다.

Async + Blocking은 개념상 모순에 가깝다. 콜백을 등록해놓고 어차피 결과를 기다리며 멈춰있으면 Async의 이점이 없다.

<br><br>

---

<br><br>

## I/O 멀티플렉싱

### 문제: 동시 접속 10,000명

서버가 소켓 10,000개를 동시에 처리해야 한다고 하자.

단순한 접근은 소켓마다 스레드를 하나씩 만드는 것이다. 스레드 10,000개가 필요하다. 스레드 하나가 약 1MB 스택을 쓰면 메모리만 10GB다. 컨텍스트 스위칭 오버헤드도 폭발한다.

Non-blocking으로 루프를 돌리는 것도 문제다. 10,000개 소켓을 매번 하나씩 확인하며 루프를 돌면 CPU가 99%의 시간을 "아직 데이터 없네" 확인에 쓴다.

### 해결: OS에게 감시를 위탁

I/O 멀티플렉싱은 "어떤 소켓이 준비됐는지"를 OS에게 물어보는 방식이다. 준비된 소켓이 없으면 스레드는 Waiting 상태로 자고, 준비되면 깨어나서 처리한다. 스레드 1개로 소켓 수만 개를 처리할 수 있다.

```python
ready = select([sock1, sock2, ..., sock10000])  # OS에게 감시 위탁
for sock in ready:         # 준비된 소켓만 처리
    data = sock.read()
    process(data)
```

### select / poll / epoll

세 가지 API가 있고 성능이 다르다.

select는 가장 오래된 방식이다. 감시할 소켓 목록을 매번 커널에 넘기고, 커널이 전부 순회해 준비된 것을 찾는다. O(n) 탐색이고 FD 1024개 제한이 있다.

poll은 FD 제한을 없앴지만 탐색 방식이 O(n)으로 같다.

epoll은 Linux에서 제공하는 방식으로 O(1)이다. 소켓이 준비되는 순간 NIC 인터럽트 → ISR → epoll 내부의 ready list에 직접 추가된다. 전체를 훑지 않는다. 소켓이 10,000개든 100,000개든 탐색 비용이 변하지 않는다.

```
select 방식:
소켓 1만개 등록 → "다 준비됐나?" O(n) 전체 스캔 → 준비된 것 반환

epoll 방식:
소켓 등록 → 잠듦
NIC 인터럽트 → ISR → ready_list에 추가 → 스레드 깨움 → 준비된 것만 반환
```

<br><br>

---

<br><br>

## DMA (Direct Memory Access)

NIC에 데이터가 도착하면 RAM으로 옮겨야 프로그램이 읽을 수 있다. CPU가 직접 옮기면 그 동안 다른 일을 할 수 없다.

DMA 컨트롤러는 이 복사를 CPU 대신 전담하는 하드웨어다.

```
CPU 직접 복사:
CPU: NIC에서 4바이트 읽기 → RAM에 쓰기 → 반복...
     (이 동안 다른 연산 불가)

DMA 사용:
CPU: "NIC 데이터를 RAM 주소 0x1000으로 복사해줘" → DMA에 지시
CPU: 다른 연산 실행
DMA: NIC → RAM 복사 진행
DMA: 완료 → 인터럽트 발생 → CPU에게 알림
CPU: 인터럽트 처리 → 데이터 사용
```

epoll과 DMA를 같이 생각해보면 흐름이 이어진다. NIC에 패킷이 도착하면 DMA가 RAM으로 복사하고, 완료 인터럽트가 발생하고, ISR에서 epoll ready list에 해당 소켓을 추가하고, 잠든 스레드가 깨어난다.

<br><br>

---

<br><br>

## Nginx vs Apache

두 서버의 성능 차이는 아키텍처의 차이다.

Apache는 요청마다 스레드(또는 프로세스)를 만드는 구조다. 동시 접속 10,000명이면 스레드 10,000개가 필요하다. 대부분의 스레드가 I/O를 기다리며 자고 있어도 메모리와 컨텍스트 스위칭 비용은 발생한다.

Nginx는 스레드 1개와 epoll 기반의 이벤트 루프로 동작한다. 소켓이 준비되면 그때 처리하고, 처리가 끝나면 다음 준비된 소켓으로 넘어간다. 컨텍스트 스위칭이 없고 메모리 사용량이 일정하다.

Apache의 문제는 스레드 수 폭발이다. select의 O(n) 탐색 문제와 혼동하기 쉽지만 별개다. Nginx가 select 대신 epoll을 쓰는 이유는 준비된 소켓 탐색을 O(1)으로 만들기 위해서다.
