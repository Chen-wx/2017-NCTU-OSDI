# Lab4 Memory management

## there are some new files and modifacations
- new files
    - [`kernel/`](nctuos/kernel)
        - [`assert.c`](nctuos/kernel/assert.c)
        - [`entrypgdir.c`](nctuos/kernel/entrypgdir.c)
        - [`kclock.c`](nctuos/kernel/kclock.c)
        - [`kclock.h`](nctuos/kernel/kclock.h)
        - [`mem.c`](nctuos/kernel/mem.c)
        - [`mem.h`](nctuos/kernel/mem.h)
    - [`inc/`](nctuos/inc)
        - [`assert.h`](nctuos/inc/assert.h)
        - [`memlayout.h`](nctuos/inc/memlayout.h)
- modifacations
    - [`kernel/`](nctuos/kernel)
        - [`entry.S`](nctuos/kernel/entry.S): use new version
        - [`Makefile`](nctuos/kernel/Makefile): add new targets into `KERN_SRCFILES` and `KERN_OBJS`
        ```makefile
        KERN_SRCFILES := kernel/entry.S \
                kernel/main.c \
                kernel/picirq.c \
                kernel/kbd.c \
                kernel/screen.c \
                kernel/printf.c \
                kernel/mem.c \
                kernel/entrypgdir.c \
                kernel/assert.c \
                kernel/kclock.c \
                lib/printfmt.c \
                lib/string.c
        KERN_OBJS = kernel/entry.o \
                kernel/main.o \
                kernel/picirq.o \
                kernel/kbd.o \
                kernel/screen.o \
                kernel/trap.o \
                kernel/trap_entry.o \
                kernel/printf.o \
                kernel/mem.o \
                kernel/entrypgdir.o \
                kernel/assert.o \
                kernel/kclock.o \
                kernel/shell.o \
                kernel/timer.o \
                lib/printfmt.o \
                lib/readline.o \
                lib/string.o
        ```
        - [`kern.ld`](nctuos/kernel/kern.ld)
            - `kernel_load_addr = 0xF0100000`
            - text section
            ```
            .text : AT (0x100000) {
                *(.text .stub .text.* .gnu.linkonce.t.*)
            }
            ```
        - [`main.c`](nctuos/kernel/main.c)
            - add `mem_init();` into `main`
## 3.1 Physical Page Management
- [`kernel/mem.c`](nctuos/kernel/mem.c)
    - use `boot_alloc` to allocate memory for pages
    ```c
    pages = (struct PageInfo*)boot_alloc(sizeof(struct PageInfo) * npages);
    memset(pages, 0, sizeof(struct PageInfo) * npages);
    ```
    - finish `page_init()` according to the hints
    ```c
    page_free_list = NULL;
    for (int i = 0; i < npages; i++) {
        if(i == 0) {
            pages[i].pp_ref = 1;
            pages[i].pp_link = NULL;
        } else if(i < npages_basemem) {
            pages[i].pp_ref = 0;
            pages[i].pp_link = page_free_list;
            page_free_list = &pages[i];
        } else if(i < (EXTPHYSMEM / PGSIZE)) {
            pages[i].pp_ref = 1;
            pages[i].pp_link = NULL;
        } else {
            if(i < ((uint32_t)nextfree - KERNBASE) / PGSIZE) {
                pages[i].pp_ref = 1;
                pages[i].pp_link = NULL;
            } else {
                pages[i].pp_ref = 0;
                pages[i].pp_link = page_free_list;
                page_free_list = &pages[i];
            }
        }
    }
    ```
    - finish `page_alloc`
    ```c
    struct PageInfo * page_alloc(int alloc_flags)
    {
        if(!page_free_list) return NULL;
        struct PageInfo *pp = page_free_list;
        page_free_list = page_free_list->pp_link;
        if(alloc_flags & ALLOC_ZERO) {
            memset(page2kva(pp), 0, PGSIZE);
        }
        pp->pp_link = NULL;
        return pp;
    }
    ```
    - finish `page_free`
    ```c
    void page_free(struct PageInfo *pp)
    {
        assert(pp->pp_ref == 0);
        assert(pp->pp_link == NULL);
        pp->pp_link = page_free_list;
        page_free_list = pp;
    }
    ```

## 3.2 Page Table Management
- [`kernel/mem.c`](nctuos/kernel/mem.c)
    - finsih `pgdir_walk`
    ```c
    pte_t * pgdir_walk(pde_t *pgdir, const void *va, int create)
    {
        pde_t *pte = &pgdir[PDX(va)];
        pte_t *pte_kva;
        if(*pte & PTE_P) {
            pte_kva = KADDR(PTE_ADDR(*pte));
        } else {
            struct PageInfo *pp;
            if(!create || !(pp = page_alloc(ALLOC_ZERO))) return NULL;
            pp->pp_ref++;
            pte_kva = page2kva(pp);
            *pte = page2pa(pp) | PTE_P | PTE_U | PTE_W;
        }
        return &pte_kva[PTX(va)];
    }
    ```
    - finish `boot_map_region`
    ```c
    static void boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
    {
        while(size >= PGSIZE) {
            pte_t *pte = pgdir_walk(pgdir, va, 1);
            *pte = pa | perm | PTE_P;
            pa += PGSIZE;
            va += PGSIZE;
            size -= PGSIZE;
        }
    }
    ```
    - finish `page_lookup`
    ```c
    struct PageInfo * page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
    {
        pte_t *pte = pgdir_walk(pgdir, va, 0);
        if(!pte || !(*pte & PTE_P)) return NULL;
        if(*pte_store) *pte_store = pte;
        return pa2page(PTE_ADDR(*pte));
    }
    ```
    - finish `page_remove`
    ```c
    void page_remove(pde_t *pgdir, void *va)
    {
        pte_t *pte;
        struct PageInfo *pp = page_lookup(pgdir, va, &pte);
        if(!pp) return;
        page_decref(pp);
        *pte = 0;
        tlb_invalidate(pgdir, va);
    }
    ```
    - finish `page_insert`
    ```c
    int page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
    {
        pte_t *pte = pgdir_walk(pgdir, va, 1);
        if(!pte) return -E_NO_MEM;
        if(*pte & PTE_P) {
            if(PTE_ADDR(*pte) == page2pa(pp))
                goto PAGE_INSERT_SUCCESS;
            page_remove(pgdir, va);
        }
        pp->pp_ref++;
        tlb_invalidate(pgdir, va);
    PAGE_INSERT_SUCCESS:
        *pte = page2pa(pp) | PTE_P | perm;
        return 0;
    }
    ```

## 3.3 Update Page Directory
- [`kernel/mem.c`](nctuos/kernel/mem.c)
    - add `boot_map_region` for `mem_init()`
    ```c
    boot_map_region(kern_pgdir, KSTACKTOP - KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);
    boot_map_region(kern_pgdir, KERNBASE, -KERNBASE, 0, PTE_W);
    ```

## 3.4 Page Fault Handler 
- [`kernel/trap_entry.S`](nctuos/kernel/trap_entry.S)
    - add `TRAPHANDLER_NOEC(trap_page_fault, T_PGFLT)`
- [`kernel/trap.c`](nctuos/kernel/trap.c)
    - `trap_init()`: add `trap_page_fault` into `idt`
    ```c
    extern void trap_page_fault();
    SETGATE(idt[T_PGFLT], 0, GD_KT, trap_page_fault, 0);
    ```
    - add `page_fault_handler`
    ```c
    static void page_fault_handler(struct Trapframe *tf) {
        uint32_t fault_va = rcr2();
        cprintf("[0316320] Page fault @ %p", fault_va);
        while(1);
    }
    ```
    - modify `trap_dispatch`
        - add switch case
- [`kernel/main.c`](nctuos/kernel/main.c)
    - add page fault testing
    ```c
    int *ptr = 0x12345678;
    *ptr = 0;
    ```

