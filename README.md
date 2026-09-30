# Operating Systems Notes

A concise study set for operating-system coding rounds, placement interviews, and practical systems work.

## Contents

- [OS coding patterns](Oscode.md): C and POSIX examples on safe input, memory allocation, scheduling, synchronization, processes, pipes, files, sockets, and Linux debugging.
- [OS notes](OSnotes.md): core concepts, interview-ready explanations, formulas, practice problems, and a revision checklist.
- [OS diagrams](diagram.md): Mermaid diagrams for process execution, scheduling, virtual memory, synchronization, and file lookup.

## Recommended Study Path

1. Review processes, threads, system calls, and context switches.
2. Practice CPU scheduling, synchronization, deadlocks, and page-replacement problems.
3. Use the [diagrams](diagram.md) to trace process execution, address translation, and synchronization flows.
4. Implement small programs using `fork`, `pipe`, `exec`, pthreads, and sockets.
5. Use Linux tools to inspect processes, memory, file descriptors, and system calls.
6. Finish with the practice questions and revision checklist in [OS notes](OSnotes.md).

## Working Safely

When writing OS-level C code, check every return value, handle partial reads and writes, close resources on every exit path, validate allocation sizes, and document ownership and synchronization assumptions clearly.