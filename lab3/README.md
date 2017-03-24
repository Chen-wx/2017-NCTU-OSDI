# Lab3 X86 I/O System and Interrupt

## if using a 64-bit system, you have to do something first
- install cross compiler
    - apt install gcc-multilib 
- add some flags for Makefile
    - [`Makefile`](nctuos/Makefile)
        - move `include boot/Makefile` and `include kernel/Makefile` to the end
        - add `-fno-stack-protector` to `CFLAGS`
        - add `LDFLAGS = -m elf_i386`
        - add '-rf' flag to `rm` in clean
    - [`kernel/Makefile`](nctuos/kernel/Makefile)
        - change `KERNLDFLAGS` = `$(LDFLAGS) -T kernel/kern.ld`

## Setup GDT
- [`boot/boot.S`](nctuoj/boot/boot.S)
    - fill up `gdt`
        - code segment
            - executable & readable
            - base = 0
            - limit = 0x7FF ((2047 + 1) * 4096 = 8MB)
        - data segment
            - writable
            - base = 0
            - limit = 0x7FF ((2047 + 1) * 4096 = 8MB)
        - data segment
    ```asm
    gdt:
        SEG_NULL
        SEG(STA_X | STA_R, 0, 0x7FF << 12)
        SEG(STA_W, 0, 0x7FF << 12)
    ```
    - fill up `gdtdesc`
        - limit = `gdtdesc - gdt - 1` (`sizeof(gdt) - 1`)
        - base = `gdt`
            - change the type of gdt base from `.word` to `.long` (important)
    ```asm
    gdtdesc:
        .word gdtdesc - gdt - 1
        .long gdt
    ```

## Setup IDT
- [`kernel/trap_entry.S`](nctuos/kernel/trap_entry.S)
    - use `TRAPHANDLER_NOEC` to declare ISR for kbd & timer
    ```asm
    TRAPHANDLER_NOEC(irq_kbd, IRQ_OFFSET + IRQ_KBD)
    TRAPHANDLER_NOEC(irq_timer, IRQ_OFFSET + IRQ_TIMER)
    ```
    - in `_alltraps`, push regs into stack for `Trapframe` before call `default_trap_handler` and pop after calling
    ```asm
    _alltraps:
        pushl %ds
        pushl %es
        pushal
        pushl %esp
        call default_trap_handler
        popl %esp
        popal
        popl %es
        popl %ds
        add $8, %esp 
        iret 
    ```
- [`kernel/trap.c`](kernel/trap.c)
    - declare idt table and idtr structure
    ```c
    struct Gatedesc idt[256];
    struct Pseudodesc idt_pd = {
        .pd_lim = (uint16_t)(sizeof(idt) - 1),
        .pd_base = (uint32_t)idt
    };
    ```
    - `trap_init`
        - use SETGATE to fill up idt table
    ```c
    void trap_init(){
        extern void irq_kbd();
        SETGATE(idt[IRQ_OFFSET + IRQ_KBD], 0, GD_KT, irq_kbd, 0);
        extern void irq_timer();
        SETGATE(idt[IRQ_OFFSET + IRQ_TIMER], 0, GD_KT, irq_timer, 0);
        lidt(&idt_pd);
    }
    ```
    - `trap_dispatch`
        - call the right interupt handler according to the trap number
    ```c
    static void trap_dispatch(struct Trapframe *tf){
        extern void kbd_intr();
        extern void timer_handler();
        switch (tf->tf_trapno){
            case IRQ_OFFSET + IRQ_KBD:
                kbd_intr();
                break;
            case IRQ_OFFSET + IRQ_TIMER:
                timer_handler();
                break;
            default:
                print_trapframe(tf);
                break;
        }
    }
    ```
- [`kernel/main.c`](nctuos/kernel/main.c)
    - in `kernel_main`, uncomment `kbd_init()`, `timer_init()` and `trap_init()`

## add `kerninfo` command
- [`kernel/kern.ld`](nctuos/kernel/kern.ld)
    - add `PROVIDE(data = .)` before data segment to indicate the start address of data segment
- [`kernel/shell.c`](nctuos/kernel/shell.c)
    - `mon_kerninfo`
        - use the symbols in `kernel/kern.ld`
    ```c
    int mon_kerninfo(int argc, char **argv)
    {
        extern char kernel_load_addr[], etext[], data[], end[];
        cprintf("Kernel code base start=0x%x size=%d\n", kernel_load_addr, etext - kernel_load_addr);
        cprintf("Kernel data base start=0x%x size=%d\n", data, end - data);
        cprintf("Kernel executable memory footprint: %dKB\n", (end - kernel_load_addr) >> 10);
        return 0;
    }
    ```
## add `chgcolor` command
- [`inc/shell.h`](nctuos/inc/shell.h)
    - add function declaration
    ```c
    int chgcolor(int argc, char **argv);
    ```
- [`kernel/shell.c`](nctuos/kernel/shell.c)
    - add an entry to `commands` for `chgcolor`
    ```c
    static struct Command commands[] = {
        { "help", "Display this list of commands", mon_help },
        { "kerninfo", "Display information about the kernel", mon_kerninfo },
        { "print_tick", "Display system tick", print_tick },
        { "chgcolor", "Change text color", chgcolor}
    };
    ```
    - add `chgcolor` function
    ```c
    int chgcolor(int argc, char **argv){
        if(argc < 2){
            cprintf("Need argument.")   ;
            return 1;
        }
        int color = argv[1][0] - '0';
        settextcolor(color, 0);
        cprintf("Change color %d!\n", color);
        return 0;
    }
    ```
