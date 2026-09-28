# Operating Systems Diagrams

Mermaid diagrams for the core concepts covered in [OSnotes.md](OSnotes.md) and [Oscode.md](Oscode.md). Render this file in a Markdown viewer that supports Mermaid.

## OS Mind Map

```mermaid
mindmap
	root((Operating System))
		Kernel
			User mode and kernel mode
			System calls
			Interrupts and exceptions
		Process management
			Processes and threads
			CPU scheduling
			Context switches
			IPC
		Concurrency
			Mutexes and semaphores
			Condition variables
			Deadlocks
		Memory management
			Virtual memory
			Paging and page tables
			TLB and page faults
			Replacement policies
		Files and I/O
			File systems and inodes
			Drivers and devices
			Buffering and DMA
			Disk scheduling
		Protection
			Permissions
			Isolation
			Least privilege
```

## Boot and Program Execution

```mermaid
flowchart TD
		A[Power on] --> B[Firmware initializes hardware]
		B --> C[Bootloader loads kernel]
		C --> D[Kernel initializes memory, drivers, and scheduler]
		D --> E[Start first user-space process]
		E --> F[Program runs in user mode]
		F --> G{Needs a protected service?}
		G -- No --> F
		G -- System call --> H[Enter kernel mode]
		H --> I[Validate request and perform operation]
		I --> J[Return result and resume user mode]
		J --> F
		K[Hardware event] --> L[Interrupt handler]
		L --> M[Handle event or schedule deferred work]
		M --> F
```

## Process State and Scheduling

```mermaid
stateDiagram-v2
		[*] --> New
		New --> Ready: admitted
		Ready --> Running: dispatched
		Running --> Ready: preempted / time slice ends
		Running --> Waiting: waits for I/O or event
		Waiting --> Ready: I/O or event completes
		Running --> Terminated: exits
		Terminated --> [*]
```

The scheduler chooses a process from the ready queue. The dispatcher performs the context switch to that process.

```mermaid
flowchart LR
		A[Ready queue] --> B[CPU scheduler selects process]
		B --> C[Dispatcher restores context]
		C --> D[Process runs on CPU]
		D --> E{What happens next?}
		E -- Time slice expires --> A
		E -- Waits for I/O --> F[Blocked queue]
		F --> G[I/O completes]
		G --> A
		E -- Process exits --> H[Terminated]
```

## Virtual Address Translation

```mermaid
flowchart TD
		A[CPU generates virtual address] --> B[Split into virtual page number and offset]
		B --> C{Translation in TLB?}
		C -- Yes --> D[Get physical frame number]
		C -- No --> E[Look up page table]
		E --> F{Page present and access permitted?}
		F -- Yes --> G[Cache translation in TLB]
		G --> D
		F -- No --> H[Page-fault trap to kernel]
		H --> I{Address valid?}
		I -- No --> J[Reject access / signal process]
		I -- Yes --> K[Load or create page; choose frame]
		K --> L[Update page table]
		L --> G
		D --> M[Combine frame number with unchanged offset]
		M --> N[Access physical memory]
```

## Producer-Consumer Synchronization

```mermaid
flowchart TD
		P[Producer] --> A[Lock buffer mutex]
		A --> B{Buffer full?}
		B -- Yes --> C[Wait on not-full; release mutex while waiting]
		C --> A
		B -- No --> D[Add item and increment count]
		D --> E[Signal not-empty]
		E --> F[Unlock mutex]
		F --> P

		Q[Consumer] --> G[Lock buffer mutex]
		G --> H{Buffer empty?}
		H -- Yes --> I[Wait on not-empty; release mutex while waiting]
		I --> G
		H -- No --> J[Remove item and decrement count]
		J --> K[Signal not-full]
		K --> L[Unlock mutex]
		L --> Q
```

Invariant: `0 <= count <= capacity`. Use `while` to recheck the condition after every wakeup.

## Deadlock: Resource Allocation Graph

```mermaid
flowchart LR
		P1((Process P1)) -->|requests| R2[Resource R2]
		R1[Resource R1] -->|allocated to| P1
		P2((Process P2)) -->|requests| R1
		R2 -->|allocated to| P2
```

With one instance of each resource, this cycle means deadlock: P1 holds R1 and waits for R2, while P2 holds R2 and waits for R1. A consistent global lock order prevents this circular-wait pattern.
