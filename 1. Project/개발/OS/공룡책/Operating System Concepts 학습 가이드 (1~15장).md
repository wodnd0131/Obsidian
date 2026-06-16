

**사용법**: 각 장마다 "주요 내용"과 "연관 관심사"를 먼저 가볍게 읽고 큰 그림을 잡은 뒤, 키워드 옆 빈 줄에 본인 언어로 직접 정리해 넣으세요. 가능하면 평소 다루는 Spring Boot / JVM / DB / Docker 경험과 1:1로 매칭해서 적으면 오래 남습니다.

---

## Part 1. 개관 — OS의 큰 그림 잡기

이 두 챕터는 이후 모든 내용의 용어와 전제를 깔아주는 파트입니다. 디테일을 외우기보다 "왜 이런 구조가 필요한가"에 집중하세요.

### 1장. 서론 (Introduction)

**주요 내용**: 운영체제가 무엇이고 어떤 역할을 하는지(자원 관리자, 사용자-하드웨어 중재자), 컴퓨터 시스템의 구성요소, 인터럽트 기반 동작 방식, 듀얼 모드(사용자/커널), 멀티프로세서·클러스터 시스템 개요.

**연관 관심사**: 이후 모든 장의 전제 / 2장과 한 묶음

**정리할 키워드**

- 커널(Kernel) vs 운영체제(OS) :
- 시스템 콜(System Call) :
- 인터럽트(Interrupt) / 트랩(Trap) :
- 듀얼 모드(사용자 모드 / 커널 모드) :
- 멀티프로그래밍 vs 멀티태스킹 :
- 대칭/비대칭 멀티프로세싱(SMP) :
- DMA(Direct Memory Access) :

---

### 2장. 운영체제 구조 (Operating-System Structures)

**주요 내용**: OS가 제공하는 서비스 종류, 사용자-OS 인터페이스(CLI/GUI), 시스템 콜이 동작하는 방식과 파라미터 전달, OS 설계 구조(모놀리식/계층형/마이크로커널/모듈형), 부팅 과정.

**연관 관심사**: 1장과 함께 "개관" / 3장(프로세스)에서 시스템 콜이 실제 어떻게 쓰이는지로 연결

**정리할 키워드**

- 시스템 콜 인터페이스(API) :
- 모놀리식 커널 vs 마이크로커널 :
- 계층형(Layered) / 모듈형(Modular) 구조 :
- 부트스트랩(Bootstrap) / 부트로더 :
- 링커(Linker) & 로더(Loader) :

---

## Part 2. 프로세스 관리 — 실행 단위와 스케줄링

3~5장은 "OS가 무엇을 실행 단위로 보고, 어떻게 CPU 시간을 나눠주는가"를 다룹니다. Tomcat 스레드풀, Spring 비동기 처리와 가장 먼저 연결지을 수 있는 구간입니다.

### 3장. 프로세스 (Processes)

**주요 내용**: 프로세스의 개념과 상태(생성-준비-실행-대기-종료), PCB, 프로세스 스케줄링 큐 구조, 컨텍스트 스위치, 프로세스 생성/종료(fork-exec-wait), 프로세스 간 통신(IPC: 공유메모리 vs 메시지패싱).

**연관 관심사**: 4장(스레드), 5장(스케줄링)과 한 묶음 / OS 프로세스 모델 ↔ JVM 프로세스 이해

**정리할 키워드**

- PCB(Process Control Block) :
- 프로세스 상태 다이어그램 :
- 컨텍스트 스위치(Context Switch) :
- fork() / exec() / wait() :
- 좀비(Zombie) / 고아(Orphan) 프로세스 :
- IPC: 공유 메모리 vs 메시지 패싱 :
- 파이프(Pipe) / 소켓(Socket) / RPC :

---

### 4장. 다중 스레드 프로그래밍 (Threads & Concurrency)

**주요 내용**: 스레드 개념, 멀티스레딩 모델(다대일/일대일/다대다), 멀티코어 프로그래밍 이슈, 스레드 라이브러리(Pthreads, Java Thread), 암묵적 스레딩(스레드풀, fork-join), 스레드 취소·TLS 같은 이슈.

**연관 관심사**: 3장 프로세스의 하위 개념 / 6~8장 동기화와 직결 / Java Thread, Tomcat 스레드풀과 거의 1:1 대응

**정리할 키워드**

- 사용자 스레드 vs 커널 스레드 :
- 멀티스레딩 모델(다대일 / 일대일 / 다대다) :
- 스레드풀(Thread Pool) :
- TLS(Thread-Local Storage) :
- 동시성(Concurrency) vs 병렬성(Parallelism) :
- 스레드 취소(Cancellation) :

---

### 5장. CPU 스케줄링 (CPU Scheduling)

**주요 내용**: 스케줄링 기준(이용률·처리량·응답시간 등), 주요 알고리즘(FCFS, SJF, 우선순위, RR, MLFQ), 스레드/멀티프로세서 스케줄링(부하분산, 프로세서 친화성), 실시간 스케줄링 개념.

**연관 관심사**: 3,4장과 함께 "프로세스 관리" 파트 마무리 / 6장에서 우선순위 역전 문제로 다시 연결됨

**정리할 키워드**

- 선점형(Preemptive) vs 비선점형(Non-preemptive) :
- FCFS / SJF / Round Robin / 우선순위 스케줄링 :
- 다단계 피드백 큐(MLFQ) :
- 기아상태(Starvation)와 에이징(Aging) :
- 프로세서 친화성(Affinity) / 부하분산(Load Balancing) :

---

## Part 3. 프로세스 동기화 — 실무 ROI가 가장 높은 구간

6~8장은 멀티스레드 환경에서 데이터를 안전하게 다루는 방법을 다룹니다. Java의 synchronized / Lock / Semaphore와 계속 비교하면서 보면 가장 효과가 큽니다.

### 6장. 동기화 도구 (Synchronization Tools)

**주요 내용**: 임계구역 문제 정의, 피터슨의 해법, 하드웨어 동기화 지원(Test-and-Set, CAS), 뮤텍스 락, 세마포어, 모니터, 라이브니스(데드락/기아상태 같은 동기화 실패 상황) 평가.

**연관 관심사**: 4장 스레드의 직접 후속 / 7,8장과 한 묶음 / Java synchronized·Lock·Semaphore와 직결

**정리할 키워드**

- 임계구역(Critical Section) / 상호배제(Mutual Exclusion) :
- 피터슨의 해법(Peterson's Solution) :
- CAS(Compare-And-Swap) :
- 뮤텍스 락(Mutex Lock) :
- 세마포어(Semaphore) — 카운팅/바이너리 :
- 모니터(Monitor) / 조건 변수(Condition Variable) :
- 라이브니스(Liveness) :

---

### 7장. 동기화 예제 (Synchronization Examples)

**주요 내용**: 고전적 동기화 문제(생산자-소비자/유한버퍼, 읽기-쓰기 문제, 식사하는 철학자 문제)와 이를 실제 OS·언어(Windows, Linux, Pthreads, Java)에서 어떻게 구현하는지.

**연관 관심사**: 6장의 응용/실전 예제 / 동시성 파트 마무리

**정리할 키워드**

- 생산자-소비자 문제(Bounded-Buffer) :
- 읽기-쓰기 문제(Readers-Writers Problem) :
- 식사하는 철학자 문제(Dining-Philosophers) :
- 언어/OS별 동기화 API 비교(Java vs Pthreads) :

---

### 8장. 교착상태 (Deadlocks)

**주요 내용**: 교착상태 발생 4대 필요조건, 자원할당그래프, 예방(Prevention)/회피(Avoidance, 은행원 알고리즘)/탐지(Detection)/복구(Recovery) 전략 비교.

**연관 관심사**: 6,7장 동기화 파트의 결론부 / DB 트랜잭션 데드락과 직결되는 구간

**정리할 키워드**

- 교착상태 4대 필요조건(상호배제/점유대기/비선점/순환대기) :
- 자원할당그래프(Resource-Allocation Graph) :
- 은행원 알고리즘(Banker's Algorithm) :
- 교착상태 탐지(Detection) vs 복구(Recovery) :
- 교착상태 vs 기아상태 차이 :

---

## Part 4. 메모리 관리 — JVM 힙/GC 이해의 기반

9~10장은 프로그램이 메모리를 어떻게 할당받고 쓰는지를 다룹니다. JVM 힙 메모리, GC, OOM 이슈를 OS 관점에서 다시 보게 해주는 구간입니다.

### 9장. 메인 메모리 (Main Memory)

**주요 내용**: 주소 바인딩(컴파일/적재/실행 시간), 논리주소 vs 물리주소, 동적 적재/링킹, 스와핑, 연속 메모리 할당 방식(최초적합/최적적합/최악적합)과 단편화, 페이징·세그멘테이션 기초.

**연관 관심사**: 10장 가상메모리와 묶여 "메모리관리" 파트 / JVM 힙 구조 이해의 기반

**정리할 키워드**

- 논리주소(Logical) vs 물리주소(Physical Address) :
- MMU(Memory Management Unit) :
- 동적 링킹(Dynamic Linking) / 동적 적재(Dynamic Loading) :
- 스와핑(Swapping) :
- 외부 단편화 vs 내부 단편화 :
- 페이징(Paging) / 페이지 테이블(Page Table) :
- 세그멘테이션(Segmentation) :

---

### 10장. 가상 메모리 (Virtual Memory)

**주요 내용**: 요구 페이징(Demand Paging)과 페이지 폴트 처리 흐름, 페이지 교체 알고리즘(FIFO, Optimal, LRU, Clock 등), 프레임 할당 전략, 스래싱(Thrashing), 메모리 매핑 파일.

**연관 관심사**: 9장과 묶여 메모리관리 파트 마무리 / GC·OOM 이슈를 이해하는 핵심 배경지식

**정리할 키워드**

- 요구 페이징(Demand Paging) :
- 페이지 폴트(Page Fault) 처리 흐름 :
- 페이지 교체 알고리즘(LRU / FIFO / Optimal / Clock) :
- 스래싱(Thrashing) :
- 작업 집합(Working Set) :
- 메모리 매핑(mmap) :

---

## Part 5. 저장장치 관리 — DB 내부 동작과 연결되는 구간

11~15장은 디스크 I/O와 파일 시스템을 다룹니다. MySQL 같은 DB가 내부적으로 디스크에 어떻게 쓰고 읽는지(WAL, 저널링, B+Tree 등)를 이해하는 데 직접 도움이 되는 파트입니다.

### 11장. 대용량 저장장치 구조 (Mass-Storage Structure)

**주요 내용**: HDD/SSD 구조 차이, 디스크 스케줄링 알고리즘(FCFS, SSTF, SCAN, C-SCAN), RAID 레벨 개념, 저장장치 부착 방식(호스트 부착, 네트워크 부착, SAN).

**연관 관심사**: 12장 I/O 시스템과 묶여 "저장장치" 파트 / DB 디스크 I/O 성능 이해와 연결

**정리할 키워드**

- HDD vs SSD 구조적 차이 :
- 디스크 스케줄링(FCFS / SSTF / SCAN / C-SCAN) :
- RAID 레벨(0, 1, 5 등) 핵심 개념만 :
- SAN(Storage Area Network) vs NAS :

---

### 12장. I/O 시스템 (I/O Systems)

**주요 내용**: I/O 하드웨어(포트, 컨트롤러, 인터럽트, DMA), 커널 I/O 서브시스템(버퍼링/캐싱/스풀링), 동기 I/O vs 비동기 I/O, 디바이스 드라이버 구조.

**연관 관심사**: 11장과 묶여 "저장장치 관리" 파트 / 비동기 I/O 개념은 Spring WebFlux·NIO 이해와 연결

**정리할 키워드**

- 폴링(Polling) vs 인터럽트 기반 I/O :
- DMA(Direct Memory Access) 동작 흐름 :
- 동기 I/O vs 비동기 I/O :
- 버퍼링(Buffering) / 캐싱(Caching) / 스풀링(Spooling) :
- 디바이스 드라이버(Device Driver) :

---

### 13장. 파일 시스템 인터페이스 (File-System Interface)

**주요 내용**: 파일의 속성·연산·구조, 파일 접근 방법(순차/직접), 디렉토리 구조(트리/그래프), 파일 시스템 마운팅, 파일 공유와 보호(권한 모델).

**연관 관심사**: 14,15장과 묶여 "파일 시스템" 파트 / 유저 입장에서 보는 파일 시스템

**정리할 키워드**

- 파일 속성(메타데이터) :
- 순차 접근 vs 직접 접근(Direct Access) :
- 디렉토리 구조(트리형 / 비순환 그래프형) :
- 마운트(Mount) 개념 :
- 파일 권한 모델(rwx 등) :

---

### 14장. 파일 시스템 구현 (File-System Implementation)

**주요 내용**: 파일 시스템 구조(부트블록, 슈퍼블록 등), 파일 할당 방법(연속/연결/색인 할당), 자유공간 관리(비트맵, 연결리스트), 복구 메커니즘(저널링/로그 기반), NFS 개요.

**연관 관심사**: 13장의 인터페이스가 실제로 어떻게 구현되는지 / DB 스토리지 엔진의 파일 관리 방식과 직접 비교 가능

**정리할 키워드**

- 연속 할당 vs 연결 할당 vs 색인 할당(Indexed Allocation) :
- 자유공간 관리(비트맵 / 연결리스트) :
- 저널링(Journaling) 파일 시스템 :
- 크래시 컨시스턴시(Crash Consistency) :
- NFS(Network File System) 개요 :

---

### 15장. 파일 시스템 내부 (File-System Internals)

**주요 내용**: 실제 파일 시스템(ext4 등) 설계 사례, 저널링 디테일, 데이터 무결성 보장 기법, 파일 시스템 성능 최적화 기법.

**연관 관심사**: 13,14장의 결론부 / B+Tree·WAL 같은 DB 내부 구조 학습으로 바로 연결되는 마지막 다리

**정리할 키워드**

- ext4 등 실제 파일시스템 설계 특징 :
- 저널링 모드별 차이(메타데이터 vs 전체 저널링) :
- 데이터 무결성(체크섬 등) 보장 기법 :
- 파일 시스템 성능 최적화(지연 할당 등) :

---

## 다음 단계 (16장 이후, 선택)

- 16~17장(보안/보호): 접근 제어 모델 — 평소 다루는 JWT 인증·인가 설계와 비교해보면 흥미로운 구간
- 18~19장(가상머신, 분산시스템/네트워크): Docker/AWS 경험과 연결 가능, 관심 있을 때만
- 20~22장(Linux/Windows 사례연구): 필요할 때 레퍼런스로 찾아보는 정도로 충분