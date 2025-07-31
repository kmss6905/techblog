---
layout: post
title: "JVM 메모리 할당 효율성 높이기: Bump pointer allocation과 TLAB"
description: "JVM은 멀티스레드 환경에서 메모리 할당 시 충돌을 방지하고 효율성을 높이기 위해 Bump pointer allocation과 TLAB(Thread-Local Allocation Buffer) 기술을 사용합니다. 이 글에서는 두 기술의 원리와 장점을 알아봅니다."
categories:
- "JVM"
tags:
- "jvm"
- "tlab"
- "bump the pointer"
- "memory allocation"
- "performance"
date: "2024-09-18 00:00:00 +0900"
toc: true
---

# JVM 메모리 할당 효율성 높이기 - Bump pointer allocation 과Thread-Local Allocation Buffer

> Java 에서 객체 생성시 JVM 의 Heap 영역에 메모리가 할당된다. 하지만 JVM 은 기본적으로 Multi-Thread 환경이기 때문에 메모리 할당시 메모리 충돌을 방지하기 위해 Bump the pointer 과 Thread-Local-Buffer(TLAB) 라는 기술을 추가하였는 데 이에 대해서 간단히 정리해보았다.

### Bump the pointer

새로운 객체를 만들면 JVM 이 Heap 영역에 새로운 객체를 위해서 메모리를 할당해야하는데, 비어있는 메모리를 찾을 때 JVM은 탐색하는 시간을 줄이기 위해 할당된 메모리 바로 뒤에 메모리를 할당하는 방법을 사용하는 데 이를 **Bump pointer allocation** 이라고 합니다.

> 대부분의 Gabage Collector 는 Gabage Collection 시 압축(Compat) 하는 과정을 통해 파편화된 메모리 공간을 하나로 모으는 작업을 합니다.
>

### **Thread-Local Allocation Buffer**

#### 등장배경
new 라는 키워드를 통해 새로운 객체를 생성했을 때를 가정해보겠습니다.  

하나의 스레드가 새로운 객체를 메모리에 할당하기 위해 비어있는 메모리 주소를 요청하게 됩니다.  이때 Bump the pointer 를 사용하고 있기 때문에 가장 최근에 할당된 메모리 공간 바로 뒤의 주소를 요청 받게 됩니다.  
문제는 JVM 은 멀티 스레드를 지원하기 때문에 여러개의 스레드가 최근에 할당된 메모리 공간을 동시에 요청하면 동시화 이슈가 발생하게 됩니다.

<p align="center">
  <img width="500" alt="스크린샷 2024-09-11 오후 8 05 07" src="/assets/images/single_thread_allocation_request.png">
<br>
  <em>그림 1 - 싱글스레드의 메모리 요청</em>   
</p>

<br>

이렇게 *그림 1* 과 같이 오직 하나의 스레드에 대해서 할당 요청하는 경우에는 메모리 크기를 요청하는 스레드가 하나이기 때문에 동기화 문제가 발생하지 않습니다.
<p align="center">
  <img width="500" alt="스크린샷 2024-09-11 오후 8 05 07" src="/assets/images/multi_thread_allocation_collision.png">
<br>
  <em>그림 2 - 멀티스레드의 메모리 요청</em>   
</p>

<br>

*그림 2* 처럼 여러개의 Thread 가 동시에 메모리를 요청하려고 하는 경우 충돌이 발생할 수 있습니다. 당연히 이럴 경우에는 Lock 과 같은 동기화 작업이 수행됩니다.
만약 10개의 스레드가 같은 메모리 공간을 요청하려고 한다면 최초 할당을 요청한 스레드가 락을 풀때까지 뒤에 9개의 스레드가 대기하게 되고 이는 Application 의 성능저하를 유발할 수 있습니다.


<p align="center">
  <img width="500" alt="스크린샷 2024-09-11 오후 8 05 07" src="/assets/images/multi_thread_memory_allocation_request.png">
<br>
  <em>그림 3 - TLAB</em>   
</p>

<br>

이러한 문제를 해결하기 위해 JVM 은 <u>TLAB(Thread-Local Allocation Buffer)</u> 이라는 기술을 도입했습니다.

<u>TLAB(Thread-Local Allocation Buffer)</u>는 *그림 3* 과 같이 각 스레드에게 Eden 영역의 일부를 독점적으로 할당하여, 해당 스레드가 객체를 생성할 때 동기화 없이 빠르게 메모리를 할당할 수 있도록 합니다.

(물론 TLAB 를 Thread 에게 최초로 할당하거나 TLAB 가 부족해서 새롭게 할당 받을 때는 동기화 이슈가 발생하는 것은 어쩔 수 없지만, 그래도 사용하지 않았을 때보다는 동기화 이슈가 많이 줄어 들어 메모리 할당에 걸리는 시간이 줄어드는 건 장점으로 작용합니다.)

### 요약

JVM 은 객체에 대한 메모리 할당 요청시 비어 있는 메모리를 찾는 시간을 줄이기 위해 최근 할당된 메모리 공간 바로 뒤의 메모리 공간을 요청하는 방식의 <u>**Bump pointer allocation**</u> 을 사용하고 있습니다.

Multi Thread 환경에서 같은 메모리 공간을 동시에 여러 스레드가 요청할 경우 동기화 이슈로 인한 병목현상이 발생하게 됩니다.
이를 방지하기 위해 Heap 에 Thread 별로 공간을 나누어 대기 현상 없이 메모리 할당을 가능하게 하는 방법인 <u>**Thread Local Allocation Buffer(TLAB)**</u> 라는 기술을 추가하였습니다.

---
### Reference
https://www.baeldung.com/java-jvm-tlab  
https://inside.java/2020/06/25/compact-forwarding/  
김한도, **『**JAVA PERFORMANCE FUNDAMENTAL**』**, 엑셈(2009), 108-109
