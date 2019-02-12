Kernel booting process. Part 2.
================================================================================

First steps in the kernel setup
--------------------------------------------------------------------------------

이전 [part](linux-bootstrap-1.md)에서 linux kernel의 내부에 뛰어들어 kernel setup coode의 초기 부분을 보았습니다. [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)에서 `main` function (C로 쓰여진 첫 function)이 호출되는 것까지 보았습니다.

이 part에서는 kernel setup code에 대한 연구를 계속하고 하기 항목을 다룰 것입니다.
* `protected mode` 가 무엇인지,
* `protected mode`로 천이,
* heap과 console의 초기화,
* memory detection, CPU validation과 keyboard 초기화
* 그리고 기타 사항.

자, 진행합시다.

Protected mode
--------------------------------------------------------------------------------

native Intel64 [Long Mode](http://en.wikipedia.org/wiki/Long_mode)로 천이하기 전에, kernel은 CPU를 protected mode로 변경해야 합니다.

[protected mode](https://en.wikipedia.org/wiki/Protected_mode)는 무엇인가? protected mode는 1982년 x86 architecture에 처음 추가되었고 Intel 64와 long mode가 등장할 때까지 [80286](http://en.wikipedia.org/wiki/Intel_80286) processor 이후의 Intel processors의 main mode 였습니다.

[Real mode](http://wiki.osdev.org/Real_Mode)에서 변경된 주된 이유는 매우 제한적인 RAM으로의 접근 때문입니다. 이전 part에서 기억하시는 것처럼 Real mode에서는 2<sup>20</sup> bytes 혹은 1 Megabyte, 때로는 오직 640 Kilobytes의 RAM만 가용합니다.

Protected mode는 많은 변화를 가져왔습니다. 그러나 주된 것은 memory management 입니다. 20-bit address bus는 32-bit address bus로 교체되었습니다. real mode에서 1 Megabyte가 가용한데 비해 그것은 4 Gigabytes의 memory에 access를 허용합니다. 또한, 다음 sections에서 읽을 수 있는 [paging](http://en.wikipedia.org/wiki/Paging)에 대한 지원이 추가되었습니다.

protected mode에서의 memory management는 거의 독립적인 2개의 부분으로 나뉩니다:

* Segmentation
* Paging

여기서는 오직 segmentation에 대해서만 논의하겠습니다. paging은 다음 sections에서 논의하겠습니다.

이전 part에서 읽을 수 있듯이, real mode에서 addresses는 2개의 parts로 구성됩니다:

* segment의 base address
* segment base로부터의 offset

이 2개의 parts를 알고 있으면 다음과 같이 physical address를 얻을 수 있습니다:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

protected mode에서 memory segmentation은 완전히 뜯어고쳐졌습니다. 64 Kilobyte의 fixed-size segments는 존재하지 않습니다. 대신 각 segment의 size와 location은 _Segment Descriptor_라는 data structure에 의해 기술됩니다. segment descriptors는 `Global Descriptor Table` (GDT)라는 data structure에 저장됩니다.

GDT는 memory에 존재하는 structure입니다. memory에서 위치가 고정되어 있지 않기 때문에, 그 address는 special `GDTR` register에 저장됩니다. 나중에 linux kernel code에서 GDT가 어떻게 load되는지 살펴볼 것입니다. 다음과 같이 memory에 GDT를 load합니다:

```assembly
lgdt gdt
```

여기서 `lgdt` instruction은 global descriptor table의 base address와 limit(size)를 `GDTR` register로 load합니다. `GDTR`은 48-bit register로 2개의 부분으로 구성됩니다:

 * global descriptor table의 size(16-bit);
 * global descriptor table의 address(32-bit).

위에서 언급한 것처럼 GDT는 memory segments를 기술하는 `segment descriptors`를 포함합니다. 각 descriptor는 64-bits이며 descriptor의 일반적인 scheme은 다음과 같습니다:

```
 63         56         51   48    45           39        32 
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 |
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------

 31                         16 15                         0 
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           |
|                             |                            |
------------------------------------------------------------
```

걱적 마십시요. real mode 이후 이것은 조금 무서워 보일 수 있습니다. 하지만, 간단합니다. 예를 들면 LIMIT 15:0은 Limit의 bits 0-15이 descriptor의 시작 부분에 위차한다는 것을 의미합니다. 나머지 부분인 LIMIT 19:16은 descriptor의 bits 48-51에 위치해 있습니다. 그러니까 Limit의 크기는 0-19 즉 20-bits입니다. 좀 더 자세히 살펴봅시다:

1. Limit[20-bits]는 bits 0-15와 48-51로 나뉩니다. 이것은 `length_of_segment - 1`를 정의합니다. 이것은 `G`(Granularity) bit에 의존합니다.

  * 만약 `G` (bit 55)이 0이고 segment limit이 0이면, segment의 size는 1 Byte 입니다.
  * 만약 `G`가 1이고 segment limit가 0이면, segment의 size는 4096 Bytes 입니다.
  * 만약 `G`가 0이고 segment limit가 0xfffff이면, segment의 size는 1 Megabyte 입니다.
  * 만약 `G`가 1이고 segment limit가 0xfffff이면, segment의 size는 4 Gigabytes 입니다.

  그러니까, 이것이 의미하는 것은
  * 만약 G가 0이면, Limit는 1 Byte로 해석되고 segment의 최대 size는 1 Megabyte가 될 수 있다.
  * 만약 G가 1이면, Limit는 4096 Bytes = 4 KBytes = 1 Page로 해석되고 segment의 최대 size는 4 Gigabytes가 될 수 있다. 실제로, G가 1일 때, Limit의 값은 왼쪽으로 12 bits shift됩니다. 그래서, 20 bits + 12 bits = 32 bits 그리고 2<sup>32</sup> = 4 Gigabytes.

2. Base[32-bits]는 bits 16-31, 32-39와 56-63로 나뉩니다. 이것은 segment의 시작 location의 physical address를 정의합니다.

3. Type/Attribute[5-bits]는 bits 40-44에 의해 표현됩니다. 이것은 segment의 type과 어떻게 access될 수 있는지를 정의합니다.
  * bit 44의 `S` flag는 descriptor type을 표시합니다. 만약 `S`가 0이면 이 segment는 system segment이고, 만약 `S`가 1이면 이 segment는 code 혹은 data segment입니다 (Stack segments는 read/write가 가능한 data segments입니다).

segment가 code 혹은 data segment인지 구분하기 위해 Ex(bit 43) Attribute (위의 diagram에서 0으로 mark된 부분)를 check할 수 있습니다. 만약 0이면 segment는 data segment이고, 그렇지 않으면 code segment입니다.

segment는 다음 types중 하나일 수 있습니다:

```
--------------------------------------------------------------------------------------
|           Type Field        | Descriptor Type | Description                        |
|-----------------------------|-----------------|------------------------------------|
| Decimal                     |                 |                                    |
|             0    E    W   A |                 |                                    |
| 0           0    0    0   0 | Data            | Read-Only                          |
| 1           0    0    0   1 | Data            | Read-Only, accessed                |
| 2           0    0    1   0 | Data            | Read/Write                         |
| 3           0    0    1   1 | Data            | Read/Write, accessed               |
| 4           0    1    0   0 | Data            | Read-Only, expand-down             |
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed   |
| 6           0    1    1   0 | Data            | Read/Write, expand-down            |
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed  |
|                  C    R   A |                 |                                    |
| 8           1    0    0   0 | Code            | Execute-Only                       |
| 9           1    0    0   1 | Code            | Execute-Only, accessed             |
| 10          1    0    1   0 | Code            | Execute/Read                       |
| 11          1    0    1   1 | Code            | Execute/Read, accessed             |
| 12          1    1    0   0 | Code            | Execute-Only, conforming           |
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed |
| 13          1    1    1   0 | Code            | Execute/Read, conforming           |
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed |
--------------------------------------------------------------------------------------
```

위에서 보시는 바와 같이 첫번째 bit(bit 43)은 _data_ segment인 경우 `0`이고 _code_ segment인 경우 `1`입니다. 다음 3개의 bits(40, 41, 42)은 `EWA`(*E*xpansion *W*ritable *A*ccessible) 혹은 CRA(*C*onforming *R*eadable *A*ccessible)를 의미합니다.
  * 만약 E(bit 42)가 0이면, expand up, 그렇지 않으면, expand down입니다. [여기](http://www.sudleyplace.com/dpmione/expanddown.html)를 참조해 주십시요.
  * 만약 W(bit 41)(Data Segments)가 1이면, write access가 허용되며, 만약 0이면, 이 segment는 read-only입니다. data segments에 대하여 read access는 항상 허용된다는 것을 주목하십시요.
  * A(bit 40)는 processor가 본 segment를 access할 수 있는지를 control합니다.
  * C(bit 43)는 conforming bit(code selectors)입니다. 만약 C가 1이면, segment code는 lower privilege (예를 들면 user) level에서 수행이 가능합니다. 만약 C가 0이면, 오직 동일한 privilege level에서만 수행이 가능합니다.
  * R(bit 41)은 code segments에 read access를 control합니다; 1이면, segment는 read될 수 있습니다. code segments에 대해서 write access는 허용되지 않습니다.

4. DPL[2-bits] (Descriptor Privilege Level)는 bits 45-46로 구성됩니다. 이것은 segment의 privilege level을 정의합니다. 이것은 0-3이 될 수 있으며 0 이 가장 privileged level 입니다.

5. P flag(bit 47)는 이 segment가 memory에서의 존재 여부를 나타냅니다. 만약 P가 0이면, segment는 _invalid_한 상태로 processor가 segment로부터 read할 경우 거부됩니다.

6. AVL flag(bit 52)는 Available and reserved bits 입니다. linux에서는 사용되지 않습니다.

7. L flag(bit 53)는 code segment가 native 64-bit code를 포함하고 있는지를 나타냅니다. 만약 set되어 있으면, code segment는 64-bit mode로 실행됩니다.

8. D/B flag(bit 54) (Default/Big flag)는 operand의 size 즉 16/32 bits를 나타냅니다. 만약 set되어 있으면, operand의 size는 32 bits이고, 그렇지 않으면, 16 bits 입니다.

real mode에서 segment registers는 segment selectors를 포함합니다. 하지만, protected mode에서는 segment selector가 다르게 다뤄집니다. 각 segment descriptor는 연관된 16-bit structure인 segment selector를 가지고 있습니다:

```
 15             3 2  1     0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

여기서,
* **Index**는 GDT에서 descriptor의 index number를 저장합니다.
* **TI**(Table Indicator)는 descriptor를 찾을 장소를 표시합니다. 만약 0이면 descriptor를 Global Descriptor Table(GDT)에서 찾습니다. 그렇지 않으면, Local Descriptor Table(LDT)에서 찾습니다.
* 그리고 **RPL**은 Requester의 Privilege Level을 포함합니다.

모든 segment register는 visible한 부분과 hidden한 부분이 있습니다.
* Visible - Segment Selector가 여기에 저장됩니다.
* Hidden -  Segment Descriptor (base, limit, attributes & flags를 포함하고 있는)가 여기에 저장됩니다.

protected mode에서 physical address를 얻기 위해서 다음과 같은 steps이 필요합니다:

* segment selector가 segment registers 중 하나에 load되어야 합니다.
* CPU는 selector로부터 `GDT address + Index` offset에서 segment descriptor를 찾아 segment register의 _hidden_ part에 load합니다.
* 만약 paging이 disable되어 있으면, segment의 linear address 혹은 physical address는 다음의 공식으로 얻어집니다: Base address (이전 step에서 찾은 descriptor에서 찾은) + offset.

도식적으로 다음과 같습니다:

![linear address](http://oi62.tinypic.com/2yo369v.jpg)

real mode에서 protected mode로 천이하기 위한 algorithm은 다음과 같습니다:

* interrupts를 disable
* `lgdt` instruction으로 GDT를 기술하고 load
* CR0 (Control Register 0)의 PE (Protection Enable) bit을 set
* protected mode code로 jump

다음 part에서 linux kernel에서 protected mode로 천이를 완료하는 것을 볼 것입니다. 하지만 protected mode로 천이하기 전에, 좀 더 준비가 필요합니다.

[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)를 살펴봅시다. keyboard 초기화, heap 초기화 등을 수행하는 routines을 볼 수 있습니다. 살펴봅시다.

Copying boot parameters into the "zeropage"
--------------------------------------------------------------------------------

"main.c"의 `main` routine부터 시작할 것입니다. `main`에서 호출되는 첫번째 function은 [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) 입니다. 이것은 kernel setup header를 [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h) header file에 정의되어 있는 `boot_params` structure의 해당하는 field로 copy합니다.

`boot_params` structure는 `struct setup_header hdr` field를 포함합니다. 이 structure는 [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)에서 정의한 동일한 fields를 포함하고 있고, 이것은 bootloader와 kernel compile/build시 채워집니다. `copy_boot_params`는 두가지 일을 합니다:

1. 그것은 [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L280)의 `hdr`을 `boot_params` structure의 `setup_header` field로 copy합니다.

2. kernel이 old command line protocol로 load된 경우 kernel command line으로의 pointer를 update합니다.

이것은 [copy.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S) source file 정의되어 있는 `memcpy` function을 사용하여 `hdr`을 copy하는 것에 주목하십시요. 내부를 살펴 봅시다:

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```

네, 이제 막 C code로 왔는데 다시 assembly 입니다 :) 무엇보다도 먼저, `memcpy`와 여기서 정의된 다른 routines을 볼 수 있습니다. 시작과 끝은 다음의 2개의 marcros를 사용하고 있습니다: `GLOBAL`과 `ENDPROC`. `GLOBAL`은 `global` directive와 label이 정의되어 있는 [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/linkage.h)에 기술되어 있습니다. `ENDPROC`은 [include/linux/linkage.h](https://github.com/torvalds/linux/blob/v4.16/include/linux/linkage.h)에 기술되어 있고 `name` symbol이 function name으로 mark되어 있고 `name` symbol의 size로 끝납니다.

이 `memcpy`의 구현은 간단합니다. 처음에, `memcpy`에서 변경될 `si`와 `di` registers의 값을 stack에 push합니다. `arch/x86/Makefile`에서 `REALMOD_CFLAGS`에서 볼 수 있는 것처럼, kernel build system은 GCC의 `-mregparm=3` option을 사용합니다. 그렇기 때문에 functions은 처음 3개의 parameters를 `ax`, `dx`, 그리고 `cx` registers로부터 얻습니다. `memcpy`의 호출은 다음과 같습니다:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

그래서,
* `ax`는 `boot_params.hdr`의 address를 포함할 것입니다.
* `dx`는 `hdr`의 addresss를 포함할 것입니다.
* `cx`는 `hdr`의 size를 bytes로 포함할 것입니다.

`memcpy`는 `boot_param hdr`의 address를 `di`에 설정하고 `cx`를 stack에 저장합니다. 이후 `cx`의 값을 오른쪽으로 2번 shift 합니다 (혹은 4로 나눕니다). 그리고 `si`의 address로부터 `di`의 address로 4 bytes를 복사합니다. 이 이후, `hdr`의 크기를 다시 restore하고, 4 bytes에 align을 맞추고 (만약 남아 있다면) `si`의 address로부터 `di`의 address로 나머지 bytes를 byte 단위로 복사합니다. 이제 `si`와 `di`의 값을 stack으로부터 복원하고 copy operation을 끝냅니다.

Console initialization
--------------------------------------------------------------------------------

`hdr`이 `boot_params.hdr`로 copy된 다음, 다음 step은 [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/early_serial_console.c)에 정의되어 있는 `console_init` function을 호출하여 console을 초기화하는 것입니다.

그것은 command line에서 `earlyprintk` option을 찾고 그 option이 발견되면, serial port의 port address와 baud rate를 parse하여 serial port를 초기화합니다. `earlyprintk` command line option은 다음과 같이 될 수 있습니다:

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

serial port 초기화 후, 첫번째 출력을 볼 수 있습니다:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

`puts`의 정의는 [tty.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c)에 있습니다. 보시는 바와 같이 loop에서 `putchar` function을 호출하여 문자별로 출력합니다. `putchar`의 구현을 살펴봅시다:

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

`__attribute__((section(".inittext")))`는 이 code가 `.inittext` section에 위치할 것이라는 것을 의미합니다. 이 section은 linker file [setup.ld](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)에서 찾아 볼 수 있습니다.

무엇보다도 먼저, `putchar`은 `\n` symbol 인지 확인하고, 맞다면 `\r`을 먼저 출력합니다. 이후 BIOS의 `0x10` interrupt call을 호출하여 VGA에 문자를 출력합니다:

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

여기서 `initregs`는 `biosregs` structure를 인자로 받아 `memset` function을 사용하여 0으로 채우고 register의 값으로 채웁니다.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

[memset](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S#L36)의 구현을 살펴봅시다:

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

위에서 읽을 수 있는 것처럼, 인자를 `ax`, `dx` 그리고 `cx` registers로 받는 `memcpy` function과 동일한 호출 관례를 사용합니다.

`memset`의 구현은 `memcpy`와 유사합니다. `di` register의 값을 stack에 저장하고 `biosregs`의 address가 저장되어 있는 `ax`의 값을 `di`에 설정합니다. 다음은 `dl`의 값을 `eax` register의 하위 2 bytes로 복사하는 `movzbl` instruction입니다. `eax`의 남은 상위 2 bytes는 0으로 채워집니다.

다음 instruction은 `eax`와 `0x01010101`을 곱합니다. 이것은 `memset`이 한번에 4 bytes를 복사하기 때문에 필요합니다. 예를 들면, `memset`으로 `0x7`이라는 값으로 4 bytes의 크기의 structure를 채운다면, `eax`는 `0x00000007`을 포함할 것 입니다. 그래서 `eax`에 `0x01010101`을 곱하여 `0x07070707`을 얻이 이 4 bytes를 structure에 복사하는 것입니다. `memset`은 `eax`를 `es:di`로 copy하기 위해 `rep; stosl` instruction을 사용합니다.

`memset` function의 나머지는 `memcpy`와 거의 동일합니다.

`memset`을 이용하여 `biosregs` structure를 채운 후, `bios_putchar`은 문자를 출력하는 [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt를 호출합니다. 그 후에 serial port가 초기화 되었는지 확인하고, 만약 초기화 되었다면 [serial_putchar](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c)과 `inb/outb` instructions으로 serial port에 문자를 출력합니다.

Heap initialization
--------------------------------------------------------------------------------

[header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) (이전 [part](linux-bootstrap-1.md) 참조)ㅇ서 stack과 bss section이 준비된 후, kernel은 [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) function으로 [heap](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)을 초기화 합니다.

무엇보다도 먼저 `init_heap`은 kernel setup header에서 [`loadflags`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) structure로부터 [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L24) flag를 확인하고 만약 이 flag가 set되어 있으면 stack의 end를 계산합니다.

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

즉 `stack_end = esp - STACK_SIZE`.

그리고 `heap_end`를 계산합니다:

```C
     heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

이것은 `heap_end_ptr` 혹은 `_end` + `512` (`0x200h`)를 의미합니다. 마지막으로 확인하는 것은 `heap_end`가 `stack_end`보다 큰지 여부입니다. 만약 그렇다면 `stack_end`를 `heap_end`로 설정하여 동일하게 만듭니다.

이제 heap이 초기화되었고 `GET_HEAP` method를 이용하여 사용할 수 있습니다. 다음 post에서 무엇때문에 사용되는지, 어떻게 사용되는지 그리고 어떻게 구현되어 있는지를 살펴 볼 것입니다.

CPU validation
--------------------------------------------------------------------------------

우리가 살펴 볼 다음 setp은 [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpu.c) source code file의 `validate_cpu` function을 통한 cpu validation 입니다.

이 function은 [`check_cpu`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpucheck.c) function를 호출하고 cpu level과 요구되어지는 cpu level을 넘겨서 kernel이 적절한 cpu level에서 실행되는지 확인합니다.

```C
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```

`check_cpu` function은 CPU의 flags와 x86_64 (64-bit) CPU의 경우 [long mode](http://en.wikipedia.org/wiki/Long_mode) 가용 여부를 확인하고 processor의 vendor를 확인하고 AMD의 경우 SSE+SSE2를 끄는 등 특정 vendor를 위한 준비를 합니다.

다음으로 setup code가 CPU가 적절하다는 것을 확인한 후 `set_bios_mode` function이 호출되는 것을 볼 수 있습니다. 하기에서 보이는 것과 같이, 이 function은 `x86_64` mode에서만 구현되어 있습니다.

```C
static void set_bios_mode(void)
{
#ifdef CONFIG_X86_64
	struct biosregs ireg;

	initregs(&ireg);
	ireg.ax = 0xec00;
	ireg.bx = 2;
	intcall(0x15, &ireg, NULL);
#endif
}
```

`set_bios_mode` function는 (만약 `bx == 2`라면) [long mode](https://en.wikipedia.org/wiki/Long_mode)가 사용될 것이라고 BIOS에게 알리는 `0x15` BIOS interrupt를 실행합니다.

Memory detection
--------------------------------------------------------------------------------

다음 step은 `detect_memory` function을 통한 memory detection 입니다. `detect_memory`는 기본적으로 가용한 RAM의 map을 CPU에 제공합니다. 이것은 `0xe820`, `0xe801` 그리고 `0x88` 과 같은 memory detection을 위한 상이한 programming interface를 사용합니다. 여기서는 오직 **0xE820** interface의 구현만 살펴볼 것입니다.

[arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/memory.c) source file에서 `detect_memory_e820` function의 구현을 살펴봅시다. 무엇보다도 먼저 `detect_memory_e820` function은 하기에서 보이는 것 처럼 `biosregs`를 초기화하고 `0xe820` 호출을 위해 특별한 값으로 채웁니다:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax`는 function의 number를 포함합니다 (우리의 경우 0xe820)
* `cx`는 memory에 대한 data가 포함될 buffer의 size를 포함합니다
* `edx`는 `SMAP` magic number를 포함해야 합니다
* `es:di`는 memory data를 포함할 buffer의 address를 포함해야 합니다
* `ebx`는 0이어야 합니다.

다음은 memory에 대한 data를 수집하는 loop입니다. address allocation table로부터 한줄을 쓰는 `0x15` BIOS interrupt를 호출하는 것으로 시작합니다. 다음줄을 얻기 위해서 (loop에서 하는) 이 interrupt를 다시 호출해야 합니다. 다음 호출 전 `ebx`는 이전에 return된 값을 포함하고 있어야 합니다:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

결국, 이 function은 address allocation table로부터 data를 수집하여 이 data를 `e820_entry` array에 기록합니다:

* memory segment의 시작
* memory segment의 size
* memory segment의 type (특정 segment가 사용 가능한지 혹은 예약되어 있는지)

이 결과는 아래와 같이 `dmesg`의 출력으로 볼 수 있습니다:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

Keyboard initialization
--------------------------------------------------------------------------------

다음 step은 [`keyboard_init`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) function을 호출하여 keyboard를 초기화하는 것 입니다. 처음에 `keyboard_init`은 `initregs` function을 사용하여 registers를 초기화합니다. 그리고나서 keyboard의 status를 query하는 [0x16](http://www.ctyme.com/intr/rb-1756.htm) interrupt를 호출합니다.

```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```

이 이후 keyboard의 repeat rate와 delay를 설정하기 위해 다시 [0x16](http://www.ctyme.com/intr/rb-1757.htm)를 호출합니다.

```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Querying
--------------------------------------------------------------------------------

다음 2개의 steps은 다른 parameters를 위한 queries 입니다. 이 queries에 대하여 자세히 다루지는 않을 것이지만 이후 part에서 살펴볼 것입니다. 이 functions에 대해 간단히 살펴봅시다:

첫번째 step은 `query_ist` function을 호출하여 [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) 정보를 얻는 것입니다. 이것은 CPU level을 확인하고 적절하다면, 해당 정보를 얻는 `0x15`를 호출하여 결과를 `boot_params`에 저장합니다.

다음은 BIOS로부터 [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) 정보를 얻는 [query_apm_bios](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/apm.c#L21) function입니다. `query_apm_bios`도 `0x15` BIOS interrupt를 호출하지만, `ah` = `0x53`으로 `APM` 설치를 확인합니다. `0x15`의 실행이 완료되면, `query_apm_bios` functions은 `PM` signature (`0x504d` 이어야 함), carry flag (`APM`이 지원되는 경우 0이어야 함), `cx` register의 값 (0x02이면 protected mode interface가 지원됨)을 확인합니다.

다음으로 `0x15`를 다시 호출하지만, `ax` = ``0x5304`로 `APM` interface를 끊고 32-bit protected mode interface로 연결합니다. 마지막으로, BIOS로부터 얻은 값으로 `boot_params.apm_bios_info`를 채웁니다.

`query_apm_bios`는 오직 configuration file에 `CONFIG_APM` 혹은 `CONFIG_APM_MODULE` flag가 set되어 있어야 실행된다는 것에 주목하십시요: 

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

마지막은 [`query_edd`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/edd.c#L122) function로, 이것은 BIOS로부터 `Enhanced Disk Drive` 정보를 질의합니다. `query_edd`의 구현을 살펴봅시다.

무엇보다도 먼저, kernel의 command line으로부터 [edd](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst) option을 읽고 `off`로 되어 있으면 그냥 `query_edd`를 return합니다.

만약 EDD가 enable되어 있으면, `query_edd`는 하기 loop에서 BIOS-supported hard disk의 EDD 정보를 질의합니다:

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    ...
    }
```

여기서 `0x80`은 첫번째 hard drive이고 `EDD_MBR_SIG_MAX` macro는 16입니다. 이 function은 data를 [edd_info](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/edd.h) structures array로 수집합니다. `get_edd_info`는 `ah`를 `0x41`로 하고 `0x13` interrupt를 호출하여 EDD가 가용함을 확인하고 EDD가 가용하면, `get_edd_info`는 `ah`를 `0x48`로 하고 `si`에 EDD 정보를 저장할 buffer의 address를 설정하여 `0x13` interrupt를 다시 호출합니다.

Conclusion
--------------------------------------------------------------------------------

이것이 linux kernel의 내부에 대한 두번째 part의 끝입니다. 다음 part에서는 video mode setting과 protected mode로 천이하기 전 나머지 준비와 protected mode로의 천이를 살펴볼 것입니다.

질문이나 제안이 있으시면 comment나 [twitter](https://twitter.com/0xAX)로 연락주십시요.

**영어는 저의 모국어가 아닙니다. 불편한 점이 있으셨다면 먼저 사죄드립니다. 만약 잘못된 점을 발견하시면 [linux-insides](https://github.com/0xAX/linux-internals)로 PR을 보내주십시요.**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst)
* [Serial console](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/serial-console.rst)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [Previous Part](linux-bootstrap-1.md)
