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
  * 만약 W(bit 41)(Data Segments)가 1이면, write access가 허용되며, 만약 0이면, 이 segment는 read-only입니다. data segments에 대하여 read access는 항상 허용된다는 것을 주의하십시요.
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

We will start from the `main` routine in "main.c". The first function which is called in `main` is [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). It copies the kernel setup header into the corresponding field of the `boot_params` structure which is defined in the [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h) header file.

The `boot_params` structure contains the `struct setup_header hdr` field. This structure contains the same fields as defined in the [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) and is filled by the boot loader and also at kernel compile/build time. `copy_boot_params` does two things:

1. It copies `hdr` from [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L280) to the `setup_header` field in `boot_params` structure.

2. It updates the pointer to the kernel command line if the kernel was loaded with the old command line protocol.

Note that it copies `hdr` with the `memcpy` function, defined in the [copy.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S) source file. Let's have a look inside:

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

Yeah, we just moved to C code and now assembly again :) First of all, we can see that `memcpy` and other routines which are defined here, start and end with the two macros: `GLOBAL` and `ENDPROC`. `GLOBAL` is described in [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/linkage.h) which defines the `globl` directive and its label. `ENDPROC` is described in [include/linux/linkage.h](https://github.com/torvalds/linux/blob/v4.16/include/linux/linkage.h) and marks the `name` symbol as a function name and ends with the size of the `name` symbol.

The implementation of `memcpy` is simple. At first, it pushes values from the `si` and `di` registers to the stack to preserve their values because they will change during the `memcpy`. As we can see in the `REALMODE_CFLAGS` in `arch/x86/Makefile`, the kernel build system uses the `-mregparm=3` option of GCC, so functions get the first three parameters from `ax`, `dx` and `cx` registers.  Calling `memcpy` looks like this:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

So,
* `ax` will contain the address of `boot_params.hdr`
* `dx` will contain the address of `hdr`
* `cx` will contain the size of `hdr` in bytes.

`memcpy` puts the address of `boot_params.hdr` into `di` and saves `cx` on the stack. After this it shifts the value right 2 times (or divides it by 4) and copies four bytes from the address at `si` to the address at `di`. After this, we restore the size of `hdr` again, align it by 4 bytes and copy the rest of the bytes from the address at `si` to the address at `di` byte by byte (if there is more). Now the values of `si` and `di` are restored from the stack and the copying operation is finished.

Console initialization
--------------------------------------------------------------------------------

After `hdr` is copied into `boot_params.hdr`, the next step is to initialize the console by calling the `console_init` function,  defined in [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/early_serial_console.c).

It tries to find the `earlyprintk` option in the command line and if the search was successful, it parses the port address and baud rate of the serial port and initializes the serial port. The value of the `earlyprintk` command line option can be one of these:

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

After serial port initialization we can see the first output:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

The definition of `puts` is in [tty.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c). As we can see it prints character by character in a loop by calling the `putchar` function. Let's look into the `putchar` implementation:

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

`__attribute__((section(".inittext")))` means that this code will be in the `.inittext` section. We can find it in the linker file [setup.ld](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld).

First of all, `putchar` checks for the `\n` symbol and if it is found, prints `\r` before. After that it prints the character on the VGA screen by calling the BIOS with the `0x10` interrupt call:

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

Here `initregs` takes the `biosregs` structure and first fills `biosregs` with zeros using the `memset` function and then fills it with register values.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

Let's look at the implementation of [memset](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S#L36):

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

As you can read above, it uses the same calling conventions as the `memcpy` function, which means that the function gets its parameters from the `ax`, `dx` and `cx` registers.

The implementation of `memset` is similar to that of memcpy. It saves the value of the `di` register on the stack and puts the value of`ax`, which stores the address of the `biosregs` structure, into `di` . Next is the `movzbl` instruction, which copies the value of `dl` to the lower 2 bytes of the `eax` register. The remaining 2 high bytes  of `eax` will be filled with zeros.

The next instruction multiplies `eax` with `0x01010101`. It needs to because `memset` will copy 4 bytes at the same time. For example, if we need to fill a structure whose size is 4 bytes with the value `0x7` with memset, `eax` will contain the `0x00000007`. So if we multiply `eax` with `0x01010101`, we will get `0x07070707` and now we can copy these 4 bytes into the structure. `memset` uses the `rep; stosl` instruction to copy `eax` into `es:di`.

The rest of the `memset` function does almost the same thing as `memcpy`.

After the `biosregs` structure is filled with `memset`, `bios_putchar` calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt which prints a character. Afterwards it checks if the serial port was initialized or not and writes a character there with [serial_putchar](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c) and `inb/outb` instructions if it was set.

Heap initialization
--------------------------------------------------------------------------------

After the stack and bss section have been prepared in [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) (see previous [part](linux-bootstrap-1.md)), the kernel needs to initialize the [heap](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) with the [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) function.

First of all `init_heap` checks the [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L24) flag from the [`loadflags`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) structure in the kernel setup header and calculates the end of the stack if this flag was set:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

or in other words `stack_end = esp - STACK_SIZE`.

Then there is the `heap_end` calculation:

```C
     heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

which means `heap_end_ptr` or `_end` + `512` (`0x200h`). The last check is whether `heap_end` is greater than `stack_end`. If it is then `stack_end` is assigned to `heap_end` to make them equal.

Now the heap is initialized and we can use it using the `GET_HEAP` method. We will see what it is used for, how to use it and how it is implemented in the next posts.

CPU validation
--------------------------------------------------------------------------------

The next step as we can see is cpu validation through the `validate_cpu` function from [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpu.c) source code file.

It calls the [`check_cpu`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpucheck.c) function and passes cpu level and required cpu level to it and checks that the kernel launches on the right cpu level.

```C
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```

The `check_cpu` function checks the CPU's flags, the presence of [long mode](http://en.wikipedia.org/wiki/Long_mode) in the case of x86_64(64-bit) CPU, checks the processor's vendor and makes preparations for certain vendors like turning off SSE+SSE2 for AMD if they are missing, etc.

at the next step, we may see a call to the `set_bios_mode` function after setup code found that a CPU is suitable. As we may see, this function is implemented only for the `x86_64` mode:

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

The `set_bios_mode` function executes the `0x15` BIOS interrupt to tell the BIOS that [long mode](https://en.wikipedia.org/wiki/Long_mode) (if `bx == 2`) will be used.

Memory detection
--------------------------------------------------------------------------------

The next step is memory detection through the `detect_memory` function. `detect_memory` basically provides a map of available RAM to the CPU. It uses different programming interfaces for memory detection like `0xe820`, `0xe801` and `0x88`. We will see only the implementation of the **0xE820** interface here.

Let's look at the implementation of the `detect_memory_e820` function from the [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/memory.c) source file. First of all, the `detect_memory_e820` function initializes the `biosregs` structure as we saw above and fills registers with special values for the `0xe820` call:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` contains the number of the function (0xe820 in our case)
* `cx` contains the size of the buffer which will contain data about the memory
* `edx` must contain the `SMAP` magic number
* `es:di` must contain the address of the buffer which will contain memory data
* `ebx` has to be zero.

Next is a loop where data about the memory will be collected. It starts with a call to the `0x15` BIOS interrupt, which writes one line from the address allocation table. For getting the next line we need to call this interrupt again (which we do in the loop). Before the next call `ebx` must contain the value returned previously:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Ultimately, this function collects data from the address allocation table and writes this data into the `e820_entry` array:

* start of memory segment
* size  of memory segment
* type of memory segment (whether the particular segment is usable or reserved)

You can see the result of this in the `dmesg` output, something like:

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

The next step is the initialization of the keyboard with a call to the [`keyboard_init`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) function. At first `keyboard_init` initializes registers using the `initregs` function. It then calls the [0x16](http://www.ctyme.com/intr/rb-1756.htm) interrupt to query the status of the keyboard.

```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```

After this it calls [0x16](http://www.ctyme.com/intr/rb-1757.htm) again to set the repeat rate and delay.

```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Querying
--------------------------------------------------------------------------------

The next couple of steps are queries for different parameters. We will not dive into details about these queries but we will get back to them in later parts. Let's take a short look at these functions:

The first step is getting [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) information by calling the `query_ist` function. It checks the CPU level and if it is correct, calls `0x15` to get the info and saves the result to `boot_params`.

Next, the [query_apm_bios](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/apm.c#L21) function gets [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) information from the BIOS. `query_apm_bios` calls the `0x15` BIOS interruption too, but with `ah` = `0x53` to check `APM` installation. After `0x15` finishes executing, the `query_apm_bios` functions check the `PM` signature (it must be `0x504d`), the carry flag (it must be 0 if `APM` supported) and the value of the `cx` register (if it's 0x02, the protected mode interface is supported).

Next, it calls `0x15` again, but with `ax = 0x5304` to disconnect the `APM` interface and connect the 32-bit protected mode interface. In the end, it fills `boot_params.apm_bios_info` with values obtained from the BIOS.

Note that `query_apm_bios` will be executed only if the `CONFIG_APM` or `CONFIG_APM_MODULE` compile time flag was set in the configuration file:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

The last is the [`query_edd`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/edd.c#L122) function, which queries `Enhanced Disk Drive` information from the BIOS. Let's look at how `query_edd` is implemented.

First of all, it reads the [edd](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst) option from the kernel's command line and if it was set to `off` then `query_edd` just returns.

If EDD is enabled, `query_edd` goes over BIOS-supported hard disks and queries EDD information in the following loop:

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

where `0x80` is the first hard drive and the value of the `EDD_MBR_SIG_MAX` macro is 16. It collects data into an array of [edd_info](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/edd.h) structures. `get_edd_info` checks that EDD is present by invoking the `0x13` interrupt with `ah` as `0x41` and if EDD is present, `get_edd_info` again calls the `0x13` interrupt, but with `ah` as `0x48` and `si` containing the address of the buffer where EDD information will be stored.

Conclusion
--------------------------------------------------------------------------------

This is the end of the second part about the insides of the Linux kernel. In the next part, we will see video mode setting and the rest of the preparations before the transition to protected mode and directly transitioning into it.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-internals).**

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
