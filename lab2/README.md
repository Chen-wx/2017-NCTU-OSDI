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
