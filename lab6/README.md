# Lab6 Symmetric Multiprocessing

## there are some new files and modifacations
- new files
    - [`kernel`](nctuos/kernel)
        - [`cpu.h`](nctuos/kernel/cpu.h)
        - [`lapic.c`](nctuos/kernel/lapic.c)
        - [`mpconfig.c`](nctuos/kernel/mpconfig.c)
        - [`mpentry.S`](nctuos/kernel/mpentry.S)
        - [`spinlock.c`](nctuos/kernel/spinlock.c)
        - [`spinlock.h`](nctuos/kernel/spinlock.h)
- modifications
    - [`kernel`](nctuos/kernel)
        - [`mem.c`](nctuos/kernel/mem.c)
        - [`main.c`](nctuos/kernel/main.c)
        - [`task.h`](nctuos/kernel/task.h)
        - [`task.c`](nctuos/kernel/task.c)
        - [`timer.c`](nctuos/kernel/timer.c)
        - [`syscall.c`](nctuos/kernel/syscall.c)
        - [`sched.c`](nctuos/kernel/sched.c)
    - [`inc`](nctuos/inc)
        - [`memlayout.h`](nctuos/inc/memlayout.h)
        - [`syscall.h`](nctuos/inc/syscall.h)
    - [`lib`](nctuos/lib)

## Implement System Call
