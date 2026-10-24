---
title: "ThreadsafeQueueLib: Learning Material for Juniors"
date: 2026-01-19
author: "Toshit Jain"
words_per_minute: 50
read_time: true
tags:
  - C++
  - Multithreading
  - Lock-Free
  - Data Structures
  - Systems
---

## Project Overview

**ThreadsafeQueueLib** is a project developed under the Coding Club, IIT Guwahati, with the aim of designing and implementing a family of **thread-safe, lock-free, and wait-free queue data structures** in modern C++. The project is motivated by real-world concurrent systems where classical standard library containers such as `std::queue` fail under contention due to race conditions and lack of atomicity guarantees.

This document outlines the motivation, background, concurrency issues in existing queues, and the design goals of the ThreadsafeQueueLib project.

This is the [Github Repository](https://github.com/CPP-CodingClubIITG/ThreadsafeQueueLib)

---

## Introduction

With the latest advancements in the C++ language and its standard library, there is now substantial scope for developing efficient systems in the domains of **multithreading and parallel processing**. However, many classical containers in the C++ Standard Library—most notably `std::queue`—were **not designed for true concurrent usage**.

As a result, these containers often **break under concurrent access**, leading to race conditions and undefined behavior. Among these issues, **data races** are particularly dangerous and form the primary focus of this project.

In the context of queues, there are typically two participants:

- **Producers**, which generate data and push it into the queue  
- **Consumers**, which retrieve and process that data  

This producer–consumer abstraction appears across numerous domains such as:

- Airport queues  
- Operating system process schedulers  
- High-frequency trading (HFT) systems  

HFT systems, in particular, served as a major inspiration for this project.

The objective is to design and implement a family of **lock-free and wait-free queues** that can safely support **multiple producers and multiple consumers** operating concurrently on a shared structure, while avoiding data races and ensuring correctness.

---

## Race Conditions and Data Races

### Race Conditions

Consider a real-world example: buying movie tickets at a cinema with multiple cashiers. If two customers attempt to book the last few seats simultaneously, the final outcome depends on **who completes the transaction first**. This is a classic **race condition**—the result depends on the relative ordering of independent operations.

In concurrent programming, a race condition refers to any scenario where program behavior depends on the **interleaving of operations across multiple threads**.

### Data Races

The C++ Standard defines a more severe form of race condition known as a **data race**.

A **data race** occurs when:
- Two or more threads access the same memory location concurrently, and
- At least one of those accesses is a write, and
- There is no proper synchronization

A data race results in **undefined behavior**, making the program fundamentally unsafe and unpredictable.

In queue-based systems, data races typically occur when:
- Multiple producers modify the queue tail concurrently
- A consumer reads an element while a producer is still writing it
- Internal pointers or indices are accessed without synchronization

Even a single unsynchronized read–write or write–write overlap can corrupt the logical structure of the queue. This explains why naïve implementations of `std::queue` or simple linked-list queues cannot be safely shared across threads.

---

## Race Conditions Specific to `std::queue`

The standard `std::queue` implementation suffers from multiple race conditions in concurrent environments because its interface provides **no atomicity guarantees**.

### Race Between `empty()` and `front()`

A common consumer pattern is:

```cpp
if (!q.empty()) {
    int value = q.front();
    q.pop();
    do_something(value);
}
```
In single-threaded code, this construct is perfectly safe: calling `front()` on an empty queue is undefined behavior, and the preceding `empty()` check ensures that this does not happen.

However, when multiple consumers operate on the same shared queue, this becomes a **classic race condition**. Another thread may call `pop()` in the small window between `empty()` and `front()`, removing the last element and causing a second thread to read from an empty queue. Even protecting the queue with a `std::mutex` does not help, because these operations are not atomic and must be performed as a single indivisible step.

---

### Race Between `front()` and `pop()`

A second race occurs when two consumer threads execute the following pattern concurrently.

If both threads observe the same element at the front of the queue (because nothing modifies the queue between their calls to `front()`), and both subsequently invoke `pop()`, then the following issues arise:

- One item is **processed twice**
- Another item is **discarded without ever being read**

This violates the fundamental FIFO invariant of queues and clearly demonstrates why naïve concurrent use of `std::queue` is unsafe. These cases motivate the need for properly designed **threadsafe queues** which guarantee atomicity and correctness under contention.

---

## Targets of the Project

Having identified the limitations of `std::queue` in concurrent settings, our goal is to design a new data structure — the **ThreadsafeQueue** — that eliminates these issues while providing a flexible and performant interface for both developers and high-throughput systems.

Before diving into implementation details, we outline the features and design goals from a user’s point of view.

---

## Bounded and Unbounded Queues

A robust concurrent queue should support both bounded and unbounded modes.

### Bounded Mode

A bounded queue restricts the maximum number of elements it can hold. This is typically implemented using a **circular buffer**, which offers extremely fast index arithmetic and predictable memory usage.

Bounded queues are commonly used in:

- Real-time systems
- Embedded applications
- Thread pools
- High-frequency trading (HFT) pipelines

In such systems, memory must remain under strict control.

### Unbounded Mode

An unbounded queue grows as needed, often backed by a linked list or a dynamically resizing container such as `std::queue`. This mode is more flexible and useful when throughput is high and memory is abundant.

The user should be able to specify, at construction time:

- Whether the queue is bounded or unbounded
- If bounded, the maximum capacity

This configurability ensures that the ThreadsafeQueue can be deployed across a wide range of environments.

---

## SPSC, MPSC, and MPMC Support

Different applications require different concurrency models. A well-designed concurrent queue must support all three primary configurations:

### SPSC — Single Producer, Single Consumer

This is the simplest model and allows aggressive optimizations such as:

- Eliminating expensive memory fences
- Using cache-friendly ring buffers

### MPSC — Multiple Producers, Single Consumer

This model is common in:

- Logging systems
- Event queues

It requires safe atomic coordination among multiple producer threads.

### MPMC — Multiple Producers, Multiple Consumers

This is the most general and complex case. Ensuring correctness under high contention requires careful handling of:

- ABA problems
- Atomic pointer manipulation
- Memory-ordering guarantees

Our goal is to provide a **unified interface** that works across all these models while maintaining correctness and high performance.

---

## Blocking vs Lock-Free Implementations

The ThreadsafeQueue should allow users to choose between blocking and non-blocking designs.

### Blocking Mode

Implemented using `std::mutex` and `std::condition_variable`, blocking queues are easier to implement and reason about. However, they may suffer from:

- Context-switch overhead
- Potential priority inversion
- Poor scalability under contention

### Lock-Free (Non-Blocking) Mode

Lock-free queues provide progress guarantees such as **lock-freedom** or **wait-freedom**. These algorithms rely on:

- Atomic operations
- Memory-ordering semantics
- Careful ABA prevention strategies

Lock-free queues excel in high-performance systems and avoid blocking delays. Due to their superior scalability and suitability for multicore environments, this project primarily focuses on **lock-free implementations** with a **bounded implementation** as well for the sake of learning.

---

## Junior Mentoring stuff

**Since I was also teaching C++ concurrency along with the project to my juniors in Coding Club, IIT Guwahati; I thought of adding all my video lectures and slides below. Feel free to watch :)

| Date | Time | Session Name | PDF Link | Video Link |
|-----|---------|-------------|
| 22nd January 2026 | 9 PM to 10 PM | Concurrency, std::thread and more | [Link](https://drive.google.com/file/d/11gFZUBynv-fPv76cTk4gVV4K1qPlXqie/view?usp=sharing) | - |
| 24th January 2026 | 3 PM to 4:30 PM | Shared data, race conditions and std::mutex | [Link](https://drive.google.com/file/d/1Z2fe-kMtei0qSuh0_Cl4pnsMBKS3FWSG/view?usp=sharing) | [Link](https://drive.google.com/file/d/1aWfwzUj9FXg3cLgam6bMdRSHNm6anTr_/view?usp=sharing) |
| 28th January 2025 | 10.05 PM to 11.30 PM | Synchronization and Condition variables | [Link](https://drive.google.com/file/d/1UfJikRmcpD7pZXdVaVLGIEW_3_KSDoPz/view?usp=sharing) | [Link](https://drive.google.com/file/d/1iWKUjQOzADzqEfxXT3rEqYes8lwHwSBA/view?usp=sharing) |
| Till 4th Feb 2026 | - | CppCon 2017 "C++ atomics, from basic to advanced. What do they really do?" | - | [Link](https://www.youtube.com/watch?v=ZQFzMfHIxng)
| 21st Feb 2026 | 3.30 PM to 5.00 PM | C++ Memory Model and std::atomic | [Link](https://drive.google.com/file/d/1DXF0yfzV6Y7xqhvWU7t_eUg8v16Jl3Ji/view?usp=sharing) | [Link](https://drive.google.com/file/d/1N_qSr-fl-aBynR5yfkntmMPyqeRC_8Y2/view?usp=sharing) |

---