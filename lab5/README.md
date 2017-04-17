# Lab5 System Calls and Scheduler

## there are some new files and modifacations
- new files
    - [`kernel`](nctuos/kernel)
        - [`sched.c`](nctuos/kernel/sched.c)
        - [`syscall.c`](nctuos/kernel/syscall.c)
        - [`syscall.h`](nctuos/kernel/syscall.h)
        - [`task.c`](nctuos/kernel/task.c)
        - [`task.h`](nctuos/kernel/task.h)
        - [`timer.h`](nctuos/kernel/timer.h)
    - [`inc`](nctuos/inc)
        - [`syscall.h`](nctuos/inc/syscall.h)
    - [`lib`](nctuos/lib)
        - [`console.c`](nctuos/lib/console.c)
        - [`printf.c`](nctuos/lib/printf.c)
        - [`readelf.c`](nctuos/lib/readelf.c)
        - [`syscall.c`](nctuos/lib/syscall.c)
    - [`user`](nctuos/user)
        - [`main.c`](nctuos/user/main.c)
        - [`shell.c`](nctuos/user/shell.c)
- modifications
    - [`kernel`](nctuos/kernel)
        - [`kern.ld`](nctuos/kernel/kern.ld)
        - [`kbd.c`](nctuos/kernel/kbd.c)
        - [`main.c`](nctuos/kernel/main.c)
        - [`mem.h`](nctuos/kernel/mem.h)
        - [`mem.c`](nctuos/kernel/mem.c)
        - [`printf.c`](nctuos/kernel/printf.c)
        - [`screen.c`](nctuos/kernel/screen.c)
        - [`timer.c`](nctuos/kernel/timer.c)
        - [`trap.c`](nctuos/kernel/trap.c)
        - [`trap.h`](nctuos/kernel/trap.h)
        - [`trap_entry.S`](nctuos/kernel/trap_entry.S)
        - [`entry.S`](nctuos/kernel/entry.S)
    - [`inc`](nctuos/inc)
        - [`mmu.h`](nctuos/inc/mmu.h)
        - [`shell.h`](nctuos/inc/shell.h)
        - [`stdio.h`](nctuos/inc/stdio.h)
        - [`trap.h`](nctuos/inc/trap.h)
    - [`lib`](nctuos/lib)
        - [`readline.c`](nctuos/lib/readline.c)
        - [`string.c`](nctuos/lib/string.c)

## Implement System Call
- [`kernel/syscall.c`](nctuos/kernel/syscall.c)
    - `syscall_init`
    ```c
    void syscall_init() {
        extern void SYSCALL();
        register_handler(T_SYSCALL, syscall_handler, SYSCALL, 1, 3);
    }
    ```
    - `syscall_handler`
    ```c
    static void syscall_handler(struct Trapframe *tf)
    {
        tf->tf_regs.reg_eax = do_syscall(
                tf->tf_regs.reg_eax,
                tf->tf_regs.reg_edx,
                tf->tf_regs.reg_ecx,
                tf->tf_regs.reg_ebx,
                tf->tf_regs.reg_edi,
                tf->tf_regs.reg_esi);
    }
    ```
    - `do_syscall`
    ```c
    extern int sys_fork();
    extern void sched_yield();
    extern void sys_kill();
    extern int32_t sys_get_num_used_page();
    extern int32_t sys_get_num_free_page();
    extern unsigned long sys_get_ticks();
    extern void sys_cls();
    extern void sys_settextcolor();
    int32_t do_syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5) {
        int32_t retVal = -1;
        extern Task *cur_task;
        switch (syscallno) {
            case SYS_fork:
                retVal = sys_fork();
                break;
            case SYS_getc:
                retVal = do_getc();
                break;
            case SYS_puts:
                do_puts((char*)a1, a2);
                retVal = 0;
                break;
           case SYS_getpid:
                retVal = cur_task->task_id;
                break;
            case SYS_sleep:
                cur_task->remind_ticks = a1;
                cur_task->state = TASK_SLEEP;
                sched_yield();
                break;
            case SYS_kill:
                sys_kill(cur_task->task_id);
                break;
            case SYS_get_num_free_page:
                retVal = sys_get_num_free_page();
                break;
            case SYS_get_num_used_page:
                retVal = sys_get_num_used_page();
                break;
            case SYS_get_ticks:
                retVal = sys_get_ticks();
                break;
            case SYS_settextcolor:
                sys_settextcolor(a1, a2);
                break;
            case SYS_cls:
                sys_cls();
                break;
        }
        return retVal;
    }
    ```
- [`lib/syscall.c`](nctuos/lis/syscall.c)
```c
int32_t get_num_used_page(void) {
    return syscall(SYS_get_num_used_page, 0, 0, 0, 0, 0);
}
int32_t cls(void) {
    return syscall(SYS_cls, 0, 0, 0, 0, 0);
}
int32_t get_num_free_page(void) {
    return syscall(SYS_get_num_free_page, 0, 0, 0, 0, 0);
}
unsigned long get_ticks(void) {
    return syscall(SYS_get_ticks, 0, 0, 0, 0, 0);
}
void settextcolor(unsigned char forecolor, unsigned char backcolor) {
    syscall(SYS_settextcolor, forecolor, backcolor, 0, 0, 0);
}
int32_t fork(void) {
    return syscall(SYS_fork, 0, 0, 0, 0, 0);
}
int32_t getpid(void) {
    return syscall(SYS_getpid, 0, 0, 0, 0, 0);
}
void kill_self() {
    syscall(SYS_kill, 0, 0, 0, 0, 0);
}
void sleep(uint32_t ticks) {
    syscall(SYS_sleep, ticks, 0, 0, 0, 0);
}
```
- [`kernel/trap_entry.S`](nctuos/kernel/trap_entry.S)
```asm
TRAPHANDLER_NOEC(SYSCALL, T_SYSCALL)
```

## Implement Task Schedule
- [`kernel/mem.c`](nctuos/kernel/mem.c)
    - `setupkvm`
    ```c
    pde_t *setupkvm() {
        struct PageInfo *pp = page_alloc(ALLOC_ZERO);
        pde_t *pgdir = NULL;
        if(pp){
            pgdir = page2kva(pp);
            boot_map_region(pgdir, UPAGES, ROUNDUP((sizeof(struct PageInfo) * npages), PGSIZE), PADDR(pages), PTE_U);
            boot_map_region(pgdir, KSTACKTOP - KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);
            boot_map_region(pgdir, KERNBASE, -KERNBASE, 0, PTE_W);
            boot_map_region(pgdir, IOPHYSMEM, ROUNDUP((EXTPHYSMEM - IOPHYSMEM), PGSIZE), IOPHYSMEM, PTE_W);
        }
        return pgdir;
    }
    ```
    - `page_alloc`, `page_free` need to maintain `num_free_pages`
- [`kernel/task.c`](nctuos/kernel/task.c)
    - `task_create`
    ```c
    int task_create() {
        Task *ts = NULL;
        for(int i = 0 ; i < NR_TASKS ; i++) {
            if(tasks[i].state == TASK_FREE) {
                ts = &tasks[i];
                break;
            }
        }
        if(!ts) return -1;
        if (!(ts->pgdir = setupkvm()))
            panic("Not enough memory for per process page directory!\n");
        for(int va = USTACKTOP ; va > USTACKTOP - USR_STACK_SIZE ; va -= PGSIZE) {
            struct PageInfo *pp = page_alloc(ALLOC_ZERO);
            if(!pp) return -1;
            if(page_insert(ts->pgdir, pp, va - PGSIZE, PTE_W | PTE_U) == -1)
                return -1;
        }
        memset( &(ts->tf), 0, sizeof(ts->tf));
        ts->tf.tf_cs = GD_UT | 0x03;
        ts->tf.tf_ds = GD_UD | 0x03;
        ts->tf.tf_es = GD_UD | 0x03;
        ts->tf.tf_ss = GD_UD | 0x03;
        ts->tf.tf_esp = USTACKTOP-PGSIZE;
        ts->task_id = ts - tasks;
        ts->parent_id = cur_task ? cur_task->task_id : 0;
        ts->state = TASK_RUNNABLE;
        ts->remind_ticks = TIME_QUANT;
        return ts - tasks;
    }
    ```
    - `task_free`
    ```c
    static void task_free(int pid) {
        lcr3(PADDR(kern_pgdir));
        for(int va = USTACKTOP ; va > USTACKTOP - USR_STACK_SIZE ; va -= PGSIZE) {
            page_remove(tasks[pid].pgdir, va - PGSIZE);
        }
        ptable_remove(tasks[pid].pgdir);
        pgdir_remove(tasks[pid].pgdir);
    }
    ```
    - `sys_kill`
    ```c
    void sys_kill(int pid) {
        if (pid > 0 && pid < NR_TASKS) {
            tasks[pid].state = TASK_FREE;
            task_free(pid);
            sched_yield();
        }
    }
    ```
    - `sys_fork`
    ```c
    int sys_fork() {
        int pid = task_create();
        if(pid == -1) return -1;
        if ((uint32_t)cur_task) {
            tasks[pid].tf = cur_task->tf;
            for(int va = USTACKTOP ; va > USTACKTOP - USR_STACK_SIZE ; va -= PGSIZE) {
                pte_t *pte_src, *pte_dst;
                if(!(pte_dst = pgdir_walk(tasks[pid].pgdir, va - PGSIZE, 0)))
                    panic("dst stack does not exist\n");
                if(!(*pte_dst & PTE_P))
                    panic("dst stack page does not present\n");
                if(!(pte_src = pgdir_walk(cur_task->pgdir, va - PGSIZE, 0)))
                    panic("src stack does not exist\n");
                if(!(*pte_src & PTE_P))
                    panic("src stack page does not present\n");
                memcpy(KADDR(PTE_ADDR(*pte_dst)), KADDR(PTE_ADDR(*pte_src)), PGSIZE);
            }
            setupvm(tasks[pid].pgdir, (uint32_t)UTEXT_start, UTEXT_SZ);
            setupvm(tasks[pid].pgdir, (uint32_t)UDATA_start, UDATA_SZ);
            setupvm(tasks[pid].pgdir, (uint32_t)UBSS_start, UBSS_SZ);
            setupvm(tasks[pid].pgdir, (uint32_t)URODATA_start, URODATA_SZ);
            cur_task->tf.tf_regs.reg_eax = pid;
            tasks[pid].tf.tf_regs.reg_eax = 0;
        }
        return pid;
    }
    ```
- [`kernel/sched.c`](nctuos/kernel/sched.c)
    - `sched_yield`
    ```c
    void sched_yield(void) {
        extern Task tasks[];
        extern Task *cur_task;
        int next_task_id = -1;
        for(int i = (cur_task - tasks + 1) % NR_TASKS ; i != (cur_task - tasks) ; i == NR_TASKS - 1 ? (i = 0) : (++i)) {
            if(tasks[i].state == TASK_RUNNABLE) {
                next_task_id = i;
                break;
            }
        }
        if(next_task_id == -1)
            next_task_id = cur_task - tasks;
        cur_task = &(tasks[next_task_id]);
        cur_task->remind_ticks = TIME_QUANT;
        cur_task->state = TASK_RUNNING;
        lcr3(PADDR(cur_task->pgdir));
        ctx_switch(cur_task);
    }
    ```
- [`kernel/timer.c`](nctuos/kernel/timer.c)
    - `time_handler`
    ```c
    void timer_handler(struct Trapframe *tf) {
        extern void sched_yield();
        int i;
        jiffies++;
        extern Task tasks[];
        extern Task *cur_task;
        if (cur_task != NULL) {
            for(int i = 0 ; i < NR_TASKS ; i++) {
                if(tasks[i].state == TASK_SLEEP) {
                    tasks[i].remind_ticks--;
                    if(tasks[i].remind_ticks <= 0)
                        tasks[i].state = TASK_RUNNABLE;
                }
            }
            cur_task->remind_ticks--;
            if(cur_task->remind_ticks <= 0) {
                cur_task->state = TASK_RUNNABLE;
                sched_yield();
            }
        }
    }
    ```
