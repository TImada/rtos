## Solo5/spt porting on aarch32 Linux

This is initial implementation on 32-bit ARM Linux environments.

I only tested it with Ubuntu 18.04.4 LTS (GNU/Linux 5.3.0-1022-raspi2 armv7l) running on Raspberry Pi 3B.

#### How to try it? (experimental)

1. You must have a system employing one or more ARM CortexV7-A (or V8-A) architecture CPU cores with the NEON support.
2. Install a 32-bit Linux distribution (Ubuntu 18.04.x LTS is recommended) on the system.
3. Install opam, the latest ocaml compiler and other packages  (such as gcc, make) required to compile Solo5/spt related packages.  
I used gcc 7.5 (Ubuntu/Linaro 7.5.0-3ubuntu1~18.04) as the default gcc.
4. Checkout the aarch32-porting branch of solo5, ocaml-freestanding, and mirage-solo5  
```
(Get solo5 code)
$ git clone https://github.com/TImada/solo5
$ cd solo5
$ git checkout -b aarch32-porting origin/aarch32-porting
```
```
(Get ocaml-freestanding code)
$ git clone https://github.com/TImada/ocaml-freestanding
$ cd ocaml-freestanding
$ git checkout -b aarch32-porting origin/aarch32-porting
```
```
(Get mirage-solo5 code)
$ git clone https://github.com/TImada/mirage-solo5
$ cd mirage-solo5
$ git checkout -b aarch32-porting origin/aarch32-porting
```
5. Pin each directory to a corresponding opam package
```
(solo5 pinning)
$ opam pin solo5-bindings-spt /path/to/solo5
```
```
(ocaml-freestanding pinning)
$ opam pin ocaml-freestanding /path/to/ocaml-freestanding
```
```
(mirage-solo5 pinning)
$ opam pin mirage-solo5 /path/to/mirage-solo5
```

6. Try Hello Mirage World!
```
$ git clone https://github.com/mirage/mirage-skeleton
$ cd /path/to/mirage-skeleton/tutorial/hello
$ mirage configure -t spt
$ make depend
$ make
$ solo5-spt --mem=16 ./hello.spt
            |      ___|
  __|  _ \  |  _ \ __ \
\__ \ (   | | (   |  ) |
____/\___/ _|\___/____/
Solo5: Bindings version v0.6.4-7-g0868bac
Solo5: Memory map: 16 MB addressable:
Solo5:   reserved @ (0x0 - 0xfffff)
Solo5:       text @ (0x100000 - 0x191fff)
Solo5:     rodata @ (0x192000 - 0x1b4fff)
Solo5:       data @ (0x1b5000 - 0x21cfff)
Solo5:       heap >= 0x21d000 < stack < 0x1000000
2020-04-13 15:37:03 -00:00: INF [application] hello
2020-04-13 15:37:04 -00:00: INF [application] hello
2020-04-13 15:37:05 -00:00: INF [application] hello
2020-04-13 15:37:06 -00:00: INF [application] hello
Solo5: solo5_exit(0) called
```

#### Remarkable changes
- solo5-bindings-spt
  - added a library named `libaeabi` which provides `__aeabi_**` builtin functions(this library can be found in `./lib/aeabi/`)
  - introduced `__arm__` macro to specify 32-bit ARM architecture
  - introduced `__BITS_32__` and `__BITS_64__` to represent 32 or 64 bits architecture
  - bindings/mem.c was converted to mem32.c and mem64.c to handle 32/64-bit architecture
  - tenders/common/elf.c was converted to elf32.c and elf64.c to handle 32/64-bit architecture
  - added bindings/cpu\_arm.h
  - added bindings/spt/sys\_linux\_arm.c
- ocaml-freestanding
  - added config.in/Makefile.Linux.arm and config.in/m.arm.h
- mirage-solo5
 - nothing especially

#### Issues, workaround, and etc ...
- `printf()` and some other functions require arithmetic operations such as `__aeabi_uldivmod()`. They are implemented in libaeabi.
- aarch32 gcc tries to use the TPIDRURO register rather than the TPIDRURW register for Thread Local Storage. This does not allow a user program to change the TPIDRURO register directly. So I employed the `-mtp=soft` option in MAKECONF\_CFLAGS so that gcc calls a user defined `__aeabi_read_tp()` function to manipulate the TPIDRURW  register. `__aeabi_read_tp()` is implemented in libaeabi too.
- aarch32 ld tries to locate parts of the spt tender program on memory address lower than 0x200000. This must be avoided because a \*.spt program should be located at such memory address. So I employed the `-Ttext-segment=0x40000000` option in HOSTLDFLAGS.
- I encountered a gcc bug in test\_fpu.c by which the validity check of vector multiply result cannot work correctly. An additional inline assembly section with repeated nop instructions was inserted to avoid the bug.
```
#elif defined(__arm__)
...
...
...
    /* TODO: This is a workaround for arm-linux-gnueabihf-gcc */
    int i;
    for (i = 0; i < 6; i++) {
        __asm__(
            "nop\n"
            :
            :
            :
        );
    }
```
