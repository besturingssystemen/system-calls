In deze oefenzitting leren jullie over de werking van system calls in xv6.

- [Voorbereiding](#voorbereiding)
- [GitHub classroom](#github-classroom)
  - [Persoonlijke repository](#persoonlijke-repository)
- [Introductie](#introductie)
- [Levenscyclus proces](#levenscyclus-proces)
  - [Aanmaak processen](#aanmaak-processen)
    - [De `fork` system call](#de-fork-system-call)
    - [Process state](#process-state)
    - [Trapframe](#trapframe)
    - [Executable Files](#executable-files)
    - [C runtime (crt0)](#c-runtime-crt0)
- [System calls](#system-calls)
  - [System calls vs function calls](#system-calls-vs-function-calls)
  - [RISC-V assembly](#risc-v-assembly)
  - [xv6 system calls in RISC-V](#xv6-system-calls-in-risc-v)
    - [Voorbereiding system call](#voorbereiding-system-call)
    - [Implementatie system call](#implementatie-system-call)
    - [System call exposen naar user-space](#system-call-exposen-naar-user-space)
    - [System call oproepen](#system-call-oproepen)
- [Permanente evaluatie](#permanente-evaluatie)
- [Bonusoefening](#bonusoefening)

# Voorbereiding

Ter voorbereiding van deze oefenzitting word je verwacht:

* De oefenzitting [os interfaces](https://github.com/besturingssystemen/os-interfaces) te hebben voltooid.
* Hoofdstuk 2 van het [xv6 boek](https://github.com/besturingssystemen/xv6-riscv) te hebben gelezen.

# GitHub classroom

## Persoonlijke repository

De submissie van de permanente evaluatie zal gebeuren via GitHub classroom. 

> :exclamation: Voor elke sessie wordt een nieuwe repository ter beschikking gesteld. De oude repository wordt afgesloten op de deadline van de permanente evaluatie!

* Klik op [deze link](https://classroom.github.com/a/DWV0ppLK) om een persoonlijke repository aan te maken.

Wanneer je een e-mail krijg van GitHub dat je repository klaar is, moet je deze clonen naar je eigen machine. Dit kan enkele minuten duren.

Indien `make qemu` ervoor zorgt dat xv6 opstart, is je repository correct gecloned.

# Introductie

In deze sessie zullen we system calls in detail bestuderen.
In het eerste deel kijken we naar de werking van enkele bestaande system calls gerelateerd aan de levenscyclus van een proces.
Vervolgens leren we hoe we zelf een system call kunnen toevoegen aan xv6.
Ten slotte vragen we jullie zelfstandig, als permanente evaluatie, een system call toe te voegen aan xv6.

# Levenscyclus proces

Een proces is een abstractie op niveau van het besturingssysteem. 
Besturingssystemen zijn verantwoordelijk voor de aanmaak, het beheer en de correcte afsluiting van processen.

De proces-abstractie is enorm krachtig.
Ze laat ons toe om complexe systemen op te bouwen als een verzameling programma's die elk een eigen specifieke rollen vervullen.
Deze programma's kunnen dankzij de procesabstractie in parallel worden uitgevoerd, geïsoleerd van elkaar met elk een eigen geheugenruimte.

Om te vermijden dat processen in user space de correct werking van het besturingssysteem in gedrang kunnen brengen, zal de kernel (de core van het besturingssysteem) zichzelf ook isoleren van deze processen.
Dit wilt niet alleen zeggen dat processen het geheugen van de kernel niet kunnen lezen of schrijven maar ook dat ze de code van de kernel niet uit kunnen voeren.
System calls zijn de interface van de kernel naar processen en laten het toe om de code in de kernel op een gecontroleerde manier uit te voeren.

## Aanmaak processen

### De `fork` system call

In [UNIX](https://en.wikipedia.org/wiki/Unix) (Linux, xv6, ...) besturingssystemen worden processen aangemaakt door middel van de `fork` system call.
Deze system call maakt een kopie van het huidige proces.

### Process state

In detail begrijpen hoe een proces gekopieerd wordt, is op dit punt in de oefenzittingen nog te vroeg. We kunnen wel al tonen welke state een besturingssysteem bewaart per proces.

* Bekijk [struct proc][struct proc] gedefinieerd in `proc.h` in xv6.
* Lees de comments bij elk veld van de `struct`. 
  
Het nut van de velden `parent`, `pid`, `sz`, `ofile`, `cwd` en `name` zou duidelijk moeten zijn. Vraag verduidelijking aan een assistent indien dit niet het geval is. 

> :bulb: Een file descriptor (`int fd`) indexeert de `ofile` array. Wanneer we dus met de `write` system call schrijven naar een bestand geven we als eerste argument aan `write` een index mee in de tabel met open bestanden van het proces.

Om het veld `pagetable` te begrijpen hebben we kennis nodig van virtual memory, een concept dat we in de volgende oefenzitting zullen bekijken.

De velden `state` en `killed` bevatten informatie voor de scheduler en zullen dus in detail worden bekeken in een toekomstige oefenzitting over scheduling.

De velden `lock`, `chan` en `xstate` hebben te maken met synchronizatie en worden in een latere oefenzitting over synchronizatie in detail bekeken.



### Trapframe

Begrip van het veld `trapframe` is belangrijk voor deze oefenzitting. Wanneer een proces de controle doorgeeft aan het besturingssysteem bij het uitvoeren van een *system call* of wanneer een proces onderbroken wordt door bijvoorbeeld een interrupt, gebeurt dit via een *trap*.

Een *trap* in user-mode zorgt ervoor dat de processor schakelt naar supervisor-mode en vervolgens de *trap handler* begint uit te voeren. Deze handler is een stuk machinecode op een vaste locatie in het geheugen.

Het uitvoerende proces, dat de trap heeft veroorzaakt, wil na afloop van de trap verder kunnen uitvoeren. Het kan echter zijn dat machinecode in het besturingssysteem bepaalde registers nodig heeft die reeds in gebruik zijn door het uitvoerende proces. Om ervoor te zorgen dat deze registerwaarden niet verloren gaan, bewaart de trap handler deze in de *trap frame*. Bij terugkeer uit de trap kunnen de registerwaarden hersteld worden.

* Bekijk de velden van [`struct trapframe`][trapframe] in kernel/proc.h.

Het veld `kernel_satp` is gerelateerd aan het veld `pagetable` in `struct proc` en zal pas in detail bekeken worden in de oefenzitting over Virtual Memory.

In xv6 heeft de kernel een eigen call stack, gesplitst van de call stack van het proces. Het veld `kstack` in `struct proc` bevat het adres van deze stack, het veld `kernel_sp` bevat een pointer naar de top van deze stack.
Merk op dat xv6 dus een kernel stack heeft _per proces_.
Waarom dit nodig is, zal duidelijk worden in latere oefenzittingen.

Het veld `kernel_hartid` bewaart de identifier van de hardware thread (CPU core) waarop het proces actief is. Dit wordt later besproken in de sessie over synchronizatie.

Op dit moment zou je een idee moeten hebben van de state die per proces bewaard wordt, en de state die bewaard en hersteld moet worden bij het uitvoeren van een trap.

<!--
**TODO** `context`

Het veld `context` zal duidelijk worden door te scrollen naar het begin van het `proc.h` bestand.
Je kan daar de definitie van `struct context` terugvinden.

Het veld `context` bevat een struct waarin alle nodige registerwaarden van het proces bewaard kunnen worden bij het uitvoeren van een context switch. 
-->

### Executable Files

De `fork` system call zal als een van zijn taken de process state kopiëren voor het nieuwe proces. Nadat een proces gekopieerd is, is het in vele gevallen de bedoeling dat het nieuwe proces een eigen taak toegewezen krijgt.
De meest gangbare manier om het proces een nieuwe taak toe te wijzen, maakt gebruik van de system call `exec`.

`exec` neemt als invoer een uitvoerbaar bestand (programma) en vervangt de code en data in het huidige proces door deze in het uitvoerbare bestand. Vervolgens wordt gesprongen naar het *entry point* van het programma. Op deze manier krijgt het geforkte proces een nieuwe taak toegewezen.

Om samen te vatten wordt een proces in UNIX aangemaakt door
een combinatie van de system call `fork`, dat de processtructuur kopieert, en `exec`, dat een nieuw programma
in het programma laadt.

### C runtime (crt0)

In vorige oefenzitting hebben we gemerkt dat wanneer we returnen uit main, we een exception krijgen. We kunnen nu begrijpen waarom dit gebeurt.

Een proces wordt gestart met behulp van `exec` door een programma in te laden en vervolgens te springen naar het entry point van dat proces. `xv6` gebruikt als entry point `main`. Dat wil zeggen dat `main` niet als functie wordt opgeroepen en je dus ook geen `return` kan uitvoeren uit main (want het zal geen geldig return adres hebben).

In vele Linux-distributies wordt gebruik gemaakt van [crt0](https://en.wikipedia.org/wiki/Crt0) om C-programma's te starten. 
Het entry point van een C executable wordt geplaatst in `crt0`, vaak in een functie genaamd `_start`.

`_start` in `crt0` is dan het absolute beginpunt bij het uitvoering van een uit C gecompileerde executable. `_start` initialiseert de C-runtime en roept vervolgens `main` op. Na uitvoering van `main` wordt de return-value van `main` doorgegeven aan de `exit` system call. Zo wordt het proces afgesloten.

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
We weten ook dat we de system call `write` kunnen gebruiken om te schrijven naar de console, een buffer, een bestand, of naar een pipe geopend met de system call `pipe`.

System calls geven processen de mogelijkheid diensten te vragen aan het besturingssysteem. 
Bij het uitvoeren van een system call geeft een proces tijdelijk de controle over de processor door aan het OS.

## System calls vs function calls

Een system call lijkt op het eerste zicht erg op een function call.
In de vorige sessie hebben de userspace functie `puts(char* str)` geschreven die een string schrijft naar de `stdout`.

Een proces dat een string wil schrijven naar de `stdout` kan de *function call* `puts` gebruiken, als volgt:
```c
const char* str = "Hello, world!";
puts(str);
```

Een proces dat een string wil schrijven naar de `stdout` kan ook de *system call* `write` gebruiken, als volgt:
```c
const char* str = "Hello, world!\n";
write(1, str, strlen(str));
```
In dit geval biedt C echter een wrapper-functie aan, zodat het mogelijk is om een system call op te roepen als een functie.
De echte syscall gebeurt dus in de implementatie van de `write` wrapper-functie.
Deze wrappers moeten echter op assembly-niveau geïmplementeerd worden.
Het is dus tijd om terug in assembly te duiken.

> :exclamation: `puts` en andere functies zoals `printf` maken intern ook gebruik van de system call `write`. Het is niet mogelijk te schrijven zonder een system call. Dat verandert niets aan het feit dat oproepen naar `puts` en `printf` gewone function calls zijn.

## RISC-V assembly

Neem onderstaand simpel C-programma:

```c
int value = 5;

int sum(int i, int j){
    return i + j;
}

int main(){
    sum(value, 10);
}
```

Vertaald naar RISC-V assembly zou dit er als volgt kunnen uitzien:

* Lees de onderstaande assembly-code en zorg ervoor dat je elke regel begrijpt.
  
> :information_source:  Denk terug aan functie-oproepen in DRAMA in het vak SOCS. Conceptueel gezien zijn deze identiek hetzelfde. SOCS gebruikt echter nederlandstalige termen voor vele concepten, dit kan tot verwarring leiden. Hier een snelle cheat sheet:
> | DRAMA | RISC V | Verklaring |
> | --- | --- | --- |
> | HIA.w reg, val | li reg, val | Laad waarde `val` in register `reg`
> | HIA reg1, (reg2)  | ld reg1, (reg2) | Laad waarde op het adres in `reg2` in register `reg1`
> | HIA.a reg, symbol | la reg, symbol | Laad het adres van `symbol` in register `reg`
> | BIG reg1, (reg2)  | sd reg1, (reg2) | Bewaar waarde in `reg1` op het adres in `reg2`
> | OPT.w reg, val | addi reg, reg, val | Tel waarde `val` op bij het register `reg`
> | OPT dst, src   | add dst, dst, src | dst = dst + src (`dst` en `src` zijn registers)
> | SPR label | j label | Spring naar symbool `label`
> | BST value | addi sp, sp, -8 | Bewaar `value` op de stack (stapel)
> |           | sd value, 0(sp) |
> | HST reg   | ld value, 0(sp) | Haal waarde van de stack en bewaar in `reg`
> |           | addi sp, sp, 8  |
> | SBR symbol| jal ra, symbol      | Schrijf de waarde van de programmateller naar het return adres register (TKA in DRAMA, ra in Risc V) en spring naar `symbol`
> | KTG       | jr ra             | Spring naar het adres in het return address register

```asm
.data                   #Start de data-sectie
    value: .word 0x5    #Init global value met waarde 5

.text                   #Start de code-sectie

.globl main             #Exporteer het symbool main
                        #Dit zorgt ervoor dat oa. crt0
                        #een functie-oproep naar main
                        #kan uitvoeren.


sum:                    #Definieer het symbool sum
                        #Calling convention:
                        #   a0: input argument 1 (i)
                        #   a1: input argument 2 (j)
                        #   a0: return value
    add a0, a0, a1      #Tel i en j op, geef terug via a0
    jr ra               #Spring naar het adres in register ra
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

    li a1, 10           #Laad de waarde 10 in a1

    jal ra, sum         #Spring naar symbool sum en
                        #sla in het return-address register
                        #ra het adres op waarnaar ret moet
                        #springen.

    ld ra, (sp)         #Haal het oude return-adres terug
                        #van de stack en bewaar het in ra

    addi sp, sp, 8      #Geef de gereserveerde 8 bytes op
                        #de stack terug vrij

    jr ra               #Spring naar het adres in register
                        #ra (een adres in crt0?)
```

In RISC-V worden functie-parameters in de eerste plaats doorgegeven via de registers `a0` - `a7`. Wanneer er niet genoeg registers zijn voor het aantal parameters, worden extra parameters via de stack meegegeven. Een volledige beschrijving van de compiler-conventies voor een RISC-V functie-oproep kan je vinden in de [RISC-V calling conventions](https://riscv.org/wp-content/uploads/2015/01/riscv-calling.pdf).

* Voeg nu een bestand `user/hello_asm_puts.S` toe. Je kan vertrekken van de volgende assembly-code:

 ```asm
.text
.globl main
main:
    #TODO functie-oproep voorbereiden (hint: ra!)
    jal ra, puts
    #TODO return adres ra herstellen
    ret

.section .rodata
hello_str: .string "Hello, world!"
```
* Voeg `$U/_hello_asm_puts\` aan [`UPROGS`][UPROGS] in de Makefile
* Implementeer `main`. Denk hierbij aan het bewaren van het return address in `ra` voor je een functie oproept.


## xv6 system calls in RISC-V

Nu functie-oproepen in assembly weer vers in het geheugen zit, is het tijd om te kijken naar de werking van een system call.

Alle system calls in RISC-V worden uitgevoerd met behulp van de `ecall` instructie. Deze instructie, wanneer uitgevoerd vanuit user mode, zal ervoor zorgen dat de processor overgaat naar *supervisor mode* en vervolgens (via de trampoline, hierover meer in de sessie over Virtual Memory) springt naar de eerder besproken trap handler. Herinner je dat in die trap handler de registers van het user-space programma bewaard worden.

De trap handler zal vervolgens de oorzaak van de trap bepalen. Indien de trap veroorzaakt was door een `ecall` weten we dat de gebruiker een system call probeerde op te roepen.
Er wordt dus gesprongen naar de system call handler, met name de functie `syscall(void)`.

* Bekijk de functie [`syscall`][syscall] in de code van xv6. Welk register wordt hier gebruikt om te bepalen welke system call opgeroepen moet worden?
* Voeg nu een bestand `user/hello_asm_write.S` toe (let op de hoofdletter `S` in de extensie). Maak gebruik van de `ecall` instructie om de system call `write` uit te voeren. Je zal het register uit bovenstaande vraag moeten gebruiken om de correcte system call op te vragen. Je kan starten vanuit onderstaande code:

```asm
#include "kernel/syscall.h"

.text
.global main
main:
    #TODO implement

    #HINT: you can use the symbols defined in the header
    #file kernel/syscall.h
    #e.g. li t0, SYS_fork loads the value 1 into t0

.section .rodata
hello_str:
    .string "Hello, world!\n"
```

> :bulb: Naast de selectie van de system call zal het ook nodig zijn om parameters door te geven aan `write`. Dit gebeurt volgens dezelfde conventies als bij een gewone functie-oproep met `jal`. De signature van `write` is `int write(int fd, const void *buf, int nbytes)`. Gebruik dus de correcte registers om deze parameters door te geven.  

We weten nu hoe we een system call opgeroepen wordt vanuit user-space, hoe de trap handler en system call handler ervoor zorgen dat de juiste system call wordt uitgevoerd en we begrijpen hoe een C-wrapper functie dit voor een programmeur erg eenvoudig kan maken.
Tijd om eens te kijken naar de implementatie van zo'n system call.

* Bekijk de implementatie van de eenvoudige system call [`getpid`][sys_getpid]. Zo eenvoudig kan het zijn.  
* Kijk eens naar de andere system calls die in hetzelfde bestand zijn geïmplementeerd. Het is nog niet nodig om deze allemaal in detail te begrijpen.
  
In de vorige sessie hebben we geleerd hoe je een user-space programma kan toevoegen aan xv6. 
Ondertussen zijn we klaar om onze eerste aanpassing te maken aan de kernel zelf. 
We zullen onze eigen system call toevoegen.
Na deze sessie kan je jezelf dus officieel een OS programmeur noemen.

De system call die we gaan toevoegen is `getnumsyscalls`.
Wanneer een user-space programma deze system call uitvoert, krijgt deze als resultaat het totaal aantal uitgevoerde system calls in het huidige proces. 

### Voorbereiding system call

Om dit mogelijk te maken moeten we in de process state in `struct proc` een teller bijhouden.

* Voeg `uint64 numsyscalls` toe aan de [`struct proc`][struct proc] in `kernel/proc.h`. Doe dit in de private sectie. (In de sessie over synchronizatie zal duidelijk worden waarom.)

Wanneer we een veld toevoegen aan de struct moeten we er uiteraard ook voor zorgen dat dit veld een initiële waarde toegewezen krijgt bij de aanmaak van een proces.

* Initialiseer het veld `numsyscalls` in de functie [`allocproc`][allocproc] in [`kernel/proc.c`][proc].
  De `allocproc` functie wordt door `fork` gebruikt om een nieuwe `struct proc` aan te maken.


Om ervoor te zorgen dat de teller het aantal system calls correct telt kunnen we deze waarde verhogen telkens wanneer de system call handler wordt opgeroepen.

* Verhoog `numsyscalls` in [`syscall`][syscall] in [`kernel/syscall.c`][syscall]

### Implementatie system call

Nu onze teller correct werkt, resteert ons enkel de effectieve implementatie van de system call.

* Voeg `SYS_getnumsyscalls` toe aan [`kernel/syscall.h`][syscall.h].
* Implementeer `sys_getnumsyscalls` in [`kernel/sysproc.c`][sysproc.c].
* Zorg ervoor dat de system call handler [`syscall`][syscall] onze nieuwe system call correct doorstuurt.

> :bulb: Om `sys_getnumsyscalls` te kunnen oproepen vanuit `syscall` zal je de C-compiler moeten vertellen dat deze functie bestaat, en deze dus moeten declareren. Volg het voorbeeld van de declaraties van de andere system calls in het bestand.

### System call exposen naar user-space

Onze system call werkt nu. We kunnen hem alleen nog niet eenvoudig oproepen vanuit user-space. 
Herinner je dat system calls opgeroepen worden via een C-wrapper.
Laten we allereerst onze C-wrapper declareren:

* Voeg `int getnumsyscalls()` toe aan [`user/user.h`][user.h]

De implementatie van de wrapper-functie gebeurt in assembly. In principe hebben jullie ondertussen genoeg kennis om deze wrapper zelf te implementeren.
xv6 biedt echter een script aan waarmee dit volledig automatisch kan verlopen.
Het [`user/usys.pl`][usys.pl] script genereert automatisch een assembly-bestand (`user/usys.S`) met daarin de implementatie van de wrappers.
Bekijk dit gegenereerde bestand en zorg dat je de wrappers begrijpt.

- voeg `entry("getnumsyscalls")` toe aan [`user/usys.pl`][usys.pl]

### System call oproepen

De system call is klaar.
Tijd om  deze uit te testen.
We zullen ervoor zorgen dat onze C-runtime na afloop van ons programma print hoeveel system calls uitgevoerd werden.

* Bewerk `crt0.c` zodat `getnumsyscalls` opgeroepen wordt na de return uit `main` en print het resultaat uit.

Indien dit correct werkt zullen de user-space programma's die returnen uit main het aantal uitgevoerde system calls printen. 

* Voer nu het programma `hello_asm_write` uit. Welk resultaat krijg je? Is dit het resultaat dat je verwacht had? Kan je achterhalen welke system calls werden uitgevoerd in het proces die je niet zelf expliciet hebt opgeroepen?
    
# Permanente evaluatie

Als permanente evaluatie van deze oefenzitting is het de bedoeling om zelfstandig een system call toe te voegen.

* Voeg een nieuwe system call `void traceme(int enable)` toe die ervoor zorgt dat (als `enable` truthy is) elke system call van het proces geprint wordt naar de console, samen met de `pid` van het proces.
  * Gebruik de kernel-versie van [`printf`][kernel printf]
  * Volg het printformaat `[{pid}]: syscall {num}`
  * Gebruik [`argint`][argint] om een syscall argument op te vragen.
* Maak een user-space programma `trace` dat een executable als argument krijgt en deze executable oproept met de `traceme` functionaliteit aangezet.
  * Hint: gebruik de system calls `traceme` en `exec`

Zoals aangekondigd op Toledo is de deadline van deze permanente evaluatie een week nadat je deze oefenzitting volgt, op donderdag om 10h30.

# Bonusoefening

- Pass `traceme` aan om nu een `fd` als input te krijgen
- Wanneer `traceme` opgeroepen wordt, valideer je eerst de `fd` (geldige open file, is een pipe, is writable)
  * Zo ja, sla op in `struct proc::tracefd`.
  * Zo nee, zet dit veld op -1
- Wanneer er een syscall gebeurt en `tracefd` niet gelijk is aan -1, schrijf een `struct tracemsg` naar `tracefd` via [`pipewrite`][pipewrite] (zet `user_src` op 0 om aan te geven dat de input in kernel space zit)
- Schrijf een user space programma `trace` dat gebruik maakt van deze nieuwe syscall

> :information_source: Deze oefening is niet verplicht. Werk eerst je permanente evaluatie af voor je hier aan zou beginnen.

[struct proc]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/proc.h#L94
[syscall]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/syscall.c#L133
[syscall.h]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/syscall.h
[allocproc]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/proc.c#L100
[proc]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/proc.c
[sys_getpid]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/sysproc.c#L21
[sysproc.c]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/sysproc.c
[user.h]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/user/user.h
[usys.pl]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/user/usys,pl
[ULIB]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/Makefile#L94
[ld rule]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/Makefile#L97
[UPROGS]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/Makefile#L122
[kernel printf]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/printf.c#L64
[argint]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/syscall.c#L58
[fork]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/proc.c#L266
[pipewrite]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/pipe.c#L77
[trapframe]: https://github.com/besturingssystemen/xv6-riscv/blob/2b5934300a404514ee8bb2f91731cd7ec17ea61c/kernel/proc.h#L52
