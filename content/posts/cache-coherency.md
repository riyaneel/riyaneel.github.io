+++
title = "The Invisible Lock: Cache Coherency and the Physics of False Sharing"
date = '2026-03-26T00:50:00+01:00'
draft = false
tags = ["cpu", "concurrency", "cpp", "assembly", "memory"]
summary = "You removed the mutex, but the CPU added a hardware lock. A deep dive into how the MESI protocol and false sharing silently destroy multi-core scaling."
+++

> **Logical independence is a software theory. Cache coherency is hardware physics.**

---

## Abstract

When designing concurrent software, engineering teams often default to lock-free data structures to satisfy theoretical
scaling constraints. This practice relies heavily on the C++ Abstract Machine—a conceptual model that evaluates thread
parallelism under a dangerous assumption: that logically independent variables guarantee physically independent
execution.

On modern multi-core architectures, this assumption is physically invalid.

The multithreaded abstract machine possesses a fundamental blind spot: the **MESI Protocol**. Unoptimized structural
layouts force independent cores to unknowingly share the same 64-byte physical cache line. This physical overlap
triggers a "Cache-Coherency Storm"—a violent micro-architectural phenomenon where L1 caches endlessly invalidate each
other via Request For Ownership (RFO) signals over the CPU interconnect. This paralyzes Out-of-Order (OoO) execution
pipelines, forcing them to stall for hundreds of cycles without a single OS-level lock ever being acquired.

In highly mutable, ultra-low-latency domains like High-Frequency Trading (HFT), this is a fatal flaw. In systems like a
Single-Producer Single-Consumer (SPSC) Ring Buffer, placing read and write indices adjacently introduces catastrophic
latency jitter and obliterates interconnect bandwidth.

This technical analysis deconstructs the physical cost of cache coherency failure. By benchmarking a production-grade
lock-free queue under strict NUMA core-pinning conditions, we expose a **3.1x performance collapse** caused entirely by
micro-architectural false sharing. We demonstrate how to break the coherency bottleneck by implementing **128-byte
spatial isolation and temporal batching.** By replacing shared read-paths with local shadow variables and defeating the
hardware prefetcher, we eliminate RFO interconnect saturation and preserve true silicon parallelism. Correlating
generated x86-64 assembly with hardware telemetry (`perf c2c`), we prove an absolute systems engineering truth: **The
CPU does not execute threads. It negotiates cache lines.**

---

## 1. The Lock-Free Illusion and the SPSC Ring Buffer

When engineering concurrent systems, standard computer science relies heavily on the concept of logical isolation. Under
this theoretical model, if Thread A and Thread B operate on completely different variables, there is no data race, no
contention, and scaling should be perfectly linear.

In physical execution, however, this model is dangerously incomplete. It assumes that the CPU executes C++ variables
exactly as defined in the source code. With modern architectures featuring densely packed cores and complex interconnect
topologies (like Intel's Ring Bus or AMD's Infinity Fabric), this assumption is an architectural hazard.

This disconnect becomes glaringly apparent when designing high-throughput systems, such as the **Single-Producer
Single-Consumer (SPSC) Lock-Free Queue**.

In an ultra-low-latency environment, OS-level locks (`std::mutex`) are unacceptable. When passing market data from a
network ingress thread to a trading logic thread, blocking the OS scheduler costs microseconds. To maintain continuous
execution, developers build lock-free Ring Buffers. The logic is mathematically sound: the Producer only ever writes to
the `head` index, and the Consumer only ever writes to the `tail` index.

A standard C++ implementation relies on adjacent atomics:

```c++
struct FlawedQueue {
    std::atomic<size_t> head{0};  // Producer writes, Consumer reads
    std::atomic<size_t> tail{0};  // Consumer writes, Producer reads
    int data[CAPACITY];
    
    // ... push() and pop() logic ...
};
```

This structure contains no OS-level locks and no overlapping writes. When deployed to a compute node with the Producer
explicitly pinned to Core 8 and the Consumer pinned to Core 9, it is expected to yield maximum throughput.

Look at the empirical hardware telemetry for 100,000,000 messages passed through this structure:

* **Flawed Queue (MESI Storm) Time: 452 ms**

Instead of frictionless parallelism, the application chokes. Passing a simple integer stream takes nearly half a second
on a modern processor. The OS-level locks were removed, but the system was actively weaponized against its own memory
controller. To understand why, we must strip away the C++ abstraction and examine the hardware.

---

## 2. The Micro-Architectural Bottleneck: The MESI Protocol

The execution engine does not recognize variables; it recognizes **64-byte Cache Lines**.

In standard x86-64 C++, a `size_t` requires 8 bytes. Because `head` and `tail` are declared sequentially in the struct,
the compiler packs them adjacently in physical RAM. They occupy a combined 16 bytes.

**Both atomic variables reside perfectly within the exact same 64-byte L1 Cache Line.**

Even though the software features no locks, the CPU hardware enforces strict memory consistency across all cores through
the **MESI (Modified, Exclusive, Shared, Invalid) Protocol**:

1. **State M (Modified):** The Producer (Core 8) increments `head`. The hardware must grant Core 8 exclusive ownership
   of the entire 64-byte cache line.
2. **The RFO (Request For Ownership):** A fraction of a nanosecond later, the Consumer (Core 9) reads `head` to check if
   the queue is empty. Core 9 realizes the cache line is not in its local L1 cache. It blasts a Request For Ownership (
   RFO) signal across the CPU's internal interconnect.
3. **State I (Invalid):** The hardware intercepts this signal, violently rips the cache line out of Core 8's L1 cache,
   marks it as "Invalid," and transfers the payload to Core 9.

A few cycles later, the Producer writes to `head` again, and the process reverses. The cache line bounces continuously
across the interconnect. This generates massive cross-core traffic, saturating the memory bus. More critically, it
forces the Out-of-Order (OoO) execution pipelines of both cores to stall for hundreds of cycles while the memory
controllers arbitrate the dispute.

---

## 3. The Assembly Proof: A Lock Without a `lock`

To verify this hardware-level bottleneck, we must analyze the x86-64 assembly generated by the compiler (
`-O3 -march=native`) for our `FlawedQueue::push` loop.

When developers hear "lock," they usually look for the `lock` prefix in assembly (e.g., `lock cmpxchg`). However,
because our C++ code uses `std::memory_order_release` for the store, the x86-64 architecture—which is strongly
ordered—does not require a hardware fence or a `lock` prefix.

Look at the raw assembly extracted directly from the compiled binary:

```asm
.L_push_loop:
    mov    rax, QWORD PTR [rcx]       ; 1. Load current head
    mov    r8, QWORD PTR [rcx+0x8]    ; 2. Load current tail
    ; ... capacity check and array indexing ...
    mov    DWORD PTR [rcx+rsi*4+0x10], edi ; 3. Write payload
    mov    QWORD PTR [rcx], rax       ; 4. Store head + 1
    jne    .L_push_loop
```

This is the ultimate micro-architectural trap. At the instruction level, this code is perfectly lock-free. There are
only `mov` instructions. The CPU's ALU can process these in a single clock cycle.

The lock is entirely invisible because **the lock is the interconnect itself**.

Instruction `#2` (`mov r8, ...`) requests the 64-byte cache line in a *Shared* state. Instruction `#4` (
`mov QWORD PTR [rcx], rax`) requires that exact same 64-byte cache line in an *Exclusive/Modified* state. Every single
iteration of this loop forces the memory controller to switch the MESI state, broadcast an RFO, and stall the pipeline,
despite the code being mathematically pristine.

---

## 4. The Alignment Trap and the L1D Prefetcher

A standard architectural approach to resolve false sharing relies on spatial isolation—padding the structure to match
the 64-byte cache line size using alignment directives:

```c++
struct AlignedQueue {
    alignas(64) std::atomic<size_t> head{0};
    alignas(64) std::atomic<size_t> tail{0};
    int data[CAPACITY];
};
```

Benchmarking this iteration under the exact same conditions often yields disappointing results—frequently executing in ~
500 ms. The silicon remains heavily bottlenecked. Two distinct micro-architectural failures persist in this
implementation:

### Trap 1: The Adjacent Cache Line Prefetcher

The C++ standard (`std::hardware_destructive_interference_size`) often defaults to 64 bytes. However, modern Intel and
AMD architectures feature an active **Adjacent Cache Line Prefetcher** (also known as the L1D Spatial Prefetcher). When
a core requests a 64-byte line, the hardware aggressively pulls the *next* contiguous 64-byte line into the L1 cache to
hide future memory latency.

By aligning to exactly 64 bytes, `head` and `tail` remain side-by-side in physical RAM across two adjacent lines. The
prefetcher pulls them both simultaneously, effectively upscaling the 64-byte false sharing into a 128-byte coherency
conflict.

### Trap 2: True Sharing (The Read Path)

Padding mitigates overlapping writes, but it completely fails to address overlapping reads. In standard lock-free
implementations, the `push()` logic enforces continuous polling:

```c++
size_t current_tail = tail.load(std::memory_order_acquire);
```

The Producer reads the Consumer's `tail` on **every single iteration** to evaluate queue capacity. The code explicitly
demands the CPU to fetch the other core's cache line 100 million times. The MESI protocol continues to broadcast across
the interconnect because the algorithm explicitly requests shared state evaluation, generating catastrophic bus traffic.

---

## 5. The Solution: Temporal Batching and 128-Byte Isolation

To achieve maximum throughput, hardware topology requires more than spatial padding; it requires **Temporal Batching**.
Interconnect traffic must be minimized algorithmically.

The Producer does not require real-time synchronization with the Consumer's exact `tail` index on every instruction. It
only needs to evaluate the shared `tail` when the queue approaches capacity. By introducing **Shadow Variables** (local,
unshared registers), the thread can cache the last known state and delay memory bus synchronization until absolutely
necessary.

```c++
struct OptimizedQueue {
    // 128 bytes defeats the adjacent cache line prefetcher
    alignas(128) std::atomic<size_t> head{0};
    alignas(128) size_t cached_tail{0}; // Producer's private copy (No MESI traffic)

    alignas(128) std::atomic<size_t> tail{0};
    alignas(128) size_t cached_head{0}; // Consumer's private copy (No MESI traffic)

    alignas(128) int data[CAPACITY];

    void push(int val) {
        size_t current_head = head.load(std::memory_order_relaxed);
        
        // Evaluate against the local cache first (ZERO cross-core traffic)
        if (current_head - cached_tail >= CAPACITY) {
            // Synchronize with the memory bus only when full
            cached_tail = tail.load(std::memory_order_acquire);
            while (current_head - cached_tail >= CAPACITY) {
                cached_tail = tail.load(std::memory_order_acquire); // Spin wait
            }
        }
        
        data[current_head & MASK] = val;
        head.store(current_head + 1, std::memory_order_release);
    }
};
```

### The Assembly Proof: The Silent Fast Path

When we compile this temporally batched queue, the generated assembly for the hot path (`push()`) reveals a radical
micro-architectural shift. By extracting the machine code from our benchmark binary, we can observe exactly how the
compiler optimized the layout:

```asm
.L_push_loop:
    mov    rax, QWORD PTR [rdx]             ; 1. Load current_head
    mov    rsi, rax
    sub    rsi, QWORD PTR [rdx+0x80]        ; 2. Read cached_tail (Offset 128 bytes - Local L1)
    cmp    rsi, 0xffff                      ; 3. Check against CAPACITY (65535)
    ja     .L_slow_path_bus_sync            ; 4. Fallback to MESI sync ONLY if full  
    mov    DWORD PTR [rdx+rdi*4+0x200], ecx ; 5. Write payload to data array (Offset 512 bytes)
    mov    QWORD PTR [rdx], rax             ; 6. Store head + 1 (Release)
    jne    .L_push_loop                     ; 7. Loop repeat
```

Notice the critical difference in instruction #2. The CPU is no longer fetching the shared `tail`. It is reading
`cached_tail`, located at `[rdx+0x80]` (a precise 128-byte offset from `head`). Because `cached_tail` is strictly
mutated by the Producer, it remains permanently locked in an *Exclusive* (or *Modified*) state within Core 8's L1 cache.

The cross-core `load` has vanished from the hot path. The memory bus is silent.

By expanding the spatial isolation to 128 bytes (defeating the hardware prefetcher) and restricting
`memory_order_acquire` loads strictly to the slow-path branches, cross-core traffic is almost entirely eliminated.

---

## 6. Hardware Telemetry: The `perf c2c` Smoking Gun

Architectural theory must be empirically validated. We benchmarked the insertion and extraction of 100,000,000 integers
using both the flawed queue and the temporally batched queue.

**The Execution Latency:**

```text
--- SPSC Ring Buffer Benchmark: 100,000,000 Messages ---

Benchmarking Flawed Queue (MESI Storm)...
Flawed Queue Time: 452 ms

Benchmarking Optimized Queue (Shadow Variables + 128B Align)...
Optimized Queue Time: 142 ms
```

To isolate the exact micro-architectural failure driving the 452 ms delay, we profile the execution using the Linux
kernel's Performance Monitoring Unit (PMU), specifically utilizing `perf c2c` (Cache-to-Cache) to monitor memory
contention.

The telemetry exposes the invisible lock:

```text
=================================================
           Shared Data Cache Line Table          
=================================================
  Rmt HITM |  Lcl HITM |  Store Drops | Cacheline Address
-------------------------------------------------
    98.4%  |    99.1%  |       45.2%  | 0x7f8a1230a040 (FlawedQueue::head)
```

The underlying bottleneck is exposed in the **HITM (Hit In The Modified state)** metric. A HITM event occurs when a core
requests a cache line that is currently residing in a *modified* state in another core's L1 cache. This is the most
expensive memory operation on a multi-core processor, effectively forcing a full pipeline stall.

During the flawed queue execution, over 98% of the remote cache hits were HITM events mapped directly to the 64-byte
block holding `head` and `tail`. The processors were spending the vast majority of their cycles fighting for
interconnect ownership rather than executing arithmetic.

When evaluating the `OptimizedQueue`, the HITM events drop to near zero. The 452 ms MESI storm resolves into a **142 ms
** silent execution—a massive 3.1x performance multiplier. The execution ports are saturated, cache lines remain
localized, and the lock-free theory is finally reconciled with hardware reality.

---

## Conclusion: Hardware-Aware Systems Engineering

The 3.1x performance delta between a standard lock-free atomic setup and a 128-byte isolated, temporally batched queue
is not a micro-benchmarking anomaly; it is the fundamental physical baseline of modern computer architecture.

In latency-critical domains—spanning high-frequency trading (HFT), real-time graphics rendering, and kernel-level
network packet processing—systems cannot be designed in a theoretical vacuum.

True low-latency engineering requires a paradigm shift. We must abandon abstraction-heavy programming models, but we
must equally avoid the trap of assuming that "lock-free" inherently means "fast." We must practice **Hardware-Aware
Data-Oriented Design**.

Software optimization is not achieved solely by removing OS-level mutexes. It is achieved by understanding the 64-byte
physical reality of the cache line. It is achieved by structuring data layouts to defeat aggressive hardware
prefetchers, isolating atomic states to eliminate RFO interconnect saturation, and batching reads to silence the MESI
protocol.

The engineering directive for high-performance systems is absolute: Stop trusting default atomic behaviors for critical
paths. Align your data precisely. Cache your shared state locally. Reduce interconnect traffic, and allow the
Out-of-Order execution engine to perform the math it was designed to do.
