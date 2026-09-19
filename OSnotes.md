
## 16. Interview Questions and Answers

**Process vs thread?** A process owns an isolated virtual address space and resources; a thread is an execution unit inside it and shares most process resources. Threads communicate cheaply but need synchronization; processes isolate failures better.

**What happens during a system call?** User code enters through a controlled instruction, the CPU switches to kernel mode, the kernel validates arguments and performs the operation, then restores user context and returns a value or error.

**Why is `volatile` insufficient for threads?** It may force observable loads/stores for compiler purposes, but it does not provide atomicity, mutual exclusion, or a happens-before relationship.

**Mutex vs semaphore?** A mutex represents ownership of a critical section; a semaphore represents a count of permits or signals and has no required owner.

**Why can deadlock happen with one mutex?** Traditional deadlock requires a cycle, so one correctly used mutex alone cannot create a multi-lock cycle; self-deadlock is possible if a non-recursive mutex is locked twice by its owner.

**What is a page fault?** A trap caused by a missing or disallowed page mapping. A valid absent page may be loaded or created; an invalid access is rejected.

**Why does a TLB improve performance?** It caches recent virtual-to-physical translations, avoiding page-table walks on TLB hits.

**What is thrashing?** Excessive paging caused by insufficient frames for active working sets, leaving little CPU time for useful execution.

**Hard link vs symbolic link?** A hard link names the same inode; a symbolic link stores a path and can cross file systems or become dangling.

**Why can `read` return less data?** Files, pipes, terminals, signals, nonblocking mode, and sockets permit partial progress. The caller must loop when the contract requires a complete buffer.

**What is copy-on-write?** Multiple address spaces share pages until a write requires a private copy, reducing `fork` cost and memory use.

**How would you investigate a slow service?** Measure latency and saturation, inspect CPU/run queue, memory faults and swap, disk and network waits, system calls, locks, and dependencies; form one hypothesis and verify it with a focused experiment.

## 17. Online-Test Formula Sheet

- $TAT = CT - AT$
- $WT = TAT - BT$
- $RT = first\ start - AT$
- CPU utilization $= busy\ time / elapsed\ time$
- Speedup $= old\ time / new\ time$
- Amdahl's law: $Speedup = 1 / ((1-p) + p/s)$
- Page offset bits $= \log_2(page\ size)$
- Number of pages $= virtual\ address\ space / page\ size$
- With TLB hit ratio $h$: $EAT = h(t_{TLB}+t_M) + (1-h)(t_{TLB}+2t_M)$, ignoring page faults.
- Disk average access is approximately seek + rotational latency + transfer + controller overhead.
- Little's Law: $L = \lambda W$ (average items = arrival rate x average time in system).

For binary conversions, $1\ KiB=2^{10}$ bytes, $1\ MiB=2^{20}$, and $1\ GiB=2^{30}$. Distinguish these from decimal KB/MB/GB.

## 18. Practice Problems

1. Given processes with arrival and burst times, draw FCFS, SJF, SRTF, and Round Robin charts and calculate average waiting and response times.
2. Explain whether two threads incrementing a shared integer is correct. Repair it with a mutex and with an atomic increment.
3. Given a resource-allocation graph, identify a deadlock cycle and propose a lock-order fix.
4. For a virtual address and page-table hierarchy, calculate page number, offset, table sizes, and physical address.
5. Count FIFO, LRU, and optimal page faults for a reference string; explain Belady's anomaly.
6. Design a bounded producer-consumer queue and state its invariants, shutdown behavior, and synchronization protocol.
7. Explain every resource leak in a program using `fork`, `pipe`, and `exec`, then identify which pipe ends must close for EOF.
8. Design a length-prefixed TCP protocol that handles partial reads, malformed lengths, timeout, and peer disconnect.
9. Diagnose high load average with low CPU utilization. Consider blocked I/O, lock contention, memory pressure, and uninterruptible sleep.
10. Compare a process pool, thread pool, and async event loop for CPU-bound and I/O-bound workloads.

### Answering coding questions safely

Check every return value, handle `EINTR`, handle partial reads/writes, close descriptors on every error path, avoid integer overflow in allocation sizes, define ownership, and state thread-safety assumptions. For concurrency, name the protected invariant and lock order.

## 19. Final Revision Checklist

- Explain user mode, kernel mode, system calls, interrupts, traps, and context switches.
- Trace `fork` + `exec` + `wait` and explain zombies, orphans, and copy-on-write.
- Compute scheduling metrics from a Gantt chart.
- Distinguish race, deadlock, livelock, starvation, and priority inversion.
- Implement or explain mutexes, semaphores, condition variables, atomics, and barriers.
- Translate virtual addresses and explain TLBs, page faults, replacement, and thrashing.
- Explain file descriptors, inodes, links, journaling, `fsync`, and disk scheduling.
- Choose IPC and networking primitives using explicit workload and failure assumptions.
- Diagnose CPU, memory, I/O, lock, and FD problems with Linux tools.
- Discuss isolation, least privilege, containers, VMs, and common resource/security failures.
- Solve at least one problem each for scheduling, synchronization, paging, IPC, and sockets under time pressure.

## 9. File Systems and Storage

### Files and descriptors

`open()` returns a per-process file descriptor (FD), a small integer referring to an open-file description containing file offset and status flags. `dup()` shares the open-file description and therefore the offset. `fork()` inherits descriptors. Close-on-exec prevents accidental FD leakage across `exec`.

```c
#include <fcntl.h>
#include <unistd.h>

int copy_file(const char *source, const char *destination) {
	int in = open(source, O_RDONLY);
	int out = open(destination, O_WRONLY | O_CREAT | O_TRUNC, 0644);
	if (in < 0 || out < 0) return -1;
	char buffer[4096];
	ssize_t count;
	while ((count = read(in, buffer, sizeof buffer)) > 0) {
		for (ssize_t written = 0; written < count;) {
			ssize_t n = write(out, buffer + written, (size_t)(count - written));
			if (n < 0) { close(in); close(out); return -1; }
			written += n;
		}
	}
	close(in);
	close(out);
	return count < 0 ? -1 : 0;
}
```

### File-system concepts

- **Inode:** metadata and block pointers; a filename is a directory entry mapping a name to an inode.
- **Hard link:** another directory entry for the same inode; normally cannot cross file systems or target directories.
- **Symbolic link:** file containing a path; can cross file systems and can dangle.
- **Journaling:** records metadata or data intent before applying changes, improving crash recovery.
- **Mount:** attaches a file-system tree at a directory.
- **Permissions:** owner/group/other read-write-execute bits. Directory execute means lookup/traversal, not running it.
- **`fsync`:** asks the kernel to flush data and metadata to stable storage; exact durability depends on the device and file-system semantics.

### Disk scheduling and RAID

Disk access includes seek, rotational latency, transfer, and controller overhead. FCFS is fair; SSTF minimizes nearby seek but can starve distant requests; SCAN sweeps like an elevator; C-SCAN gives more uniform wait.

RAID 0 stripes without redundancy, RAID 1 mirrors, RAID 5 uses distributed single parity, and RAID 6 uses double parity. RAID is not a backup: deletion and corruption can replicate to every member.

## 10. I/O and Devices

The **device driver** translates OS operations to hardware commands. **Polling** repeatedly checks status and wastes CPU; **interrupt-driven I/O** lets a device notify the CPU; **DMA** transfers buffers directly between device and memory, with the kernel configuring and completing the operation. Buffering handles speed mismatch, caching avoids repeated reads, and spooling queues work for a serial device such as a printer.

Blocking I/O waits for completion. Nonblocking I/O returns immediately. Readiness multiplexing (`select`, `poll`, `epoll`, `kqueue`) reports descriptors that can make progress; it does not itself read the data. Edge-triggered APIs require draining until `EAGAIN`.

## 11. Interprocess Communication

- **Pipe:** one-way byte stream, usually for related processes; EOF occurs when all write ends close.
- **Named pipe (FIFO):** pipe represented in the file system.
- **Message queue:** discrete messages with boundaries and optional priorities.
- **Shared memory:** fastest bulk data exchange, but requires synchronization.
- **Unix domain socket:** local bidirectional stream or datagram and credential passing.
- **Signal:** asynchronous notification with limited payload; handlers must be async-signal-safe.
- **Memory-mapped file:** shared persistent or file-backed pages.

IPC choice depends on payload size, topology, ordering, durability, security, and failure semantics. Shared memory minimizes copying but increases coordination complexity.

## 12. Networking and Sockets

TCP provides an ordered, reliable byte stream with flow control, congestion control, retransmission, and a connection handshake. UDP provides datagrams without delivery, ordering, or congestion guarantees. TCP `read`/`recv` may return fewer bytes than requested and `send` may write only part of a buffer.

```c
#include <errno.h>
#include <unistd.h>

ssize_t read_full(int fd, void *buffer, size_t requested) {
	size_t offset = 0;
	while (offset < requested) {
		ssize_t n = read(fd, (char *)buffer + offset, requested - offset);
		if (n == 0) break;
		if (n < 0) {
			if (errno == EINTR) continue;
			return -1;
		}
		offset += (size_t)n;
	}
	return (ssize_t)offset;
}
```

TCP has no message boundaries: use a fixed-size header, length prefix, delimiter, or self-delimiting encoding. Validate lengths before allocation and treat peer input as untrusted.

## 13. Security and Isolation

Protection goals are confidentiality, integrity, and availability. **Authentication** answers who; **authorization** answers what they may do. Apply least privilege, validate at trust boundaries, avoid shell injection, use secure random values, and never assume a path or file descriptor is safe merely because a user supplied it.

Processes isolate virtual address spaces; user/kernel mode protects privileged instructions; page permissions enforce read/write/execute policy. Containers isolate processes using namespaces and limit resources with cgroups, but share the host kernel. Virtual machines provide a stronger hardware-virtualization boundary with more overhead.

Common vulnerabilities include TOCTOU races, symlink attacks, buffer overflows, use-after-free, confused deputy bugs, FD leaks, and denial of service through unbounded input or resource exhaustion. Mitigations include atomic APIs, `O_NOFOLLOW` where appropriate, ASLR, NX/DEP, stack canaries, sanitizers, quotas, and privilege dropping.

## 14. Linux Developer Toolkit

```sh
ps aux                         # process snapshot
top                            # CPU and memory activity
free -h; vmstat 1              # memory and paging pressure
strace -f -c ./program        # system-call summary
lsof -p PID                   # files and sockets held by a process
cat /proc/PID/status          # process state and memory counters
cat /proc/meminfo             # kernel memory view
ss -lntp                      # listening TCP sockets
gcc -Wall -Wextra -g file.c -pthread
valgrind --leak-check=full ./program
```

For races, use ThreadSanitizer where supported: `gcc -fsanitize=thread -g file.c -pthread`. For memory errors, use AddressSanitizer: `gcc -fsanitize=address,undefined -g file.c`.

### Diagnosis playbook

1. Define the symptom and time window.
2. Check CPU, run queue, memory, swap, disk latency, and network sockets.
3. Identify the process and its system-call or stack behavior.
4. Separate saturation from contention, leak, deadlock, and external dependency failure.
5. Reproduce with a small workload, measure before changing code, then verify the fix.

## 15. OS Design and Production Trade-offs

**Caching** improves latency and throughput but creates staleness and invalidation problems. **Durability** means an acknowledged write survives the stated failure model; clarify whether that means kernel memory, disk cache, or stable storage. **Backpressure** bounds queues and makes producers slow down instead of allowing unlimited memory growth.

For a worker service, define ownership of every resource: who creates and closes FDs, who joins threads, who cancels work, and what happens on partial failure. Use timeouts, cancellation, retry budgets, idempotency, and observability. Retries without backoff can amplify an outage.

For multicore software, prefer immutable data or message passing where practical. False sharing occurs when independent hot variables share a cache line; padding or layout changes can help, but measure first. NUMA systems have nonuniform memory latency, so thread and memory placement matter.
# Operating Systems for Software Engineers

An interview, placement, online-test, and day-to-day development handbook. Examples use Linux/POSIX and C unless stated otherwise.

## Table of Contents

- [1. How to Use These Notes](#1-how-to-use-these-notes)
- [2. OS Foundations](#2-os-foundations)
- [3. Processes and Threads](#3-processes-and-threads)
- [4. CPU Scheduling](#4-cpu-scheduling)
- [5. Synchronization](#5-synchronization)
- [6. Deadlocks](#6-deadlocks)
- [7. Memory Management](#7-memory-management)
- [8. Virtual Memory](#8-virtual-memory)
- [9. File Systems and Storage](#9-file-systems-and-storage)
- [10. I/O and Devices](#10-io-and-devices)
- [11. Interprocess Communication](#11-interprocess-communication)
- [12. Networking and Sockets](#12-networking-and-sockets)
- [13. Security and Isolation](#13-security-and-isolation)
- [14. Linux Developer Toolkit](#14-linux-developer-toolkit)
- [15. OS Design and Production Trade-offs](#15-os-design-and-production-trade-offs)
- [16. Interview Questions and Answers](#16-interview-questions-and-answers)
- [17. Online-Test Formula Sheet](#17-online-test-formula-sheet)
- [18. Practice Problems](#18-practice-problems)
- [19. Final Revision Checklist](#19-final-revision-checklist)

## 1. How to Use These Notes

**Placement path:** learn Sections 2-8, memorize the formula sheet, then solve the practice problems without looking at answers.

**Software-engineer path:** add Sections 9-15. Be able to explain what happens from a system call to hardware, diagnose resource pressure, and justify a design trade-off.

**Interview method:** answer in this order: definition, mechanism, example, trade-off, failure mode. State assumptions such as single-core versus multicore and preemptive versus cooperative scheduling.

**Useful experiments:** run `strace`, inspect `/proc`, compile the C snippets with `gcc -Wall -Wextra`, and use `top`, `vmstat`, `iostat`, `ss`, and `lsof` while a program runs.

**Coding companion:** use [Oscode.md](Oscode.md) for complete placement-style implementations and starter templates. The topic map below shows where to practice each section.

| Notes topic | Coding practice |
|---|---|
| OS foundations | [System calls and `errno`](Oscode.md#system-call-and-errno) |
| Processes and threads | [`fork`, pipes, `exec`, and thread lifecycle](Oscode.md#3-processes-pipes-and-exec) |
| CPU scheduling | [FCFS, SJF, SRTF, Priority, and Round Robin](Oscode.md#cpu-scheduling) |
| Synchronization | [Bounded producer-consumer queue](Oscode.md#producer-consumer-with-a-bounded-buffer) |
| Deadlocks | [Cycle detection and lock ordering](Oscode.md#deadlock-cycle-detection) |
| Memory management | [First-fit allocator simulation](Oscode.md#first-fit-memory-allocator-simulation) |
| Virtual memory | [Address translation and page replacement](Oscode.md#virtual-address-translation) |
| File systems and storage | [File copying and disk scheduling](Oscode.md#file-systems-and-storage) |
| I/O and devices | [Nonblocking `poll` event loop](Oscode.md#nonblocking-event-loop) |
| Interprocess communication | [Pipes, sockets, and shared memory](Oscode.md#interprocess-communication) |
| Networking | [Length-prefixed TCP server](Oscode.md#networking) |
| Security and isolation | [Safer file opening](Oscode.md#safer-file-opening) |
| Linux developer toolkit | [Process diagnosis exercise](Oscode.md#linux-developer-toolkit) |
| OS design and production trade-offs | [Bounded worker service](Oscode.md#os-design-and-production-trade-offs) |

## 2. OS Foundations

### What an operating system does

An OS is privileged software that abstracts hardware and manages resources. Its core responsibilities are:

- **Abstraction:** processes instead of CPU state, virtual memory instead of physical frames, files instead of disk sectors.
- **Resource management:** allocate CPU time, memory, storage, devices, and network access.
- **Protection:** isolate users and processes and enforce permissions.
- **Concurrency:** safely coordinate work that overlaps in time.
- **Persistence and recovery:** maintain file-system consistency across failures.

The **kernel** runs in privileged mode. Applications run in user mode and request services through **system calls**. A system call changes mode, validates arguments, performs protected work, and returns a result or error (`errno`). A library call such as `printf` may eventually use a system call such as `write`, but not every library call is a system call.

### Boot and execution path

Firmware initializes hardware and loads a bootloader. The bootloader loads the kernel and an initial RAM filesystem. The kernel initializes memory management, drivers, and schedulers, then starts the first user-space process (commonly `systemd`). A process executes instructions in user mode until it makes a system call, receives a device interrupt, triggers an exception such as a page fault, or is preempted by a timer interrupt.

### Kernel architectures

- **Monolithic kernel:** most services and drivers run in kernel space; fast paths, larger trusted computing base. Linux is modular monolithic.
- **Microkernel:** minimal kernel (IPC, scheduling, basic memory); drivers and services run outside it, improving isolation at IPC cost.
- **Hybrid:** combines both strategies.
- **Exokernel:** exposes hardware securely and leaves abstractions to applications.

### Interrupts, traps, and exceptions

An **interrupt** is usually an asynchronous hardware notification. A **trap** is a deliberate synchronous transfer, commonly a system call. An **exception** is synchronous due to the current instruction. Handlers do minimal urgent work; deferred work completes the rest later.

### Essential vocabulary

| Term | Meaning |
|---|---|
| Program | Passive executable file and data |
| Process | Program in execution with its own virtual address space |
| Thread | Schedulable execution path sharing a process's resources |
| Context switch | Save one execution context and restore another |
| Throughput | Work completed per unit time |
| Latency | Time until a requested result |
| Fairness | How evenly resources are distributed |

## 3. Processes and Threads

### Process layout and states

A typical process has text (machine code), read-only data, initialized and uninitialized data, shared libraries, heap, and stack. The process control block stores PID, state, registers, program counter, scheduling data, credentials, open-file references, and memory metadata.

Common states are **new**, **ready**, **running**, **blocked/waiting**, and **terminated**. A blocked process cannot use the CPU until an event occurs, such as I/O completion.

### `fork`, `exec`, and `wait`

`fork()` creates a child with a copy-on-write view of the parent's address space. It returns `0` in the child and the child's PID in the parent. `exec*()` replaces the current process image; it does not create a process. `waitpid()` collects a child and prevents a zombie.

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void) {
	pid_t child = fork();
	if (child < 0) return EXIT_FAILURE;
	if (child == 0) {
		execlp("ls", "ls", "-l", (char *)NULL);
		perror("execlp");
		_exit(127);
	}
	int status;
	if (waitpid(child, &status, 0) < 0) return EXIT_FAILURE;
	printf("child exit status: %d\n", WIFEXITED(status) ? WEXITSTATUS(status) : -1);
	return EXIT_SUCCESS;
}
```

**Zombie:** terminated child whose exit status has not been collected. **Orphan:** child whose parent terminated; it is adopted by a reaper process. A zombie consumes a process-table entry, not normal user memory.

### Threads and context switches

Threads share code, data, heap, and open files, but each has its own registers, program counter, and stack. Threads reduce creation and communication cost, but shared state creates races. Processes provide stronger fault isolation. On multicore machines, threads can execute simultaneously; concurrency does not always imply parallelism.

The kernel saves registers and scheduling state, switches address-space metadata if needed, and restores another task. Cost includes kernel work and cache/TLB disruption.

### Common traps

- `fork()` can return twice, but the return values differ.
- Buffered output may be duplicated after `fork()` if it was not flushed.
- `exec()` only returns on failure.
- A thread crash can terminate the whole process; a process crash normally does not corrupt another process's address space.

## 4. CPU Scheduling

### Metrics

For arrival time $A$, burst time $B$, completion time $C$, and first-run time $F$:

- Turnaround time: $TAT = C - A$
- Waiting time: $WT = TAT - B$
- Response time: $RT = F - A$
- Throughput: completed jobs / elapsed time

### Algorithms

- **FCFS:** non-preemptive and simple; the convoy effect makes short jobs wait behind a long job.
- **SJF:** shortest burst first; optimal average waiting time when burst lengths are known.
- **SRTF:** preemptive SJF; improves short-job latency but adds overhead and can starve long jobs.
- **Round Robin:** every ready task receives a time quantum; a small quantum improves response but raises context-switch cost.
- **Priority scheduling:** urgent tasks first; aging prevents starvation.
- **Multilevel queue:** separate queues for classes with usually fixed policy.
- **Multilevel feedback queue:** tasks move between queues based on behavior and favors interactive work.
- **Real-time scheduling:** deadlines matter more than average throughput; EDF chooses the nearest deadline.

Draw a Gantt chart. At each decision point add arrived jobs, apply the algorithm, record first execution and completion, then calculate metrics. Include context-switch cost only when specified. On multicore systems, schedule each core separately.

**Starvation** means indefinite postponement. **Priority inversion** occurs when a high-priority task waits for a lock held by a low-priority task while medium-priority tasks run. Priority inheritance temporarily raises the lock holder's priority.

## 5. Synchronization

### Race conditions and critical sections

A race occurs when the result depends on timing. A critical section accesses shared mutable state. Correct synchronization requires **mutual exclusion**, **progress**, and **bounded waiting**. An operation is **atomic** when it appears indivisible to observers.

Hardware primitives include test-and-set, compare-and-swap (CAS), fetch-and-add, and load-linked/store-conditional. A mutex is generally preferable to a hand-written spinlock for long or blocking critical sections.

### Mutex and condition variable

A condition variable waits for a predicate while atomically releasing the mutex. Always use `while`, not `if`, because wakeups can be spurious and another thread may consume the condition first.

```c
#include <pthread.h>

static int items;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t available = PTHREAD_COND_INITIALIZER;

void produce(void) {
	pthread_mutex_lock(&lock);
	++items;
	pthread_cond_signal(&available);
	pthread_mutex_unlock(&lock);
}

void consume(void) {
	pthread_mutex_lock(&lock);
	while (items == 0) pthread_cond_wait(&available, &lock);
	--items;
	pthread_mutex_unlock(&lock);
}
```

### Synchronization tools

- **Mutex:** one owner; protects an invariant.
- **Semaphore:** integer permit count; `wait/P` decrements or blocks, `post/V` increments.
- **Read-write lock:** many readers or one writer; beware starvation.
- **Monitor:** shared state plus procedures and implicit mutual exclusion.
- **Barrier:** all participating threads wait until a phase completes.
- **Spinlock:** repeatedly polls; useful only for very short sections where sleeping is undesirable.
- **Atomic variable:** lock-free operations for simple state; atomics still require memory-order reasoning.

### Memory ordering

Compiler and CPU reordering can make unsynchronized observations surprising. Acquire operations prevent later operations from moving before them; release operations prevent earlier operations from moving after them. Sequential consistency is easiest to reason about but may cost more. `volatile` does not make a variable atomic or provide inter-thread synchronization.

## 6. Deadlocks

Deadlock requires all four Coffman conditions:

1. Mutual exclusion.
2. Hold and wait.
3. No preemption.
4. Circular wait.

**Prevention** breaks a condition, such as imposing a global lock order. **Avoidance** grants a request only if the system remains safe; Banker's algorithm needs maximum demands and available resources. **Detection and recovery** finds cycles and aborts, rolls back, or preempts. Some systems ignore rare deadlocks and rely on recovery.

Lock ordering is practical: assign each lock a rank and acquire only in increasing order. A **livelock** means tasks keep changing state but make no progress. **Starvation** means one task never gets a needed resource; it may occur without deadlock.

## 7. Memory Management

### Address translation

The CPU produces a virtual address. The MMU translates it to a physical address using page tables, commonly assisted by a TLB. A page-table entry may contain a frame number, present bit, read/write bit, user/supervisor bit, accessed bit, dirty bit, and execute-disable bit.

For page size $2^p$ bytes and virtual-address width $v$, offset bits are $p$ and virtual page-number bits are $v-p$. A single-level table has $2^{v-p}$ entries, which can be huge; multi-level and inverted tables reduce wasted space.

### Allocation strategies

- **Contiguous allocation:** fast translation but external fragmentation.
- **First fit:** first adequate hole; usually fast.
- **Best fit:** smallest adequate hole; may create many tiny holes.
- **Worst fit:** largest hole; preserves medium holes but can waste space.
- **Buddy allocator:** splits power-of-two blocks and merges quickly; internal fragmentation is possible.
- **Slab allocator:** caches objects of common kernel types.

**Internal fragmentation** is wasted space inside an allocated block. **External fragmentation** is free space split into unusable holes. Paging removes external fragmentation for physical allocation but adds page-table and internal-fragmentation overhead.

### C memory errors

Match every `malloc` with `free`. A use-after-free, double free, buffer overflow, or uninitialized read is undefined behavior. `realloc` may move an allocation; assign its result to a temporary before replacing the original pointer.

```c
#include <stdlib.h>

int *grow(int *values, size_t old_count, size_t new_count) {
	int *candidate = realloc(values, new_count * sizeof *values);
	if (candidate == NULL && new_count != 0) return values;
	for (size_t i = old_count; i < new_count; ++i) candidate[i] = 0;
	return candidate;
}
```

## 8. Virtual Memory

### Demand paging

Pages are loaded when first referenced. A page fault traps to the kernel, which validates the access, finds a free frame or chooses a victim, writes a dirty victim if necessary, reads the requested page, updates page tables and the TLB, and restarts the instruction. Invalid access becomes a protection fault, often observed as `SIGSEGV`.

### Page replacement

- **FIFO:** evicts the oldest page and can show Belady's anomaly.
- **Optimal:** evicts the page used farthest in the future; theoretical lower bound.
- **LRU:** evicts the least recently used page; exact tracking is costly.
- **Clock/second chance:** practical LRU approximation using a reference bit.

The **working set** is the recently used pages needed by a process. If total working sets exceed available frames, the system **thrashes**: page-fault time dominates useful work. Reduce concurrency, add memory, or improve locality.

Effective access time can be approximated by $EAT = (1-p) \times m + p \times page\text{-}fault\ cost$, where $p$ is page-fault probability and $m$ is normal memory access time. Even a tiny $p$ can dominate because storage is much slower than RAM.

### Copy-on-write and memory mapping

After `fork`, parent and child initially share physical pages marked read-only. A write causes a fault and the kernel copies that page. `mmap` maps files or anonymous pages into virtual memory and enables shared memory and efficient file access.
