In deze oefenzitting leren jullie over de werking van system calls.

- [Voorbereiding](#voorbereiding)
- [GitHub classroom](#github-classroom)
- [Introductie](#introductie)
- [Levenscyclus proces](#levenscyclus-proces)
  - [Aanmaak processen](#aanmaak-processen)
    - [De `fork` system call](#de-fork-system-call)
    - [Process state](#process-state)
    - [Executable Files](#executable-files)
    - [ELF in Linux](#elf-in-linux)
    - [ELF in xv6](#elf-in-xv6)
    - [Application Binary Interface (ABI)](#application-binary-interface-abi)
    - [De `exec` system call](#de-exec-system-call)
    - [C runtime (crt0)](#c-runtime-crt0)
- [System calls](#system-calls)
  - [System calls vs function calls](#system-calls-vs-function-calls)
    - [RISC-V assembly](#risc-v-assembly)
  - [Scratchpad](#scratchpad)
- [Permanente evaluatie](#permanente-evaluatie)

# Voorbereiding

Ter voorbereiding van deze oefenzitting wordt je verwacht:

* De oefenzitting [os interfaces](https://github.com/besturingssystemen/os-interfaces) te hebben voltooid.
* Hoofdstuk 2 van het [xv6 boek](https://github.com/mit-pdos/xv6-riscv-book/) te hebben gelezen.

# GitHub classroom

* **TODO** Instructies geven (nieuwe repo's vanuit eigen fork?)

# Introductie

In deze sessie zullen we system calls in detail bestuderen.
In het eerste deel kijken we naar de werking van enkele bestaande system calls gerelateerd aan de levenscyclus van een proces.
Vervolgens leren we hoe we zelf een system call kunnen toevoegen aan xv6.
Ten slotte vragen we jullie zelfstandig, als permanente evaluatie, een system call toe te voegen.

# Levenscyclus proces

Een proces is een abstractie op niveau van het besturingssysteem. 
Besturingssystemen zijn verantwoordelijk voor de aanmaak, het beheer en de correcte afsluiting van processen.

De proces-abstractie is enorm krachtig.
Ze laat ons toe om complexe systemen op te bouwen als een verzameling programma's die elk een eigen specifieke rollen vervullen.
Deze programma's kunnen dankzij de procesabstractie in parallel worden uitgevoerd, geïsoleerd van elkaar met elk een eigen geheugenruimte.

## Aanmaak processen

### De `fork` system call

In [UNIX](https://en.wikipedia.org/wiki/Unix) (Linux, xv6, ...) besturingssystemen worden processen aangemaakt door middel van de `fork` system call.
Deze system call maakt een kopie van het huidige proces.

### Process state

In detail begrijpen hoe een proces gekopieerd wordt, is op dit punt in de oefenzittingen nog te vroeg. We kunnen wel al tonen welke state een besturingssysteem bewaart per proces.

* Bekijk [struct proc][struct proc] gedefinieerd in `proc.h` in xv6.
* Lees de comments bij elk veld van de `struct`. 
  
Het nut van de velden `state`, `parent`, `killed`, `pid`, `sz`, `ofile`, `cwd` en `name` zou duidelijk moeten zijn. Vraag verduidelijking aan een assistent indien dit niet het geval is. 

> :bulb: Een file descriptor (int fd) indexeert de `ofile` array. Wanneer we dus met de `write` system call schrijven naar een bestand geven we als eerste argument aan `write` een index mee in de tabel met open bestanden van het proces.

De velden `lock`, `chan` en `xstate` hebben te maken met synchronizatie en worden in een latere oefenzitting in detail bekeken.

Om velden `kstack`, `pagetable` en `trapframe` te begrijpen hebben we kennis nodig van virtual memory, een concept dat we in de volgende oefenzitting zullen bekijken.

Het veld `context` zal duidelijk worden door te scrollen naar het begin van het `proc.h` bestand.
Je kan daar de definitie van `struct context` terugvinden.
Wanneer een proces de controle doorgeeft aan het besturingssystem, onder andere bij het uitvoeren van een *system call*, noemen we dit een *context switch* van het proces naar de kernel.

Het veld `context` bevat een struct waarin alle nodige registerwaarden van het proces bewaard kunnen worden bij het uitvoeren van een context switch. 

### Executable Files

De `fork` system call zal als een van zijn taken de process state kopiëren voor het nieuwe proces. Nadat een proces gekopieerd is, is het in vele gevallen de bedoeling dat het nieuwe proces een eigen taak toegewezen krijgt.
De meest gangbare manier om het proces een nieuwe taak toe te wijzen, maakt gebruik van de systeemoproep `exec`.

Om `exec` te begrijpen is het belangrijk om eerst te snappen wat we bedoelen wanneer we spreken over een *executable*.
Een executable file of uitvoerbaar bestand bevat de volledige informatie, onder andere de machinecode en data, die nodig is om een bepaald programma uit te voeren.

Executables worden meestal gegenereerd door een *linker*. Met behulp van *compilers* worden broncode-bestanden omgezet in *object files*. De linker neemt als input verschillende object-files, verbindt (linkt) deze met elkaar en genereert vervolgens een executable als output.

In UNIX volgen `executables` het [`ELF`](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)-formaat.
Onderstaande afbeelding (bron: [Wikipedia](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format#/media/File:ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png)) illustreert de opbouw van een ELF-bestand:

![Opbouw ELF-bestand](https://upload.wikimedia.org/wikipedia/commons/e/e4/ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png)

In `ELF`-bestanden worden programma's opgedeeld in verschillende secties. Enkele veelvoorkende secties:

* `.text` bevat de machinecode van het programma
* `.data` bevat de initiële data die bij opstart van het proces geïnitialiseerd moet worden in het geheugen. Geïnitialiseerde global variables horen in de data-sectie
* `.bss` bevat globale data die niet geïnitialiseerd moet worden. Gedeclareerde maar niet-geïnitialiseerde globals horen in de bss-sectie
* `.rodata` bevat read-only data. Hierin worden vaak gedefinieerde strings geplaatst.
* ...

Elk van deze secties moeten in het geheugen geladen worden om een proces op te starten.
Een *loader* neemt als invoer een ELF-bestand en laadt de verschillende secties in het geheugen.
Een loader zet dus als het ware een programma, beschreven in een ELF-bestand, om in een proces dat kan uitvoeren binnen een besturingssysteem.

Naast secties bevat een ELF-bestand ook een *symbol table*. De symbol table bevat informatie over de inhoud van een ELF-bestand, bijvoorbeeld welke functies gedefinieerd zijn in het ELF-bestand en op welke (relatieve) locatie je deze kan terugvinden.

> :bulb: Een proces is niet hetzelfde als een programma. Beide woorden hebben een verschillende betekenis. Een programma beschouwen is de abstracte voorstelling van een taak die door een machine uitgevoerd kan worden. Een ELF-bestand of executable beschrijft een programma. Een proces bevat een instantie van een programma en kan geïnitialiseerd worden met behulp van een executable. Processen zijn de structuren die het mogelijk maken voor een besturingssysteem om programma's uit te voeren.

### ELF in Linux

Om ELF-bestanden te leren kennen zullen we deze eerst bekijken in onze eigen Linux-omgeving (dus niet in xv6):

* Schrijf een C-programma `hello-world.c` binnen je Linux-omgeving

```c
#include <stdio.h>

int main(){
    printf("Hello, world!\n");
    return 0;
}
```

* Compileer en link het programma met `gcc`

```shell
gcc hello-world.c -o hello-world
```
* Bekijk alle secties in het gegenereerde ELF-bestand met behulp van `readelf`

```shell
readelf -S hello-world
```

* Bekijk alle symbolen in het generereerde ELF-bestand met behulp van `readelf`

```shell
readelf -s hello-world
```

* Bekijk de ELF-header met `readelf`. 
```shell
readelf -h hello-world | less
```

* Gebruik het programma `objdump` om de uitvoerbare ELF-secties met machinecode in hello-world te *disassemblen* (omzetten van machinetaal naar leesbare assembly). Met <kbd>↑</kbd> en <kbd>↓</kbd> kan je scrollen in de `less`-omgeving. Met <kbd>q</kbd> kan je de `less`-omgeving afsluiten.

  
```shell
objdump -d hello-world | less
```

### ELF in xv6

We bekijken nu de ELF-files van xv6. ELF-files van xv6 zijn niet rechtstreeks uit te voeren binnen je Linux-omgeving. De ELF-files zijn namelijk gegenereerd voor de RISC-V ISA en bevatten dus enkel RISC-V instructies. 

De programma's `objdump` en `readelf` zijn in Linux gecompileerd voor de specifieke ISA van je processor. Deze programma's kunnen dus niet rechtstreeks gebruikt worden om de ELF-files van xv6 te bekijken.

In de eerste oefenzitting hebben we een RISC-V compiler geïnstalleerd. In dezelfde package zaten gelukkig ook RISC-V varianten van `objdump` en `readelf`.

* Analyseer de `_helloworld` ELF-file die we in vorige oefenzitting gemaakt hebben met behulp van `riscv64-linux-gnu-readelf`

```shell
riscv64-linux-gnu-readelf -a user/_helloworld | less
```

Merk op dat het aantal secties in de RISC-V ELF verschilt van het aantal secties in de x86-ELF. De compiler en linker beslissen welke secties worden toegevoegd aan een ELF-bestand.

* Disassemble de RISC-V machinecode in `_helloworld` met `riscv64-linux-gnu-objdump` 

```shell
riscv64-linux-gnu-objdump -d user/_helloworld | less
```


### Application Binary Interface (ABI)

> **TODO** Kort ABI's introduceren?

### De `exec` system call

Nadat een proces met behulp van `fork` gekopieerd is, willen we in vele gevallen dat dit proces een andere taak kan uitvoeren.
Hiervoor gebruiken we de system call `exec`.

`exec` neemt als invoer een ELF-bestand en initialiseert de ELF-secties in het werkgeheugen. Exec is dus de loader van UNIX-omgevingen.

Exec is ook verantwoordelijk voor een correcte initialisatie van de registers. Nadat het geheugen en de registers correct geïnitialiseerd worden, wordt het programma gestart door te springen naar het *entry point*. Het entry point wordt gespecifieerd in de header-sectie van een ELF-bestand.


* Zoek het entry point dat gebruikt wordt in xv6-programma's door gebruik te maken van `readelf`

De linker is verantwoordelijk om het entry point van een programma correct te bewaren in een ELF-bestand. De GNU linker [ld](https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_mono/ld.html#SEC24) die standaard door `gcc` gebruikt wordt, maakt gebruik van de volgende prioriteitenlijst om te bepalen wat er in het entry point bewaard wordt:

    1. the `-e' entry command-line option;
    2. the ENTRY(symbol) command in a linker control script;
    3. the value of the symbol start, if present;
    4. the address of the first byte of the .text section, if present;
    5. The address 0. 

In `xv6` wordt in de `Makefile` de `-e` flag opgegeven met als waarde `main`. De uitvoering van een xv6 executable zal dus starten bij het symbool `main`.


### C runtime (crt0)

In vorige oefenzitting hebben we gemerkt dat wanneer we returnen uit main, we een exception krijgen. We kunnen nu begrijpen waarom dit gebeurt.

Een proces wordt gestart door te springen naar het entry point van dat proces. `xv6` gebruikt als entry point `main`. Dat wil zeggen dat `main` niet als functie wordt opgeroepen en je dus ook geen `return` kan uitvoeren uit main.

In vele Linux-distributies wordt gebruik gemaakt van [crt0](https://en.wikipedia.org/wiki/Crt0) om C-programma's te starten. 
Het entry point van een C executable wordt geplaatst in `crt0`, met name bij het daarin gedefinieerde `_start` symbool.

`crt0` is dus het absolute beginpunt bij het uitvoering van een uit C gecompileerde executable. `crt0` initialiseert de C-runtime en roept vervolgens `main` op. Na uitvoering van `main` wordt de return-value van `main` doorgegeven aan de `exit` system call. Zo wordt het proces afgesloten.

Om het mogelijk te maken om te returnen uit `main` zonder exceptions, voegen we nu onze eigen `crt0` toe aan xv6.

* Maak een bestand `user/ctr0.c` aan
* Voeg `$U/crt0.o` toe aan `ULIB` in de Makefile van `xv6`. We zorgen er hiermee dus voor dat `crt0` gelinked wordt aan elk user-space programma.
* Implementeer `void _start(int argc, char* argv[])` in `user/crt0.c`

```c
#include "kernel/types.h"
#include "user/user.h"

extern int main(int argc, char* argv[]);

void _start(int argc, char* argv[])
{
    //TODO: Implement!
}
```


* Pas het entry point (`-e` flag) aan dat aan [`ld` wordt gegeven][ld rule] in de `Makefile` van xv6
* Test nu je implementatie van `crt0` door `_helloworld` te bewerken. Vervang je oproep naar `exit` door een `return` uit `main`. Indien `crt0` correct is geïmplementeerd, krijg je geen user exception na uitvoering van het programma.

# System calls

Ondertussen weten we hoe een proces opgestart kan worden, door middel van de system calls `fork` en `exec`. 
We weten dat we de system call `write` kunnen gebruiken om te schrijven naar de console, een buffer (bvb een pipe) of een bestand.

System calls geven processen de mogelijkheid diensten te vragen aan het besturingssysteem. 
Bij het uitvoeren van een system call geeft een proces tijdelijk de controle over de processor door aan het OS.

## System calls vs function calls

Een system call lijkt op het eerste zicht erg op een function call.
In de vorige sessie hebben de userspace functie `puts(char* str)` geschreven die een string schrijft naar de `stdout`.

Een proces dat een string wil schrijven naar de `stdout` kan de *function call* `puts` gebruiken, als volgt:
```c
char* str = "Hello, world!\n";
puts(str);
```

Een proces dat een string wil schrijven naar de `stdout` kan ook de *system call* `write` gebruiken, als volgt:
```c
char* str = "Hello, world!\n";
write(1, str, sizeof(str));
```

In C kan je het verschil niet zien tussen een system call en een function call. 
Indien we dit echter op assembly niveau bekijken is er wel een duidelijk verschil.
Tijd dus om in assembly te duiken.

> :exclamation: `puts` en andere functies zoals `printf` maken intern ook gebruik van de system call `write`. Het is niet mogelijk te schrijven zonder een system call. Dat verandert niets aan het feit dat oproepen naar `puts` en `printf` gewone function calls zijn.

### RISC-V assembly

Neem onderstaand simpel C-programma:

```c
int value = 5;

int doubleIt(int i){
    return i*2;
}

int main(){
    doubleIt(i);
}
```

Vertaald naar RISC-V assembly zou dit er als volgt kunnen uitzien.

* Lees de onderstaande assembly-code en zorg ervoor dat je elke regel begrijpt.
> :information_source:  Denk terug aan functie-oproepen in DRAMA in het vak SOCS. Conceptueel gezien zijn deze identiek hetzelfde. SOCS gebruikt echter nederlandstalige termen voor vele concepten, dit kan tot verwarring leiden. Hier een snelle cheat sheet:
> | DRAMA | RISC V | Verklaring |
> | --- | --- | --- |
> | HIA.w reg, val | li reg, val | Laad waarde `val` in register `reg`
> | HIA reg, addr  | ld reg, addr | Laad waarde op adres `addr` in register `ref`
> | OPT.w reg, val | addi reg, val | Tel waarde `val` op bij register `reg`
> | SPR label | j label | Spring naar symbool `label`
> | BST value | addi sp, sp, -8 | Bewaar `value` op de stack
> |           | sd value, 0(sp) |
> | HST reg   | ld value, 0(sp) | Haal waarde van de stack en bewaar in `reg`
> |           | addi sp, sp, 8  |
> 

```asm
.data                   #Start de data-sectie
    value: .word 0x5    #Init global value met waarde 5

.text                   #Start de code-sectie

.globl main             #Exporteer het symbool main
                        #Dit zorgt ervoor dat oa. crt0
                        #een functie-oproep naar main
                        #kunnen uitvoeren.


doubleIt:               #Definieer het symbool doubleIt
                        #Calling convention:
                        #   a0: input argument 1
                        #   a0: return value
    add a0, a0, a0      #Verdubbel i, geef terug via a0
    ret                 #Spring naar het adres in register ra
                        #ra is het return-address register

main:                   #Definieer het symbool main

    addi sp, sp, -8     #Reserveer 8 bytes (64 bit) 
                        #op de call stack door de stack
                        #pointer te verlagen met 8
                        #(Stack groeit van hoog naar laag)

    sd ra, (sp)         #Bewaar de huidige waarde van
                        #het return-adres register ra
                        #op de call stack

    la t0, value        #Laad in t0 het adres van 
                        #de global value

    ld a0, (t0)         #Laad de waarde op het adres in t0
                        #in a0

    jal doubleIt        #Spring naar symbool doubleIt en
                        #sla in het return-address register
                        #ra het adres op waarnaar ret moet
                        #springen.

    ld ra, (sp)         #Haal het oude return-adres terug
                        #van de stack en bewaar het in ra

    ret                 #Spring naar het adres in register
                        #ra (een adres in crt0?)
```



## Scratchpad

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
        - **REMOVE** [solution](https://github.com/besturingssystemen/xv6-solutions/commit/f5671422e83c36303acd41abd29faa49eb2eb5c3)
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
        - **REMOVE** [solution](https://github.com/besturingssystemen/xv6-solutions/commit/b3b23725a8ecd62b54e67dfaf1acab4fbc5ead5f)
- xv6 system call dispatch code
    - [`syscall`][syscall]
      Vanaf hier zou de syscall code begrijpbaar moeten zijn.
      We houden de eerdere code (traphandler enzo) voor de scheduling oefenzitting.
      We moeten enkel uitleggen wat trapframe precies is.
    - [`getpid`][sys_getpid] als voorbeeld (eenvoudigste syscall)
- system call toevoegen: `getnumsyscalls`
    - voeg `uint64 numsyscalls` toe aan [`struct proc`][struct proc] (private section -> explain)
    - initialiseer `numsyscalls` in [`allocproc`][allocproc]
    - verhoog `numsyscalls` in [`syscall`][syscall]
    - voeg `SYS_getnumsyscalls` toe aan [kernel/syscall.h][syscall.h]
    - implementeer `sys_getnumsyscalls` in [kernel/sysproc.c][sysproc.c]
    - dispatch `sys_getnumsyscalls` in [`syscall`][syscall]
- system call gebruiken in user space
    - voeg `int getnumsyscalls()` toe aan [user/user.h][user.h]
    - voeg `entry("getnumsyscalls")` toe aan [user/usys.pl][usys.pl] (leg uit dat dit script gewoon een assembly bestand genereerd)
    - gebruik die nieuwe system call in een user space programma
- **REMOVE** [solution](https://github.com/besturingssystemen/xv6-solutions/commit/fc2ac55f12d039b83a6068f9e3b9f08fd442b44c)
- call `getnumsyscalls` in `_start` na main
    - verklaar waarom `hello_asm_write` 4 syscalls doet voor exit (`sbrk` door `malloc` in `sh` (waarschijnlijk niet altijd), `exec` in `sh`, `write` in `hello`, `getnumsyscalls` zelf in `hello`)

# Permanente evaluatie

- Voeg een nieuwe system call `void traceme(int enable)` toe die ervoor zorgt dat (als `enable` truthy is) elke system call van het proces geprint wordt naar de console (gebruik de kernel [`printf`][kernel printf]) samen met de pid van het proces. (`[{pid}]: syscall {num}`)
  Gebruik [`argint`][argint] om een sycall argument op te vragen.
    - **NOTE** in een eerdere versie van de opgave was er een `fd` argument. Het blijkt echter moeilijk om in de kernel geformatteerde strings naar een file te schrijven (geen `sprintf` en `filewrite` verwacht een user space adres)
    - **REMOVE** [solution](https://github.com/besturingssystemen/xv6-solutions/commit/a6ec06062f1fd1925687347143ec431359c1a7f8)
- Maak een user space programma `trace` dat een executable als argument krijgt en deze executable oproept met de `traceme` functionaliteit aangezet. (`traceme`, `exec`)
    - **REMOVE** [solution](https://github.com/besturingssystemen/xv6-solutions/commit/2fa37b2beb1b60d0aeb7cd584828076146041b4f)


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
[kernel printf]: https://github.com/besturingssystemen/xv6-riscv/blob/3c44dade20d87b259a3713c6d84ecccfd3056bef/kernel/printf.c#L64
[argint]: https://github.com/besturingssystemen/xv6-riscv/blob/3c44dade20d87b259a3713c6d84ecccfd3056bef/kernel/syscall.c#L58
[fork]: https://github.com/besturingssystemen/xv6-riscv/blob/280d2aa694114e7a6e7eb2a9c4f62e3c314983c6/kernel/proc.c#L266
