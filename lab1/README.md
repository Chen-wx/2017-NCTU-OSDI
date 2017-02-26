# Lab1 Establish Lab Environment

## download [osdi.img](http://grass8.cs.nctu.edu.tw/OSDI2015/lab1/osdi.img) from http://grass8.cs.nctu.edu.tw/OSDI2015/lab1/osdi.img

Build New Version Qemu(for Ubuntu)
- git clone git://git.qemu-project.org/qemu.git
- cd qemu
- git submodule update --init pixman dtc
- sudo apt install libncursesw5-dev
- mkdir build && cd build
- ./../configure --enable-curses
- make
- sudo make install

Bugs
- `Makefile`: missing separator => need to add a tab
- `include/string.h`: move function implementation to `lib/string.c`
- `kernel/blk_dev/floppy.c`: add `static` to `setup_rw_floppy`
- `fs/buffer.c`: add `static` to `invalidate_buffer`
- `/tools/build.sh`: add execute mode to this file
- `init/main.c`: remove `panic("")` on line 140.
- `include/linux/sched.h`: set `NR_TASKS` to 64

Modification
- `init/main.c`: add `printf("Hello 0316320\n")` on line 192.

Qemu
- open emulator gui
    - `qemu-system-i386 -boot a -m 16M -gdb tcp::1234 -S -fda Image -hda ../osdi.img`
- open in console
    - `qemu-system-i386 -boot a -m 16M -gdb tcp::1234 -S -fda Image -hda ../osdi.img -nographic -curses`
