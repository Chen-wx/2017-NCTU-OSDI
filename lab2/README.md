# Lab2: Kernel Booting

## Lab 2.1
- add `printf("THIS IS OSDI LAB2!!!\n")` to `seabios/src/bootsplash.c` on line 50.
- use different bios
    - `-bios /path/to/bios.bin`
- show bootsplash image
    - show boot menu
        - `-boot menu=on`
    - modify boot menu wait time(ms)
        - `-boot splash-time=10000`
    - specify bootsplash image
        - `-boot splash=/path/to/bootsplash.jpg`
        - `-fw_cfg bootsplash.jpg,file=/path/to/bootsplash.jpg`

## Lab 2.2 & 2.3
- add [`boot/hello.s`](linux-0.11/boot/hello.s)
```asm
    .code16
    .global _start
    .text
_start:
    mov %cs, %ax
    mov %ax, %ds
    mov %ax, %es

    mov $0x03, %ah      # read cursor pos
    xor %bh, %bh
    int $0x10

    mov $24, %cx
    mov $0x0007, %bx        # page 0, attribute 7 (normal)
    #lea    msg1, %bp
    mov     $msg1, %bp
    mov $0x1301, %ax        # write string, move cursor
    int $0x10
end_hello:
    jmp end_hello

msg1:
    .byte 13,10
    .ascii "Hello OSDI Lab2!"
    .byte 13,10
```
- modify [`boot/bootsect.s`](linux-0.11/boot/bootsect.s)
    - line 42. ~ 43. 
    ```asm
            .equ HELLOSEG, 0x0100       # hello starts here
            .equ HELLOLEN, 1            # nr of hello-sectors
    ```
    - line 67. ~ 95. 
    ```asm
    multiboot:
        mov $0x03, %ah          # read cursor pos
        xor %bh, %bh
        int $0x10

        mov $34, %cx
        mov $0x0007, %bx        # page 0, attribute 7 (normal)
        mov $msg_multiboot, %bp
        mov $0x1301, %ax        # write string, move cursor
        int $0x10

        mov $0x0, %ah           # set function
        int $0x16               # read key press
        cmp $0x31, %al          # if 1 then jump to load_setup
        je load_setup
        cmp $0x32, %al          # if 2 then jump to load_hello
        je load_hello

        # show error msg
        mov $0x03, %ah          # read cursor pos
        xor %bh, %bh
        int $0x10

        mov $29, %cx
        mov $0x0007, %bx        # page 0, attribute 7 (normal)
        mov $msg_multiboot_error, %bp
        mov $0x1301, %ax        # write string, move cursor
        int $0x10
        jmp multiboot           # jump to multiboot again
    ```
    - line 98. ~ 113. 
    ```asm
    load_hello:
        mov $HELLOSEG, %ax
        mov %ax, %es            # set es
        mov $0x0000, %dx        # drive 0, head 0
        mov $0x0002, %cx        # sector 2, track 0
        mov $0x0000, %bx        # address = 0, in HELLOSEG
        .equ AX, 0x0200+HELLOLEN
        mov $AX, %ax            # service 2, nr of sectors
        int $0x13               # read it
        jnc ok_load_hello       # ok - continue
        mov $0x0000, %dx
        mov $0x0000, %ax        # reset the diskette
        int $0x13
        jmp load_hello          # read again
    ok_load_hello:
        ljmp $HELLOSEG, $0      # jump to hello
    ```
    - line 118. ~ 131. 
    ```asm
    load_setup:
        mov $INITSEG, %ax
        mov %ax, %es                    # set es
        mov     $0x0000, %dx            # drive 0, head 0
        mov     $0x0003, %cx            # sector 3, track 0
        mov     $0x0200, %bx            # address = 512, in INITSEG
        .equ    AX, 0x0200+SETUPLEN
        mov     $AX, %ax                # service 2, nr of sectors
        int     $0x13                   # read it
        jnc     ok_load_setup           # ok - continue
        mov     $0x0000, %dx
        mov     $0x0000, %ax            # reset the diskette
        int     $0x13
        jmp     load_setup
    ```
    - line 202.
    ```asm
    sread:	.word 2 + SETUPLEN	# sectors read of current track
    ```
    - line 304. ~ 314.
    ```asm
    msg_multiboot:
        .byte 13, 10
        .ascii "[1] Load setup"
        .byte 13, 10
        .ascii "[2] Load hello"
        .byte 13, 10

    msg_multiboot_error:
        .byte 13, 10
        .ascii "You can only press 1 or 2"
        .byte 13, 10
    ```
- modify [`boot/Makefile`](linux-0.11/boot/Makefile)
    - line 18. ~ 26. 
    ```Makefile
    hello: hello.o
        @$(LD) $(LDFLAGS) -o hello hello.o
        @objcopy -R .pdr -R .comment -R.note -S -O binary hello

    hello.o: hello.s
        @$(AS) -o hello.o hello.s

    clean:
        @rm -f bootsect bootsect.o setup setup.o head.o hello.o hello
    ```
- modify [`Makefile`](linux-0.11/Makefile)
    - line 45. ~ 52.
    ```Makefile
    Image: boot/bootsect boot/setup tools/system boot/hello
        @cp -f tools/system system.tmp
        @strip system.tmp
        @objcopy -O binary -R .note -R .comment system.tmp tools/kernel
        @tools/build.sh boot/bootsect boot/setup tools/kernel boot/hello Image $(ROOT_DEV)
        @rm system.tmp
        @rm tools/kernel -f
        @sync
    ```
    - line 94. ~ 95.
    ```Makefile
    boot/hello: boot/hello.s
        @make hello -C boot
    ```
    - line 102. ~ 105. 
    ```Makefile
    clean:
        @rm -f Image System.map tmp_make core boot/bootsect boot/setup boot/hello
        @rm -f init/*.o tools/system boot/*.o typescript* info bochsout.txt
        @for i in mm fs kernel lib boot; do make clean -C $$i; done
    ```
- modify [`tools/build.sh`](linux-0.11/tools/build.sh)
    - line 6. ~ 11.
    ```bash
    bootsect=$1
    setup=$2
    system=$3
    hello=$4
    IMAGE=$5
    root_dev=$6
    ```
    - line 30. ~ 32.  
    ```bash
    # Write hello (512 bytes, one sector) to stdout
    [ ! -f "$hello" ] && echo "there is no hello binary file there" && exit -1
    dd if=$hello seek=1 bs=512 count=1 of=$IMAGE 2>&1 >/dev/null
    ```
    - line 36.
    ```bash
    dd if=$setup seek=2 bs=512 count=4 of=$IMAGE 2>&1 >/dev/null
    ```
    - line 42. 
    ```bash
    dd if=$system seek=6 bs=512 count=$((2888-1-4)) of=$IMAGE 2>&1 >/dev/null
    ```
