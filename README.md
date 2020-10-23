# Process startup

- `fork` creëert [`struct proc`][struct proc]
    - En andere dingen die later besproken worden
    - Verwijs naar velden die studenten zouden moeten begrijpen: `pid`, `parent`, `ofile` (indexed via fd), (`pagetable`?)
- `exec` laadt een ELF bestand in het geheugen
    - oefening: inspecteer ELF file met `readelf`, `objdump`,...
- `exec()` roept ELF entry point op met `argc` en `argv` als argumenten
    - oefening: vind entry point met `readelf`
    - oefening: implementeer `crt0.c` die `main` oproept en dan `exit`
        - voeg user/crt0.c toe
        - voeg `$U/crt0.o` toe aan [`ULIB`][ULIB]
        - implementeer `void _start(int argc, char* argv[])` in user/crt0.c
        - pas het entry point (`-e` flag) aan dat aan [`ld` wordt gegeven][ld rule]
        - **REMOVE** [solution](https://github.com/besturingssystemen/xv6-solutions/commit/eb21960e6323efedde447ba39df5c6b7d831e409)

# System calls

- RISC-V calling convention
    - oefening: hello world in assembly (zonder syscalls: call puts, ret from main werkt met crt0)
        - voeg user/hello_asm_puts.S toe met contents (TODO: meer/minder hulp? welke uitleg bij deze contents?):
            ```asm
            .text
            .global main
            main:

            .section .rodata
            hello_str:
                .asciz "Hello, world!"
            ```
        - voeg `$U/_hello_asm_puts\` aan [`UPROGS`][UPROGS]
        - implementeer `main` (denk aan bewaren `ra`!)
        - **REMOVE** [solution](https://github.com/besturingssystemen/xv6-solutions/commit/489fa639b46ab2168e4c7abe9d7d82d650c4b67b)
- xv6 system call convention
    - oefening: hello world in assembly via `write`
        - ongeveer zelfde begin als vorige oefening maar gebruik user/hello_asm_write.S
            ```asm
            #include "kernel/syscall.h"

            .text
            .global main
            main:

            .section .rodata
            hello_str:
                .ascii "Hello, world!\n"
            ```
        - **REMOVE** [solution](https://github.com/besturingssystemen/xv6-solutions/commit/db8241a6803334b7d225017b400d848ba31204e0)
- xv6 system call dispatch code
    - [`syscall`][syscall]
      Vanaf hier zou de syscall code begrijpbaar moeten zijn.
      We houden de eerdere code (traphandler enzo) voor de scheduling oefenzitting.
      We moeten enkel uitleggen wat trapframe precies is.
    - [`getpid`][sys_getpid] als voorbeeld (eenvoudigste syscall)
- system call toevoegen: `getnumsyscalls`
    - voeg `numsyscalls` toe aan [`struct proc`][struct proc]
    - initialiseer `numsyscalls` in [`allocproc`][allocproc]
    - verhoog `numsyscalls` in [`syscall`][syscall]
    - voeg `SYS_getnumsyscalls` toe aan [kernel/syscall.h][syscall.h]
    - implementeer `sys_getnumsyscalls` in [kernel/sysproc.c][sysproc.c]
    - dispatch `sys_getnumsyscalls` in [`syscall`][syscall]
- system call gebruiken in user space
    - voeg `int getnumsyscalls()` toe aan [user/user.h][user.h]
    - voeg `entry("getnumsyscalls")` toe aan [user/usys.pl][usys.pl] (leg uit dat dit script gewoon een assembly bestand genereerd)
    - gebruik die nieuwe system call in een user space programma

# Permanente evaluatie

- Voeg een nieuwe system call `void traceme(int fd)` toe die ervoor zorgt dat elke system call van het proces geprint wordt naar `fd` samen met de pid van het proces. (`{pid}: syscall {num}`)
  (TODO kunnen we vanuit kernel space gemakkelijk naar een fd schrijven?)
- Maak een user space programma `traceme` dat een executable als argument krijgt en deze executable oproept met de `traceme` functionaliteit aangezet. (`fork`, `traceme`, `exec`)


[struct proc]: https://github.com/besturingssystemen/xv6-riscv/blob/280d2aa694114e7a6e7eb2a9c4f62e3c314983c6/kernel/proc.h#L86
[syscall]: https://github.com/besturingssystemen/xv6-riscv/blob/280d2aa694114e7a6e7eb2a9c4f62e3c314983c6/kernel/syscall.c#L133
[syscall.h]: https://github.com/besturingssystemen/xv6-riscv/blob/280d2aa694114e7a6e7eb2a9c4f62e3c314983c6/kernel/syscall.h
[allocproc]: https://github.com/besturingssystemen/xv6-riscv/blob/280d2aa694114e7a6e7eb2a9c4f62e3c314983c6/kernel/proc.c#L100
[sys_getpid]: https://github.com/besturingssystemen/xv6-riscv/blob/280d2aa694114e7a6e7eb2a9c4f62e3c314983c6/kernel/sysproc.c#L21
[sysproc.c]: https://github.com/besturingssystemen/xv6-riscv/blob/280d2aa694114e7a6e7eb2a9c4f62e3c314983c6/kernel/sysproc.c
[user.h]: https://github.com/besturingssystemen/xv6-riscv/blob/280d2aa694114e7a6e7eb2a9c4f62e3c314983c6/user/user.h
[usys.pl]: https://github.com/besturingssystemen/xv6-riscv/blob/280d2aa694114e7a6e7eb2a9c4f62e3c314983c6/user/usys.pl
[ULIB]: https://github.com/besturingssystemen/xv6-riscv/blob/2baca184bce1e0d11f55460a6b8ec0c260f08a10/Makefile#L92
[ld rule]: https://github.com/besturingssystemen/xv6-riscv/blob/2baca184bce1e0d11f55460a6b8ec0c260f08a10/Makefile#L95
[UPROGS]: https://github.com/besturingssystemen/xv6-riscv/blob/4edbcfd22ccadded04c36aeef8872c5ab4a92f28/Makefile#L120
