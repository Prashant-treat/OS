 # Operating Systems Coding for Placements

This sheet focuses on the coding questions commonly asked in placement interviews and online assessments. Examples use C and POSIX APIs. In an interview, explain the invariant, ownership of resources, failure handling, and time/space complexity before writing code.

## 1. C Coding Patterns

### Read input safely

Prefer bounded input and check every conversion. Do not assume that input is valid or that a requested allocation cannot overflow.

```c
#include <limits.h>
#include <stdio.h>

int read_nonnegative_int(int *value) {
	long parsed;
	char line[64];

	if (fgets(line, sizeof line, stdin) == NULL) return 0;
	char extra;
	if (sscanf(line, "%ld %c", &parsed, &extra) != 1 ||
		parsed < 0 || parsed > INT_MAX) return 0;
	*value = (int)parsed;
	return 1;
}
```

### Dynamic allocation

Check multiplication before allocating an array, initialize ownership clearly, and free memory on every exit path.

```c
#include <stdint.h>
#include <stdlib.h>

int *allocate_ints(size_t count) {
	if (count > SIZE_MAX / sizeof(int)) return NULL;
	return calloc(count, sizeof(int));
}
```

### Swap and reverse

```c
void reverse(int *values, size_t length) {
	for (size_t left = 0, right = length; left < right / 2; ++left) {
		size_t other = right - left - 1;
		int temporary = values[left];
		values[left] = values[other];
		values[other] = temporary;
	}
}
```

Common edge cases are an empty array, one element, duplicate values, negative values, integer overflow, and a `NULL` pointer contract.

## 2. OS Algorithms Asked in Coding Rounds

### FCFS scheduling

For each process, track arrival time, burst time, completion time, turnaround time, waiting time, and response time. Sort by arrival time, maintain the current CPU time, and advance it to the next arrival when the ready queue is empty.

```text
completion = max(current_time, arrival) + burst
turnaround = completion - arrival
waiting = turnaround - burst
response = first_start - arrival
```

Use a stable tie-break rule, usually arrival order or process ID, and state it explicitly.

### Round Robin scheduling

Use a queue of ready process IDs. Add processes whose arrival time is at most the current time, run the front process for `min(quantum, remaining_time)`, then enqueue newly arrived processes before re-adding the unfinished process. A quantum of zero is invalid.

The usual implementation is $O(n \\, log n)$ if arrivals are sorted and a queue is used; it can become $O(n^2)$ if the ready list is scanned repeatedly.

### Producer-consumer with a bounded buffer

The invariant is `0 <= count <= capacity`. Producers wait while the buffer is full; consumers wait while it is empty. Every access to the buffer and `count` is protected by the same mutex.

```c
#include <pthread.h>

typedef struct {
	int *items;
	size_t capacity;
	size_t count;
	size_t head;
	size_t tail;
	pthread_mutex_t mutex;
	pthread_cond_t not_empty;
	pthread_cond_t not_full;
} queue_t;

int queue_put(queue_t *queue, int value) {
	pthread_mutex_lock(&queue->mutex);
	while (queue->count == queue->capacity)
		pthread_cond_wait(&queue->not_full, &queue->mutex);
	queue->items[queue->tail] = value;
	queue->tail = (queue->tail + 1) % queue->capacity;
	++queue->count;
	pthread_cond_signal(&queue->not_empty);
	pthread_mutex_unlock(&queue->mutex);
	return 1;
}

int queue_get(queue_t *queue, int *value) {
	pthread_mutex_lock(&queue->mutex);
	while (queue->count == 0)
		pthread_cond_wait(&queue->not_empty, &queue->mutex);
	*value = queue->items[queue->head];
	queue->head = (queue->head + 1) % queue->capacity;
	--queue->count;
	pthread_cond_signal(&queue->not_full);
	pthread_mutex_unlock(&queue->mutex);
	return 1;
}
```

Use `while`, not `if`, around condition-variable waits because wakeups can be spurious and another thread may consume the condition first. A production queue also needs a shutdown flag so blocked threads can exit.

### Page replacement

- **FIFO:** maintain a queue of pages and evict the oldest page. Simple, but it can show Belady's anomaly.
- **LRU:** evict the least recently used page. A hash map plus linked list gives $O(1)$ average access.
- **Optimal:** evict the page whose next use is farthest in the future. It is useful as a theoretical minimum, not as a practical online algorithm.

For each reference string, update the frame state after every reference and count a fault only when the page is absent. State whether empty frames are available before evicting a page.

### Banker's safety check

To test whether a resource state is safe, set `work = available`, mark all processes unfinished, and repeatedly find a process whose remaining need is at most `work`. Pretend it finishes and add its allocation to `work`. If all processes can finish, the state is safe; otherwise it is unsafe.

### Virtual-address calculations

For a page size of $2^p$ bytes, the page offset uses `p` bits. For a virtual address width of `v` bits, the virtual page number uses `v - p` bits and the number of virtual pages is $2^{v-p}$. For physical memory of $2^m$ bytes, the frame number uses `m - p` bits.

To translate an address, split it into page number and offset, look up the page number in the page table, then combine the frame number with the unchanged offset. For a multi-level page table, divide the page-number bits among the levels and include the page-table entry size when calculating table memory.

### Disk scheduling

Given a current head position and request queue, calculate total head movement by writing the service order first, then summing the absolute difference between consecutive positions.

- **FCFS:** fair arrival order, usually larger movement.
- **SSTF:** nearest request first, but distant requests can starve.
- **SCAN:** moves in one direction and services requests, then reverses.
- **C-SCAN:** services in one direction and jumps back, producing more uniform waits.

State the initial direction and whether the disk endpoints count as visited; both details change the answer.

## 3. Processes, Pipes, and `exec`

### Correct `fork` reasoning

After `fork`, both processes continue from the next instruction. The return value is:

- `0` in the child;
- the child PID in the parent;
- `-1` if creation failed.

The number of processes can double at each unconditional `fork`, but branches and loop bounds determine the actual count. Draw a process tree before calculating output order; scheduling makes output order nondeterministic unless synchronization is used.

### Pipe checklist

1. Create the pipe before `fork`.
2. The reader closes its write end.
3. The writer closes its read end.
4. Close unused descriptors in every process.
5. The reader sees EOF only after every write end is closed.
6. Call `waitpid` so the parent does not leave a zombie.

```c
#include <sys/wait.h>
#include <unistd.h>

int run_child(const char *program, char *const arguments[]) {
	int descriptors[2];
	if (pipe(descriptors) == -1) return -1;

	pid_t child = fork();
	if (child == -1) {
		close(descriptors[0]);
		close(descriptors[1]);
		return -1;
	}
	if (child == 0) {
		close(descriptors[0]);
		if (dup2(descriptors[1], STDOUT_FILENO) == -1) _exit(127);
		close(descriptors[1]);
		execvp(program, arguments);
		_exit(127);
	}

	close(descriptors[1]);
	char buffer[256];
	ssize_t count;
	while ((count = read(descriptors[0], buffer, sizeof buffer)) > 0) {
		/* Process or store count bytes from buffer. */
	}
	close(descriptors[0]);
	int status;
	if (count == -1 || waitpid(child, &status, 0) == -1) return -1;
	return WIFEXITED(status) ? WEXITSTATUS(status) : -1;
}
```

Use `_exit` in the child after a failed `exec` to avoid flushing copied stdio buffers twice.

## 4. File and Socket Coding

### Complete reads and writes

Regular files, pipes, terminals, and sockets may perform only part of the requested operation. Loop until the buffer is complete, EOF occurs, or an unrecoverable error occurs. Retry `EINTR`; do not retry forever on `EAGAIN` without waiting for readiness.

```c
#include <errno.h>
#include <unistd.h>

ssize_t write_all(int fd, const void *data, size_t length) {
	const char *bytes = data;
	size_t offset = 0;
	while (offset < length) {
		ssize_t written = write(fd, bytes + offset, length - offset);
		if (written > 0) {
			offset += (size_t)written;
		} else if (written < 0 && errno == EINTR) {
			continue;
		} else {
			return -1;
		}
	}
	return (ssize_t)offset;
}
```

For a TCP protocol, choose a message framing method such as a fixed header plus length, validate the length before allocation, handle disconnect (`read == 0`), and apply timeouts or readiness polling.

### File-copy checklist

- Check `open`, `read`, `write`, `fsync` when durability matters, and `close` where its error is meaningful.
- Preserve partial-write handling.
- Use `O_CLOEXEC` where descriptors must not cross `exec`.
- Do not follow untrusted paths blindly; consider permissions, symlinks, and TOCTOU races.
- Decide whether the destination should be truncated, created exclusively, or atomically replaced.

## 5. Concurrency Questions

### Race condition

A race occurs when correctness depends on the timing of unsynchronized accesses to shared state. `volatile` does not make an operation atomic. Fix the specific invariant with a mutex, semaphore, condition variable, atomic operation, or message passing.

### Deadlock

Remember the four Coffman conditions: mutual exclusion, hold and wait, no preemption, and circular wait. The most practical prevention technique is a global lock order: every thread acquires multiple locks in the same order and releases them in reverse order.

### Reader-writer and dining philosophers

For reader-writer problems, state whether readers or writers have priority and how starvation is prevented. For dining philosophers, break circular wait with resource ordering, a waiter, or limiting the number of philosophers who may try simultaneously.

### Atomic counter

Use an atomic increment when the only invariant is the value of one counter. Use a mutex when multiple fields must change consistently or when the protected operation is larger than one atomic instruction.

### Threads and lifecycle

Create a thread with `pthread_create`, pass a pointer whose lifetime exceeds the thread's use, and call `pthread_join` unless the thread was deliberately detached. Never return the address of a local variable from a thread function. A detached thread releases its own resources, but its result cannot be joined later.

Cancellation is a cooperative protocol, not an immediate guarantee. Protect cleanup-sensitive resources with cleanup handlers or an explicit shutdown path, and make sure every producer and consumer can observe the shutdown flag and wake from its condition variable.

### Atomics and memory ordering

An atomic variable prevents data races on that variable, but it does not automatically make a group of variables consistent. Use release-store and acquire-load when publishing initialized data to another thread; use sequential consistency when simplicity matters and performance has not been measured. Explain which operation establishes the happens-before relationship.

### Signals and shared memory

Signal handlers should set a `volatile sig_atomic_t` flag or write to a pipe. Do not call `malloc`, `printf`, most locking functions, or other non-async-signal-safe functions from a handler. The main loop can perform the real work after observing the flag.

Shared memory is fast because processes avoid copying, but it is only storage, not synchronization. Pair it with a process-shared mutex/semaphore or an atomic protocol, define initialization and ownership, and specify what happens if a process exits while holding a lock.

## 6. Online-Test Strategy

1. Read the input and output contract twice, including limits and modulo rules.
2. Write down the invariant and a brute-force solution for tiny inputs.
3. Choose the data structure from the constraints: queue, heap, hash table, stack, graph, or sliding window.
4. Test empty input, one item, duplicates, maximum values, overflow boundaries, and already sorted or reverse-sorted data.
5. Compile with warnings: `gcc -std=c11 -Wall -Wextra -Wpedantic -g file.c`.
6. For threaded code, compile with `-pthread`; use AddressSanitizer and ThreadSanitizer when supported.

### Common traps

- confusing arrival, burst, completion, turnaround, waiting, and response time;
- forgetting that a context switch has overhead if the question includes it;
- using `if` instead of `while` for a condition-variable wait;
- assuming `fork` output order is deterministic;
- forgetting inherited or duplicated file descriptors;
- treating TCP as message-oriented;
- confusing a virtual page number with a physical frame number;
- calculating disk movement without stating the initial direction;
- allocating `count * size` without checking overflow;
- using a signed integer for a value that can exceed `INT_MAX`;
- returning a pointer to a local variable;
- joining a detached thread or passing it a pointer to expired storage;
- doing non-async-signal-safe work inside a signal handler;
- ignoring cleanup after an error.

## 7. Practice Set

1. Implement FCFS and Round Robin scheduling and print average waiting and response times.
2. Implement FIFO, LRU, and optimal page replacement for the same reference string.
3. Build a bounded producer-consumer queue with clean shutdown.
4. Write a parent-child program where the child runs `sort` through `exec` and the parent reads its output through a pipe.
5. Implement `read_full` and `write_all`, then use them in a length-prefixed client-server protocol.
6. Find and repair a race in two threads incrementing a shared counter.
7. Detect a cycle in a wait-for graph and print a deadlock explanation.
8. Implement a small shell supporting one pipe, redirection, and `waitpid`.
9. Given page size, virtual address width, and physical memory size, calculate page offset bits, page count, frame count, and address fields.
10. Diagnose a program that has high CPU, growing memory, blocked threads, or leaked file descriptors using Linux tools.
11. Calculate FCFS, SSTF, SCAN, and C-SCAN disk-head movement for the same request queue.
12. Design a thread shutdown protocol and explain how it wakes blocked producer and consumer threads.

## 8. Final Coding Checklist

- Can I explain the invariant before writing synchronization code?
- Do I check all system-call results and handle `EINTR`?
- Do I handle partial reads and writes?
- Is ownership of every allocation and file descriptor clear?
- Can I explain the complexity and worst-case behavior?
- Have I tested empty, boundary, duplicate, overflow, and failure cases?
- Can I explain the difference between a process, thread, coroutine, and async event loop?

## 9. Coding Task for Every OS Topic

Use this section as a one-exercise-per-topic revision map. For each task, write the smallest working version first, then explain its failure cases and complexity.

### OS foundations

Write a program that calls `write` directly instead of `printf`, checks its return value, and prints an `errno` description when the call fails. Explain the user-mode to kernel-mode transition, system-call arguments, and why a short write must be handled.

### Processes and threads

Build a parent that creates two children, sends each a different command through `exec`, collects both with `waitpid`, and reports whether each exited normally or from a signal. Then implement the same workload with two pthreads and compare shared memory, failure isolation, and cleanup.

### CPU scheduling

Implement FCFS, SJF, SRTF, Priority, and Round Robin behind one scheduling interface. Feed all algorithms the same process list and print completion, turnaround, waiting, response time, throughput, and number of context switches. Define tie-breaking and preemption rules in the input contract.

### Synchronization

Implement a bounded queue with multiple producers and consumers. Protect `head`, `tail`, and `count` with a mutex, wait in loops on condition variables, and add a shutdown flag that wakes every blocked thread. Test with zero, one, and many producers and consumers.

### Deadlocks

Represent processes and resources as a graph, detect a cycle with DFS, and print the cycle. Extend the program with a lock-order checker that rejects an attempt to acquire a lower-ranked lock after a higher-ranked lock.

### Memory management

Implement first-fit and best-fit allocation over a simulated free-list. Support allocate, free, coalesce adjacent holes, and print internal and external fragmentation. Check every size calculation for overflow and reject zero-sized or impossible requests according to the stated contract.

### Virtual memory

Write functions that split a virtual address into page number and offset, translate it through a simulated page table, and report a page fault when the entry is invalid. Add a small TLB and compare hit and miss counts. Test non-power-of-two input rejection and the largest representable address.

### File systems and storage

Write a file copier that handles short reads and writes, retries `EINTR`, preserves an error from either file descriptor, and closes descriptors on every path. Separately implement FCFS, SSTF, SCAN, and C-SCAN disk-head movement and print total seek distance.

### I/O and devices

Create a nonblocking event loop using `poll` or `epoll` that reads from multiple descriptors until `EAGAIN`, handles EOF, and removes closed descriptors. Explain why readiness notification does not consume data and why edge-triggered loops must drain the descriptor.

### Interprocess communication

Implement the same producer-consumer workload with a pipe, a Unix-domain socket, and shared memory. Document message boundaries, copying cost, synchronization, EOF behavior, and what happens when one participant exits early.

### Networking

Build a TCP echo server that accepts multiple clients, frames messages with a length prefix, handles partial `recv` and `send`, rejects oversized lengths, and closes disconnected clients. Add a timeout and explain why TCP does not preserve application message boundaries.

### Security and isolation

Write a privileged-file access example using least privilege, checked permissions, `O_CLOEXEC`, and an appropriate `openat`-style directory boundary. Reject untrusted lengths before allocation and never construct a shell command by concatenating user input. Explain the TOCTOU risk in a check-then-open sequence.

### Linux developer toolkit

Create a small diagnostic script that records CPU, memory, load average, open descriptors, sockets, and system-call summaries for a process. Use it to distinguish CPU saturation, blocked I/O, memory pressure, lock contention, and a file-descriptor leak instead of guessing from one metric.

### OS design and production trade-offs

Design a worker service with a bounded queue, backpressure, cancellation, timeouts, retry limits, and graceful shutdown. State ownership for every resource, make work idempotent before retrying, and expose queue length, latency, error rate, and saturation metrics.

### Interview questions and answers

For each common question, write a tiny experiment rather than memorizing only a definition: observe `fork` output ordering, measure a mutex versus atomic counter, trigger a partial pipe read, inspect copy-on-write memory after `fork`, and demonstrate a zombie before calling `waitpid`.

### Online-test formulas

Write a calculator that accepts arrival, burst, completion, page-size, address-width, TLB, and disk parameters and prints every intermediate value. Use integer arithmetic for counts, validate powers of two, and test boundary values before trusting the final formula.

### Practice and revision

Solve each practice problem twice: first with a straightforward implementation that is easy to verify, then with the data structure suggested by the constraints. Keep a small test table containing normal, empty, duplicate, maximum, overflow, timeout, and failure cases.

## 10. Starter Code for the Remaining Topics

These are deliberately small templates. In an interview, complete the missing policy decisions and explain the limitations instead of presenting them as production-ready libraries.

### System call and `errno`

```c
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void) {
	const char message[] = "hello from write\n";
	ssize_t written = write(STDOUT_FILENO, message, sizeof message - 1);
	if (written < 0) {
		fprintf(stderr, "write failed: %s\n", strerror(errno));
		return 1;
	}
	if ((size_t)written != sizeof message - 1) {
		fprintf(stderr, "short write\n");
		return 1;
	}
	return 0;
}
```

### Deadlock cycle detection

```c
#include <stddef.h>

static int has_cycle(size_t node, const int graph[][4], size_t count,
					 int *visiting, int *finished) {
	if (finished[node]) return 0;
	if (visiting[node]) return 1;
	visiting[node] = 1;
	for (size_t next = 0; next < count; ++next)
		if (graph[node][next] && has_cycle(next, graph, count, visiting, finished))
			return 1;
	visiting[node] = 0;
	finished[node] = 1;
	return 0;
}
```

The graph must be directed with one node per process for a wait-for graph. For a resource-allocation graph with resource nodes, first reduce resource ownership and requests to process-to-process edges.

### First-fit memory allocator simulation

```c
#include <stddef.h>

typedef struct block {
	size_t start;
	size_t length;
	struct block *next;
} block_t;

block_t *first_fit(block_t *free_list, size_t request) {
	if (request == 0) return NULL;
	for (block_t *block = free_list; block != NULL; block = block->next) {
		if (block->length < request) continue;
		block->start += request;
		block->length -= request;
		return block;
	}
	return NULL;
}
```

For a complete allocator, return the original start address, split a block only when the remainder can hold metadata, and coalesce adjacent free blocks after `free`. The simple template above is useful for explaining the first-fit decision but is not a complete `malloc` replacement.

### Virtual-address translation

```c
#include <stdint.h>

int translate(uint32_t virtual_address, unsigned offset_bits,
			  const uint32_t *page_table, size_t page_count,
			  uint32_t *physical_address) {
	uint32_t offset_mask = (UINT32_C(1) << offset_bits) - 1;
	uint32_t page = virtual_address >> offset_bits;
	if (page >= page_count || page_table[page] == UINT32_MAX) return 0;
	*physical_address = (page_table[page] << offset_bits) |
		(virtual_address & offset_mask);
	return 1;
}
```

Validate that `offset_bits` is smaller than 32 before shifting. A real page-table entry also needs permission and present-bit checks; a TLB should be consulted before the page table.

### Nonblocking event loop

```c
#include <errno.h>
#include <poll.h>
#include <unistd.h>

int drain_ready(int fd) {
	char buffer[1024];
	for (;;) {
		ssize_t count = read(fd, buffer, sizeof buffer);
		if (count > 0) continue;
		if (count == 0) return 0;
		if (errno == EINTR) continue;
		if (errno == EAGAIN || errno == EWOULDBLOCK) return 1;
		return -1;
	}
}

int wait_for_input(int fd) {
	struct pollfd event = {.fd = fd, .events = POLLIN};
	int result;
	do {
		result = poll(&event, 1, 1000);
	} while (result < 0 && errno == EINTR);
	if (result <= 0) return result;
	if (event.revents & (POLLERR | POLLNVAL)) return -1;
	return (event.revents & (POLLIN | POLLHUP)) ? drain_ready(fd) : 1;
}
```

Set the descriptor to nonblocking mode before using `drain_ready`. A real server keeps one event record per client and removes a descriptor on EOF or error.

### Signal-safe shutdown flag

```c
#include <signal.h>
#include <stdio.h>

static volatile sig_atomic_t stop_requested;

static void handle_term(int signal_number) {
	(void)signal_number;
	stop_requested = 1;
}

int main(void) {
	struct sigaction action = {0};
	action.sa_handler = handle_term;
	sigemptyset(&action.sa_mask);
	sigaction(SIGTERM, &action, NULL);
	while (!stop_requested) {
		/* Do normal work outside the signal handler. */
	}
	return 0;
}
```

Use `sigaction` rather than the historical `signal` interface. If the main loop must wake from blocking I/O, use a self-pipe or `signalfd` rather than calling unsafe functions in the handler.

### Safer file opening

```c
#include <fcntl.h>
#include <sys/stat.h>
#include <unistd.h>

int open_private_file(const char *name) {
	return open(name, O_WRONLY | O_CREAT | O_EXCL | O_CLOEXEC,
				0600);
}
```

This prevents accidental descriptor inheritance and avoids overwriting an existing file, but it does not solve every path-traversal or symlink problem. For an untrusted directory boundary, open the directory first and use `openat` with platform-appropriate no-follow and resolution restrictions.
