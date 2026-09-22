# Operating Systems Notes

Study material for operating-systems coding rounds, placement interviews, and online assessments.

## Contents

- [OS coding patterns](Oscode.md): C and POSIX examples covering safe input, memory allocation, scheduling, synchronization, processes, pipes, files, sockets, and Linux debugging.
- [OS notes](OSnotes.md): core concepts, interview questions, formulas, practice problems, and a final revision checklist.

## Suggested Study Path

1. Review processes, threads, system calls, and context switches.
2. Practice scheduling, synchronization, deadlocks, and page replacement problems.
3. Implement small programs using `fork`, `pipe`, `exec`, pthreads, and sockets.
4. Use the Linux toolkit to inspect processes, memory, file descriptors, and system calls.
5. Finish with the practice problems and revision checklist in [OS notes](OSnotes.md).

## Working Safely

When writing OS-level C code, check return values, handle partial reads and writes, close resources on every exit path, validate allocation sizes, and state ownership and synchronization assumptions explicitly.