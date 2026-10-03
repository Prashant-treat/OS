# Operating Systems Diagrams

This file shows the core operating-system ideas as clean Mermaid diagrams. The diagrams below focus on the flow of control, memory, and resources in a Linux-like system.

## 1. OS Architecture

```mermaid
flowchart TB
    U1[User programs] --> S1[System calls]
    S1 --> K1[Kernel]
    K1 --> P1[Process management]
    K1 --> M1[Memory management]
    K1 --> F1[File system]
    K1 --> I1[IPC and networking]
    K1 --> D1[Device drivers]
    D1 --> H1[Hardware]
    H1 --> K1
    M1 --> T1[MMU + page tables + TLB]
    T1 --> H1
```

The kernel is the privileged layer that mediates user requests to hardware and enforces protection, isolation, and resource sharing.

## 2. Boot and Program Execution

```mermaid
flowchart TD
    A[Power on] --> B[Firmware initializes hardware]
    B --> C[Bootloader loads kernel]
    C --> D[Kernel initializes memory, drivers, and scheduler]
    D --> E[Start the first user-space process]
    E --> F[Process runs in user mode]
    F --> G{Needs privileged service?}
    G -- No --> F
    G -- Yes --> H[Execute system call]
    H --> I[CPU switches to kernel mode]
    I --> J[Kernel validates arguments and performs operation]
    J --> K[Restore user context and return]
    K --> F

    L[Hardware interrupt or trap] --> M[Kernel interrupt handler]
    M --> N[Schedule work or resume waiting task]
    N --> F
```

This flow shows the boundary between user mode and kernel mode. Protected operations require a controlled transition through the kernel.

## 3. Process States and Scheduling

```mermaid
stateDiagram-v2
    [*] --> New
    New --> Ready: admitted
    Ready --> Running: scheduled
    Running --> Ready: time slice ends or preempted
    Running --> Waiting: blocks on I/O or event
    Waiting --> Ready: event completes or I/O finishes
    Running --> Terminated: exits
    Terminated --> [*]
```

```mermaid
flowchart LR
    A[Ready queue] --> B[Scheduler picks next process]
    B --> C[Dispatcher performs context switch]
    C --> D[Process runs on CPU]
    D --> E{What happens next?}
    E -- Time slice expires --> A
    E -- Waits for I/O --> F[Blocked queue]
    F --> G[Kernel wakes process]
    G --> A
    E -- Process exits --> H[Terminated]
```

The ready queue holds runnable tasks; the scheduler chooses which one gets CPU time, and the dispatcher restores the selected context.

## 4. Virtual Memory Translation

```mermaid
flowchart TD
    A[CPU generates virtual address] --> B[Split into VPN and page offset]
    B --> C{TLB hit?}
    C -- Yes --> D[Translate to physical frame number]
    C -- No --> E[Walk page table]
    E --> F{Page present and permitted?}
    F -- Yes --> G[Update TLB]
    G --> D
    F -- No --> H[Page-fault trap to kernel]
    H --> I{Address valid?}
    I -- No --> J[Reject access or signal process]
    I -- Yes --> K[Load or allocate page and choose frame]
    K --> L[Update page table]
    L --> G
    D --> M[Combine frame number with offset]
    M --> N[Access physical memory]
```

A TLB reduces repeated page-table walks. A page fault occurs when the requested virtual page is not mapped or is not currently resident.

## 5. Producer-Consumer Synchronization

```mermaid
flowchart TD
    P[Producer] --> A[Lock mutex]
    A --> B{Buffer full?}
    B -- Yes --> C[Wait on not-full condition]
    C --> A
    B -- No --> D[Write item to buffer]
    D --> E[Signal not-empty]
    E --> F[Unlock mutex]

    Q[Consumer] --> G[Lock mutex]
    G --> H{Buffer empty?}
    H -- Yes --> I[Wait on not-empty condition]
    I --> G
    H -- No --> J[Read item from buffer]
    J --> K[Signal not-full]
    K --> L[Unlock mutex]
```

The key invariant is: `0 <= count <= capacity`. Conditions must be checked in a loop, not with a single test, after every wakeup.

## 6. Deadlock with Resource Allocation Graph

```mermaid
flowchart LR
    P1((Process P1)) -->|requests| R2[Resource R2]
    R1[Resource R1] -->|allocated to| P1
    P2((Process P2)) -->|requests| R1
    R2 -->|allocated to| P2
```

This cycle shows circular wait: P1 holds R1 and waits for R2, while P2 holds R2 and waits for R1. A global lock-order rule prevents this pattern.

## 7. `fork` and `exec`

```mermaid
flowchart TD
    A[Parent process runs] --> B{"Calls fork()?"}
    B -- Yes --> C[Kernel creates child process]
    C --> D[Child gets copy of address space and file table]
    D --> E{"Child calls exec()?"}
    E -- No --> F[Child continues running its own code]
    E -- Yes --> G[Replace child image with new program]
    G --> H[Load new code, data, and stack]
    F --> I{Parent waits?}
    G --> I
    I -- Yes --> J["Parent calls wait()"]
    I -- No --> K[Parent continues independently]
    J --> L[Child exits]
    L --> M[Parent reaps status]
```

`fork()` creates a new process; `exec()` replaces the current process image without creating another process. They are often used together to start a new program from a parent.

## 8. File Lookup and Open

```mermaid
flowchart TD
    A["User calls open('/etc/passwd')"] --> B[Kernel parses path components]
    B --> C[Resolve directories and locate final entry]
    C --> D[Find inode for the file]
    D --> E{Exists and permissions are valid?}
    E -- No --> F[Return ENOENT or EACCES]
    E -- Yes --> G[Create open-file description]
    G --> H[Add file descriptor to process table]
    H --> I[User reads or writes using the offset]
    I --> J[Return file descriptor to caller]
```

The path lookup flow includes directory traversal, inode lookup, validation, and descriptor creation. Caches and the VFS layer may hide lower-level device details.

## 9. I/O and Hardware Interaction

```mermaid
flowchart LR
    A[User process] --> B[System call]
    B --> C[Kernel driver layer]
    C --> D[Device command or DMA setup]
    D --> E[Hardware device]
    E --> F[Interrupt or completion signal]
    F --> G[Kernel handles result]
    G --> H[Process resumes]
```

Interrupt-driven I/O and DMA reduce CPU overhead. The kernel coordinates the transfer and only interrupts the CPU when the device is ready or the operation completes.
