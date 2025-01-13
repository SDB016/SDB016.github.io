---
layout: single
title: "GC (Garbage Collector)"
date: 2025-01-13
toc: true
toc_sticky: true
toc_label: "목차"
toc_icon: "list"
---
[Garbage Collector](Develop/CS/Garbage%20Collector/Garbage%20Collector.md)
# GC 란 무엇인가
GC(Garbage Collector)는 JVM의 Heap 영역에서 동적으로 할당했던 메모리 중 필요 없게된 메모리 객체를 주기적으로 제거하는 프로세스이다.
자바 뿐만 아니라 파이썬, 자바스크립트, Go 등 다른 프로그래밍 언어에도 GC가 기본적으로 내장되어 있는 경우가 많다.
- 장점
	- 개발자가 메모리 누수를 신경쓰지 않아도 되어 생산성 향상
- 단점
	- 메모리가 언제 해제되는지 정확히 알 수 없어 제어하기 힘듦
	- GC가 동작할 때 STW(Stop-the-world)[^1]하는 오버헤드가 존재

# 동작 방식
## GC 대상
특정 객체에 연결된 레퍼런스가 있다면 Reachable, 없다면 Unreachable로 구분하여 수거
레퍼런스는 Stack 또는 Method Area에 존재해 heap의 객체 주소를 참조
## 청소 단계
청소 3단계 과정
![mark-sweep-compact](/images/mark-sweep-compaction.png)
- Mark: Root space[^2]로부터 그래프 순회를 통해 연결된 객체를 찾아내어 어떤 객체를 참조하고 있는지 찾아서 마킹 단계
- Sweep: unreachable 객체들을 heap에서 제거 단계
- Compact: sweep 후 분산된 객체들을 heap의 주소로 모으는 압축 단계 (GC에 따라 하지 않는 경우도 있음)
## Heap 구조
GC 동작 과정을 알아보기 전에 heap[^3]의 구조를 살펴보자.
![heap memory](/images/heap_memory.png)
- Young Generation
	- Eden
	- Survivor 0/1
- Old Generation

> Heap 설계 배경 (Weak Generation Hypothesis)
>	- 대부분 객체는 금방 Unreachable 상태가 됨
>	- 오래된 객체에서 새로운 객첼로의 참조는 아주 적은 빈도
>  객체 대부분 **일회성**이며 **오랫동안 남아있는 경우는 드물다**.
### Young Generation
- 새롭게 생성된 객체가 할당되는 영역
- 대부분 객체가 여기서 생성되었다가 사라짐
Eden
- 새로 생성된 객체가 할당되는 영역
Survivor 0/1
- Eden에서 최소 1번의 GC에서 살아남은 객체가 보내지는 영역
- survivor 0 또는 1 둘 중 하나는 항상 비워져 있어야 함
### Old Generation
- Young 영역에서 Reachable 상태를 유지해 살아남은 객체가 복사되는 영역
- 일부 JVM 설정에 따라 큰 객체는 바로 Old 영역에 할당될 수 있음 (예: `-XX:PretenureSizeThreshold`)
- Young 영역보다 크게 할당되고 GC는 적게 발생
- Young Generation 객체가 Old Generation 객체를 참조할 경우를 효율적으로 관리하기 위해 [[Develop/CS/Garbage Collector/카드 테이블]]이 존재
## GC 동작 과정
### Minor GC
- Young Generation에서 일어나는 GC
- 크기가 작아 상대적으로 적은 시간이 걸림

Minor GC는 아래 순서로 진행된다.
1. 새로 생성된 객체가 Eden에 위치
2. Eden 공간이 꽉 차면 Minor GC 실행
3. Mark를 진행해 reachable 객체 탐색
4. Eden에서 살아남은 reachable 객체는 survivor 영역(0 또는 1)으로 이동
5. Eden에 남은 unreachable 객체의 메모리 해제 (sweep)
6. 살아남은 객체들의 age 값[^4] 1 증가
7. 이후 Eden이 다시 꽉 차서 Minor GC가 실행
8. Eden과 현재 사용중인 survivor 영역의 객체들에 대해 mark 진행
9. 현재 사용 중인 Survivor 영역의 Reachable 객체를 비어 있는 다른 Survivor 영역으로 이동
10. Sweep 진행 후 age 1 증가
### Major GC
- Old Generation에서 일어나는 GC
- Promotion으로 인해 메모리가 가득 차면 발생

Major GC는 아래 순서로 진행된다.
1. Survivor 영역의 객체의 age가 임계값에 도달
2. Old Generation으로 이동 (promotion)
3. Old Generation이 부족해지면 Major GC 실행
4. Major GC 알고리즘에 따라 Stop-the-world 발생 (CMS, G1 등에서는 일부 단계가 동시 실행)

Major GC는 공간이 큰 old generation에 대해 일어나기 때문에 오랜 시간이 걸린다. 
이때 스레드를 멈추고 mark/sweep 작업을 해야하기 때문에 stop-the-world가 발생한다.
이를 효율적으로 해결하기 위해 GC 알고리즘은 계속 발전하고 있다.
# GC 알고리즘
## Serial GC
![serailGC](/images/SerailGC.png)
- CPU 코어가 1개일 때를 위한 가장 단순한 GC
- GC 처리 스레드가 1개라 가장 stop-the-world 오래 걸림
- Minor GC에는 mark-sweep, major GC에는 mark-sweep-compact
```bash
java -XX:+UseSerialGC -jar Application.java
```
## Parallel GC
![Parallel GC](/images/parallelGC.png)
- Java 8의 기본 GC
- Minor GC를 멀티 스레드로 수행 (Major GC는 싱글 스레드)
- GC 스레드는 기본적으로 cpu 개수만큼 할당 (옵션으로 변경 가능)
```bash
java -XX:+UseParallelGC -jar Application.java
# -XX:ParallelGCThreads=N : 사용할 쓰레드의 갯수
```
## Parallel Old GC
![parallel old GC](/images/parallelOldGC.png)
- Parallel GC를 개선
- Young/Old generation 모두 멀티 스레드로 GC 수행
- Mark-Summary-Compact 방식
```bash
java -XX:+UseParallelOldGC -jar Application.java
# -XX:ParallelGCThreads=N : 사용할 쓰레드의 갯수
```
## CMS GC (Concurrent Mark Sweep)
![cms GC](/images/cmsGC.png)
- 어플리케이션의 스레드와 GC 스레드가 동시에 실행
- GC 과정이 매우 복잡
- Mark 단계에서 CPU 사용량이 높아지고, Sweep 단계에서 메모리 단편화가 발생할 가능성이 있음
- Java 9에서 deprecated, java 14에서 사용 중지
```bash
# deprecated in java9 and finally dropped in java14
java -XX:+UseConcMarkSweepGC -jar Application.java
```
### 실행 단계
1. Initial mark: GC 루트에서 reachable 객체를 찾아 마킹 (매우 짧은 Stop-the-world)
2. Concurrent mark: 이전 단계에서 찾은 객체에서 참조하고 있는 객체를 애플리케이션 실행과 동시에 마킹
3. Remark: concurrent mark에서 새로 추가도거나 참조가 끊긴 객체에 대해 다시 마킹 (매우 짧은 Stop-the-world)
4. Concurrent sweep: 더 이상 참조되지 않는 객체를 애플리케이션 실행과 동시에 제거
### 장점
- 짧은 stop-the-world
### 단점
- 다른 알고리즘 대비 많은 cpu를 사용
- Compaction 단계를 수행하지 않아 메모리 파편화 -> compaction 단계 실행시 stop-the-world 더 오래 걸림

## G1 GC (Garbage First)
[[Develop/CS/Garbage Collector/G1 GC]]
![g1gc](/images/g1GCRegion.png)
- JDK 7에서 최초 release
- Java 9+ 버전에서 기본 GC
- 4GB 이상의 heap, stop-the-world 0.5초 상황에 적합
- 기존에는 heap 영역을 young/old 영역으로 나누었으나, G1 GC는 Region이라는 개념을 새로 도입
	- 전체 heap 영역을 격자같은 region으로 분할하여 상황에 따라 메모리 영역을 동적 부여
- Garbage로 가득 찬 영역을 빠르게 회수 가능
```bash
java -XX:+UseG1GC -jar Application.java
```
## Shenandoah GC
- Java 12에 release
- 기존 CMS가 가진 단편화, G1이 가진 pause 이슈를 해결
- 
## ZGC
	- Java 15에 release
	- 대량의 메모리(8MB ~ 16TB)를 low latency로 처리하기 위한 GC
# 버전 별 GC의 차이점


[^1]: GC를 수행하기 위해 JVM이 프로그램 실행을 멈추는 현상. GC 관련 스레드 이외에 모든 스레드가 멈춤
[^2]: GC의 root space: stack (로컬 변수), method area (static 변수), native method stack (JNI 참조)
[^3]: Heap: JVM의 heap 영역은 동적 레퍼런스 데이터가 저장되며 GC 대상인 공간
[^4]: Age 값은 Survivor 영역에서 살아남은 횟수를 나타내며, object header에 기록됨. Minor GC 때마다 1씩 증가. age 값이 임계에 도달하면 promotion(old 영역으로 이동) 여부를 정함