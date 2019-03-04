Kernel booting process. Part 4.
================================================================================

Transition to 64-bit mode
--------------------------------------------------------------------------------

이것은 `kernel booting process`의 네번째 part입니다. 여기서 CPU가 [long mode](http://en.wikipedia.org/wiki/Long_mode)와 [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)를 지원하는지 확인하는 것과 같은 [protected mode](http://en.wikipedia.org/wiki/Protected_mode)에서의 첫번째 step과 page table을 초기화 하는 것과 마지막으로 [long mode](https://en.wikipedia.org/wiki/Long_mode)로 천이하는 것에 대하여 논의할 것입니다.

**NOTE: 이 part에서는 많은 assembly code가 나옵니다. 만약 이에 친숙하지 않다면 관련 서적을 참조해야 할 수도 있습니다.**

이전 [part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)에서 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S)의 `32-bit` entry point로 jump한 곳에서 멈췄습니다:

```assembly
jmpl	*%eax
```

`eax` register가 32-bit entry point의 address를 가지고 있었다는 것을 기억하실 겁니다. [linux kernel x86 boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)에서 이것에 대해 읽어 볼 수 있습니다:

```
When using bzImage, the protected-mode kernel was relocated to 0x100000
```

32-bit entry point에서 register의 값을 확인해서 이것이 사실인지 확인해 봅시다:

```
eax            0x100000	1048576
ecx            0x0	    0
edx            0x0	    0
ebx            0x0	    0
esp            0x1ff5c	0x1ff5c
ebp            0x0	    0x0
esi            0x14470	83056
edi            0x0	    0
eip            0x100000	0x100000
eflags         0x46	    [ PF ZF ]
cs             0x10	16
ss             0x18	24
ds             0x18	24
es             0x18	24
fs             0x18	24
gs             0x18	24
```

여기서 `cs` register는 `0x10`을 가지고 있고, (이전 [part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)에서 `Golbal Descriptor Table`의 두번째 index였습니다) `eip` register는 `0x100000`을 가지고 있고 code segment를 포함하여 모든 segment의 base address는 0입니다.

그래서 boot protocol에서 기술한 것처럼 physical address는 `0:0x00000` 혹은 `0x00000`이 됩니다. 이제 `32-bit` entry point에서 시작해 봅시다.

32-bit entry point
--------------------------------------------------------------------------------

`32-bit` entry point의 선언은 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) assembly source code file에서 찾아볼 수 있습니다:

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
....
....
....
ENDPROC(startup_32)
```

먼저, 왜 directory의 이름이 `compressed`일까요? 실제 `bzimage`는 `vmlinux + header + kerenl setup code`를 gzip으로 압축한 것입니다. 이전 part에서 kernel setup code를 살펴보았습니다. `head_64.S`의 주요 목적은 long mode로 진입하기 위해 준비하는 것, long mode로 진입하는 것, 그리고 kernel의 압축을 해제하는 것입니다. 이 part에서 kernel 압축 해제를 위한 모든 step을 살펴볼 것입니다.

`arch/x86/boot/compressed` directory에서 2개의 file을 찾을 수 있을 겁니다:

* [head_32.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_32.S)
* [head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)

하지만, 이 책은 오직 `x86_64`에 대해서만 다루기 때문에 `head_64.S` source code file에 대해서만 다룰 것입니다; [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/Makefile)를 살펴봅시다. 여기서 다음과 같은 `make` target을 찾을 수 있습니다:

```Makefile
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

`$(obj)/head_$(BITS).o`를 살펴봅시다.

이것은 `$(BITS)`에 무엇이 설정되어 있는지에 따라 `head_32.o` 혹은 `head_64.o`의 file이 link된다는 것을 의미합니다. `$(BITS)` 변수는 kernel configuration에 따라 [arch/x86/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)에 정의되어 있습니다:

```Makefile
ifeq ($(CONFIG_X86_32),y)
        BITS := 32
        ...
        ...
else
        BITS := 64
        ...
        ...
endif
```

이제 어디서 시작해야 할지를 알았습니다. 살펴봅시다.

Reload the segments if needed
--------------------------------------------------------------------------------

위에서 살펴본 데로, [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S) assembly source code file에서 시작합니다. 먼저 `startup_32` 정의 전에 special section attribute의 정의를 볼 수 있습니다:

```assembly
    __HEAD
    .code32
ENTRY(startup_32)
```

`__HEAD` macro는 [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) header file에 정의되어 있고 이것은 다음 section의 정의로 확장됩니다:

```C
#define __HEAD		.section	".head.text","ax"
```

이것은 `.head.text`라는 이름과 `ax`라는 flags의 section 입니다. 이 flags는 이 section이 [executable](https://en.wikipedia.org/wiki/Executable), 다시 말하면 code를 포함하고 있다는 것을 의미합니다. 이 section의 정의는 [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/vmlinux.lds.S) linker script에서 찾아볼 수 있습니다:

```
SECTIONS
{
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
     }
     ...
     ...
     ...
}
```

만약 `GNU LD` linker scripting language의 syntax에 친숙하지 않다면, [documentation](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)에서 자세한 내용을 찾아볼 수 있습니다. 요약하면, `.` symbol은 linker의 special 변수로 location counter 입니다. 값은 segment의 offset로의 상대 offset으로 설정됩니다. 이 경우 location counter는 0으로 설정됩니다. 이것은 이 code는 memory에서 offset `0`으로부터 실행되도록 link된다는 것을 의미합니다. 게다가 comment에서 하기와 같은 정보를 찾을 수 있습니다:

```
Be careful parts of head_64.S assume startup_32 is at address 0.
```

좋습니다, 이제 어디에 있는지 알았습니다. 이제 `startup_32` 함수를 살펴볼 차례입니다.

`startup_32` 함수의 도입부에서 [flags](https://en.wikipedia.org/wiki/FLAGS_register) register에서 `DF` bit을 clear하는 `cld` instruction을 볼 수 있습니다. direction flag가 clear되면 [stos](http://x86.renejeschke.de/html/file_module_x86_id_306.html), [scas](http://x86.renejeschke.de/html/file_module_x86_id_287.html)와 같은 모든 string operation은 `esi`나 `edi`의 index register를 증가시키게 됩니다. 나중에 page table 등의 공간을 clear시키기 위해 string operation을 사용할 것이기 때문에 direction flag를 clear시켜야 합니다.

`DF` bit을 clear한 후, 다음 step은 `loadflags` kernel setup header field의 `KEEP_SEGMENTS` flag를 확인하는 것입니다. 이 책의 첫 [part](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html)에서 heap의 사용 가능 여부를 얻는 `CAN_USE_HEAP` flag를 확인하기 위해 `loadflags`를 살펴보았습니다. 여기서는 `KEEP_SEGMENTS` flag를 확인합니다. 이 flag는 linux [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) documentation에 기술되어 있습니다:

```
Bit 6 (write): KEEP_SEGMENTS
  Protocol: 2.07+
  - If 0, reload the segment registers in the 32bit entry point.
  - If 1, do not reload the segment registers in the 32bit entry point.
    Assume that %cs %ds %ss %es are all set to flat segments with
		a base of 0 (or the equivalent for their environment).
```

`loadflags`의 `KEEP_SEGMENTS` bit이 설정되어 있지 않다면, `ds`, `ss`, `es`의 segment register를 base가 0인 data segment의 index로 설정해야 합니다. 다음과 같이 설정합니다:

```C
	testb $KEEP_SEGMENTS, BP_loadflags(%esi)
	jnz 1f

	cli
	movl	$(__BOOT_DS), %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
```

`__BOOT_DS`가 `0x18` ([Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)에서 data segment의 index) 라는 것을 기억하십시요. 만약 `KEEP_SEGMENTS`가 set되어 있다면 가장 가까운 `1f` label로 jump하고, 만약 set되어 있지 않다면 segment register를 `__BOOT_DS`로 update합니다. 이것은 쉽습니다만 흥미로운 부분이 있습니다. 만약 이전 [part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)를 읽었다면 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S)에서 [protected mode](https://en.wikipedia.org/wiki/Protected_mode)로 변경한 직후 이미 segment register를 update한 것을 기억할 것입니다. 왜 다시 segment register의 값을 조정해 줘야 하는 것일까요? 답은 간단합니다. linux kernel은 32-bit boot protocol을 가지고 있고 bootloader가 linux kernel을 load하기 위해 이것을 사용한다면 `startup_32` 이전의 모든 code가 miss될 것이기 때문입니다. 이 경우 `startup_32`는 bootloader 직후 linux kernel의 최초 entry point가 되기 때문에 segment register가 known state라는 보장이 없습니다.

`KEEP_SEGMENTS` flag를 확인한 후 적절한 값을 segment register에 넣습니다. 다음 step은 load된 곳과 compile된 곳의 차이를 계산하는 것입니다.  `setup.ld.S`가 `.head.text` section의 도입 부분에 `. = 0`라는 정의를 포함하고 있었다는 것을 기억하십시요. 이것은 이 section의 code는 address `0`에서 실행되도록 compile되었다는 것을 의미합니다. `onjdump`의 출력에서도 이것을 확인할 수 있습니다:

```
arch/x86/boot/compressed/vmlinux:     file format elf64-x86-64


Disassembly of section .head.text:

0000000000000000 <startup_32>:
   0:   fc                      cld
   1:   f6 86 11 02 00 00 40    testb  $0x40,0x211(%rsi)
```

`objdump` util은 `startup_32`의 address가 `0`이라고 report하지만 실제로는 그렇지 않습니다. 현재 저희의 goal은 어디서 실행되고 있는지를 아는 것입니다. [long mode](https://en.wikipedia.org/wiki/Long_mode)에서는 `rip` relative address를 지원하기 때문에 쉽게 알 수 있습니다만 지금은 [protected mode](https://en.wikipedia.org/wiki/Protected_mode) 입니다. `startup_32`의 address를 알기 위해 일반적인 pattern을 사용할 것 입니다. label을 정의하고 해당 label을 호출하고 stack의 top을 register로 pop합니다:

```assembly
call label
label: pop %reg
```

이렇게 하면 `%reg` register는 label의 address를 갖게 됩니다. linux kernel에서 `startup_32`의 address를 찾는 유사한 code를 살펴봅시다:

```assembly
        leal	(BP_scratch+4)(%esi), %esp
        call	1f
1:      popl	%ebp
        subl	$1b, %ebp
```

이전 part에서 기억하시는 것처럼, `esi` register는 protected mode로 이동하기 전에 채운 [boot_params](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L113) structure의 address를 가지고 있습니다. `boot_params` structure는 offset `0x1e4`에 `scratch`라는 특별한 field를 가지고 있습니다. 이 4 bytes field는 `call` instruction을 위한 임시 stack 입니다. `scratch` field의 address + `4`를 `esp` register에 넣습니다. `x86_64` architecture에서 stack pointer는 stack의 top을 가르키고 위에서 아래로 자라기 때문에 `BP_scratch` field에 `4` bytes를 더합니다. 이제 stack pointer가 stack의 top을 가르킵니다. 다음으로 위에서 언급한 pattern을 볼 수 있습니다. `1f` label을 호출하면 `call` instruction일 수행된 후 return address가 stack의 top에 저장되기 때문에 이 label의 address를 `ebp` register에 넣습니다. 그러면 `1f` label의 address를 알 수 있고 `startup_32`의 address도 쉽게 알 수 있습니다. stack으로부터 얻은 address로부터 address를 빼주면 됩니다:

```
startup_32 (0x0)     +-----------------------+
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
1f (0x0 + 1f offset) +-----------------------+ %ebp - real physical address
                     |                       |
                     |                       |
                     +-----------------------+
```

`startup_32`는 `0x0` address에서 실행되도록 link되었고 이것은 `1f`는 `0x0 + 1f의 offset` 약 `0x21` bytes의 address를 가진다는 것을 의미합니다. `ebp` register는 `1f` label의 physical address를 가지고 있습니다. 그렇기 때문에 `ebp`에서 `1f`를 빼주면 `startup_32`의 실제 physical address를 얻을 수 있습니다. linux kernel의 [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)은 protected mode kernel의 base address는 `0x100000`로 기술하고 있습니다. [gdb](https://en.wikipedia.org/wiki/GNU_Debugger)로 이것을 확인할 수 있습니다. debugger를 시작하고 `0x100021`인 `1f`의 address에 breakpoint를 설정해 봅시다. 만약 이것이 정확하다면 `ebp` register에서 `0x100021`을 볼 것입니다:

```
$ gdb
(gdb)$ target remote :1234
Remote debugging using :1234
0x0000fff0 in ?? ()
(gdb)$ br *0x100022
Breakpoint 1 at 0x100022
(gdb)$ c
Continuing.

Breakpoint 1, 0x00100022 in ?? ()
(gdb)$ i r
eax            0x18	0x18
ecx            0x0	0x0
edx            0x0	0x0
ebx            0x0	0x0
esp            0x144a8	0x144a8
ebp            0x100021	0x100021
esi            0x142c0	0x142c0
edi            0x0	0x0
eip            0x100022	0x100022
eflags         0x46	[ PF ZF ]
cs             0x10	0x10
ss             0x18	0x18
ds             0x18	0x18
es             0x18	0x18
fs             0x18	0x18
gs             0x18	0x18
```

다음 instruction `subl $1b, %ebp` 을 실행시키면, 다음을 볼 것입니다:

```
(gdb) nexti
...
...
...
ebp            0x100000	0x100000
...
...
...
```

네, 정확합니다. `startup_32`의 address는 `0x100000` 입니다. `startup_32` label의 address를 알아낸 후, [long mode](https://en.wikipedia.org/wiki/Long_mode)의 천이를 위한 준비를 할 수 있습니다. 우리의 다음 goal은 stack을 설정하고 CPU가 long mode와 [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)를 지원하는지 검증하는 것입니다.

Stack setup and CPU verification
--------------------------------------------------------------------------------

`startup_32` label의 address를 알지 못하면 stack을 설정할 수 없습니다. stack은 array이고 stack pointer register `esp`는 이 array의 끝을 가르키고 있어야 한다고 가정할 수 있습니다. 물론 code에서 array를 정의할 수 있지만 stack pointer를 정확히 설정하기 위해서 실제 address를 알아야 합니다. 다음 code를 살펴봅시다:

```assembly
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
```

`boot_stack_end` label은 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) assembly source code file에서 정의되고 [.bss](https://en.wikipedia.org/wiki/.bss) section에 위치합니다:

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

먼저, `boot_stack_end`의 address를 `eax` register에 넣으면 `eax` register는 link 시점의 `boot_stack_end`인 `0x0 + boot_stack_end`를 가지게 됩니다. `boot_stack_end`의 실제 address를 얻기 위해 `startup_32`의 실제 address를 더해야 합니다. 기억하시는 것처럼 위에서 이 address를 찾아 `ebp` register에 넣었습니다. 마침내 `eax` register는 `boot_stack_end`의 실제 address를 가지게 되고 이것을 stack pointer에 넣으면 됩니다.

stack을 설정한 후, 다음 step은 CPU 검증입니다. `long mode`로 천이할때 CPU가 `long mode`와 `SSE`를 지원하는지 확인해야 합니다. `verify_cpu` 함수를 호출하여 이 작업을 수행합니다:

```assembly
	call	verify_cpu
	testl	%eax, %eax
	jnz	no_longmode
```

이 함수는 [arch/x86/kernel/verify_cpu.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/kernel/verify_cpu.S) assembly file에 정의되어 있고 몇개의 [cpuid](https://en.wikipedia.org/wiki/CPUID) instruction 호출로 구성되어 있습니다. 이 instruction은 processor의 정보를 얻기 위해 사용됩니다. 이 경우 `long mode`와 `SSE` 지원 여부를 확인하며 `eax` register를 통해 성공시 `0`, 실패시 `1`을 return 합니다.

만약 `eax`의 값이 0이 아닌 경우 hardware interrupt가 발생하지 않는 동안 `hlt` instruction을 호출하여 CPU를 정지시키는 `no_longmode` label로 jump합니다:

```assembly
no_longmode:
1:
	hlt
	jmp     1b
```

`eax`의 값이 0이면 모든 것이 정상이고 계속 진행할 수 있습니다.

Calculate relocation address
--------------------------------------------------------------------------------

다음 step은 필요시 압축 해제를 위한 relocation address를 계산하는 것입니다. 먼저, kernel이 `relocatable`이라는 것이 무슨 의미인지 알아야 합니다. linux kernel의 32-bit entry point의 address가 `0x100000`라는 것은 이미 알고 있습니다. 하지만 이것은 32-bit entry point입니다. linux kernel의 default base address는 `CONFIG_PHYSICAL_START` kernel configuration option의 값에 의해 결정됩니다. 이것의 default 값은 `0x1000000` 혹은 `16MB` 입니다. 여기서 주요한 문제는 linux kernel이 crash되는 경우, kernel developer는 [kdump](https://www.kernel.org/doc/Documentation/kdump/kdump.txt)를 위해 다른 address에 load되도록 설정된 `rescue kernel`을 가지고 있어야 한다는 것입니다. linux kernel은 이 문제를 해결하기 위해 `CONFIG_RELOCATABLE`이라는 special configuration option을 제공합니다. linux kernel의 documentation에서 다음과 같은 부분을 볼 수 있습니다:

```
This builds a kernel image that retains relocation information
so it can be loaded someplace besides the default 1MB.

Note: If CONFIG_RELOCATABLE=y, then the kernel runs from the address
it has been loaded at and the compile time physical address
(CONFIG_PHYSICAL_START) is used as the minimum location.
```

간단히 말하면, 이것은 같은 configuration의 linux kernel이 다른 address에서 boot될 수 있다는 것입니다. 기술적으로 이것은 decompressor를 [position independent code](https://en.wikipedia.org/wiki/Position-independent_code)하게 compile하여 이루어 집니다. [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/Makefile)을 보면 decompressor는 `-fPIC` flag로 compile되는 것을 볼 수 있습니다:

```Makefile
KBUILD_CFLAGS += -fno-strict-aliasing -fPIC
```

position-independent code를 사용하면 address는 instruction의 address field와 program counter의 값을 더해서 얻어집니다. 이러한 address 방법을 사용하는 code는 어떤 address에도 load 할 수 있습니다. 이것이 `startup_32`의 실제 physical address를 얻어야 하는 이유입니다. 이제 linux kernel code로 돌아가 봅시다. 우리의 현재 goal은 압축 해제를 위해 kernel을 relocate할 수 있는 address를 계산하는 것입니다. 이 address의 계산은 `CONFIG_RELOCATABLE` kernel configuration option에 따라 달라집니다. 다음 code를 살펴봅시다:

```assembly
#ifdef CONFIG_RELOCATABLE
	movl	%ebp, %ebx
	movl	BP_kernel_alignment(%esi), %eax
	decl	%eax
	addl	%eax, %ebx
	notl	%eax
	andl	%eax, %ebx
	cmpl	$LOAD_PHYSICAL_ADDR, %ebx
	jge	1f
#endif
	movl	$LOAD_PHYSICAL_ADDR, %ebx
```

`ebp` register의 값은 `startup_32` label의 physical address라는 것을 기억하십시요.  만약 `CONFIG_RELOCATABLE` kernel configuration option이 enable되어 있으면, 이 address를 `2MB`에 align 시키고 `LOAD_PHYSICAL_ADDR` 값과 비교하여 `ebx` register에 넣습니다. `LOAD_PHYSICAL_ADDR` macro는 [arch/x86/include/asm/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/boot.h) header file에 다음과 같이 정의되어 있습니다:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

보시는 바와 같이 kernel이 load될 physical address를 `CONFIG_PHYSICAL_ALIGN` 값에 align시킵니다. `LOOAD_PHYSICAL_ADDR`와 `ebx` register 값을 비교한 후, 압축된 kernel image를 압축 해제한 곳에 `startup_32`로부터의 offset을 더합니다. 만약 `CONFIG_RELOCATABLE` option이 enable되어 있지 않다면, kernel을 load할 default address를 넣고 `z_extract_offset`을 더합니다.

이 계산이 모두 끝나면, `ebp`는 load되어 있는 address를 가지고 있고, `ebx`는 압축 해제후 kernel이 이동할 address를 가지게 됩니다. 하지만 이것이 끝이 아닙니다. 압축된 kernel image는 후에 kernel이 위치될 곳의 계산을 간단하게 하기 위해 압축 해제 buffer의 끝으로 이동되어야 합니다. 다음과 같습니다:

```assembly
1:
    movl	BP_init_size(%esi), %eax
    subl	$_end, %eax
    addl	%eax, %ebx
```

`boot_params.BP_init_size` (혹은 `hdr.init_size`의 kernel setup header 값) 의 값을 `eax` register에 넣습니다. `BP_init_size`는 압축된 것과 압축 해제된 [vmlinux](https://en.wikipedia.org/wiki/Vmlinux) 중 큰 값을 가집니다. 다음으로 이 값에서 `_end` symbol의 address를 빼고 그 값을 kernel 압축 해제를 위한 base address를 저장하는 `ebx` register에 더합니다.

Preparation before entering long mode
--------------------------------------------------------------------------------

압축된 kernel image를 relocate시킬 base address를 얻은 후, 64-bit mode로 천이하기 전 마지막 step이 필요합니다. 먼저, relocatable kernel은 512G 이하의 어떤 address에서도 실행될 수 있기 때문에 64-bit segment로 [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)를 update해야 합니다:

```assembly
	addl	%ebp, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

여기서 Global Descriptor table의 base address를 실제 load된 address로 수정하고 `lgdt` instruction으로 `Global Descriptor Table`을 load합니다.

마법같은 `gdt` offset을 이해하기 위해 `Global Descriptor Table`의 정의를 살펴볼 필요가 있습니다. 이 정의는 같은 source code [file](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)에서 찾아볼 수 있습니다:

```assembly
	.data
gdt64:
	.word	gdt_end - gdt
	.long	0
	.word	0
	.quad   0
gdt:
	.word	gdt_end - gdt
	.long	gdt
	.word	0
	.quad	0x00cf9a000000ffff	/* __KERNEL32_CS */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
gdt_end:
```

이것은 `.data` section에 위치해 있고 5개의 descriptor를 포함하고 있습니다: 첫음부터 kernel code segment를 위한 `32-bit` descriptor, `64-bit` kernel segment, kernel data segment 그리고 2개의 task descriptor입니다.

이전 [part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)에서 이미 `Global Descriptor Table`을 load했었는데, 여기서 거의 같은 일을 다시합니다. 하지만 `64` bit mode에서의 수행을 위해 `CS.L = 1` 그리고 `CS.D = 0`으로 수행합니다. 보시는 바와 같이 `gdt`의 정의는 `gdt` table의 마지막 byte 혹은 table의 크기를 나타내는 `gdt_end - gdt`의 2 bytes로 시작합니다. 다음 4 bytes가 `gdt`의 base address를 가지고 있습니다.

`lgdt` instruction으로 `Global Descriptor Table`을 load한 후, `cr4` register의 값을 `eax`에 읽어 5 bit을 set하고 그것을 다시 `cr4`에 넣어 [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension) mode를 enable해야 합니다:

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

이제 64-bit mode로 이동하기 위한 거의 모든 준비가 끝났습니다. 마지막 step은 page table을 구성하는 것입니다. 하지만 그 전에 long mode에 대해 약간 설명하도로 하겠습니다.

Long mode
--------------------------------------------------------------------------------

[Long mode](https://en.wikipedia.org/wiki/Long_mode)는 [x86_64](https://en.wikipedia.org/wiki/X86-64) processor의 native mode 입니다. 먼저 `x86_64`와 `x86`의 차이점에 대해 살펴봅시다.

`64-bit` mode는 다음과 같은 기능을 제공합니다:

* `r8`에서 `r15`까지의 새로운 8개의 general purpose register + 모든 general purpose register는 64-bit 입니다;
* 64-bit instruction pointer - `RIP`;
* New operating mode - Long mode;
* 64-Bit Addresses와 Operands;
* RIP Relative Addressing (다음 part에서 예를 살펴볼 것입니다).

long mode는 기존의 protected mode의 확장입니다. 이것은 2개의 sub-mode로 구성됩니다:

* 64-bit mode;
* compatibility mode.

`64-bit` mode로 변경하기 위해서는 다음과 같은 것들이 필요합니다:

* [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension)의 enable;
* page table 구성과 top level page table의 address를 `cr3` register에 load;
* `EFER.LME`의 enable;
* paging의 enable.

`PAE`는 `cr4` control register의 `PAE` bit을 설정하여 이미 enable하였습니다. 우리의 다음 goal은 [paging](https://en.wikipedia.org/wiki/Paging)을 위한 structure를 구성하는 것입니다. 다음 문단에서 설명할 것입니다.

Early page table initialization
--------------------------------------------------------------------------------

`64-bit` mode로 이동하기 전에 page table을 구성해야 한다는 것을 이미 알고 있습니다. 선두 `4G` boot page table을 구성하는 것을 살펴봅시다.

**NOTE: 여기서 virtual memory의 이론에 대하여 설명하지는 않을 것입니다. 자세한 내용을 알고 싶다면, 이 part의 마지막에 있는 link를 참조해 주세요.**

linux kernel은 `4-level` paging을 사용하고 일반적으로 6개의 page table을 구성합니다:

* 하나의 `PML4` 혹은 하나의 entry를 가지는 `Page Map Level 4` table;
* 하나의 `PDP` 혹은 4개의 entry를 가지는 `Page Directory Pointer` table;
* `2048` entry를 가지는 4개의 Page Directory table.

이것의 구현을 살펴봅시다. 먼저 memory에서 page table을 위한 buffer를 clear 합니다. 모든 table은 `4096` bytes로 `24` kilobyte의 buffer를 clear해야 합니다:

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$(BOOT_INIT_PGT_SIZE/4), %ecx
	rep	stosl
```

`pgtable`의 address와 `ebx` (`ebx`는 압축 해제를 위한 kernel을 relocate하기 위한 address를 가지고 있다는 것을 기억하십시요) 를 더한 값을 `edi` register에 넣고, `eax` register를 clear하고 `ecx` register를 `6144`로 설정합니다.

`rep stosl` instruction이 `eax`의 값을 `edi`에 쓰고, `edi` register의 값을 `4`만큼 증가시키고 `ecx` register의 값을 `1`만큼 감소시킵니다. 이 동작은 `ecx` register의 값이 0보다 큰동안 반복됩니다. 이것이 `6144` 혹은 `BOOT_INIT_PGT_SIZE/4`를 `ecx`에 넣은 이유입니다.

`pgtable`은 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) assembly file의 마지막에 다음과 같이 정의되어 있습니다:

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill BOOT_PGT_SIZE, 1, 0
```

보시는 바와 같이, 이것은 `.pgtable` section에 위치해 있고 그 크기는 `CONFIG_X86_VERBOSE_BOOTUP` kernel configuration option에 의해 결정됩니다:

```C
#  ifdef CONFIG_X86_VERBOSE_BOOTUP
#   define BOOT_PGT_SIZE	(19*4096)
#  else /* !CONFIG_X86_VERBOSE_BOOTUP */
#   define BOOT_PGT_SIZE	(17*4096)
#  endif
# else /* !CONFIG_RANDOMIZE_BASE */
#  define BOOT_PGT_SIZE		BOOT_INIT_PGT_SIZE
# endif
```

`pgtable` structure를 위한 buffer를 확보한 후, top level page table - `PML4` - 를 다음과 같이 구성할 수 있습니다:

```assembly
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)
```

여기서 다시 `ebx`에서 상대적인 `pgtable`의 address, 다시 말하면 `startup_32`의 상대 address를 `edi` register에 넣습니다. 다음으로 이 address의 `0x1007` offset을 `eax` register에 넣습니다. `0x1007`은 `PML4`의 크기인 `4096` bytes + `7` 입니다. 여기서 `7`은 `PML4` entry를 나타내는 flags 입니다. 이 경우 이 flags는 `PRESENT+RW+USER` 입니다. 마침내 첫번째 `PDP` entry의 address를 `PML4`에 씁니다.

다음 step은 `Page Directory Pointer` table에 4개의 `Page Directory` entry를 동일한 `PRESENT+RW+USER` flags로 구성하는 것입니다:

```assembly
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:  movl	%eax, 0x00(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

`pgtable` table로부터 `4096` 혹은 `0x1000` offset인 page directory pointer의 base address를 `edi`에 넣고 첫번째 page directory pointer entry의 address를 `eax` register에 넣습니다. 다음 loop에서 counter가 될 `4`를 `ecx` register에 넣고 첫번째 page directory pointer table entry의 address를 `edi` register에 넣습니다. 이후 이 `edi`는 flags `0x7`인 첫번째 page directory pointer entry의 address를 가지게 됩니다. 다음으로 각 entry가 `8` bytes인 이어지는 page directory pointer entry의 address를 계산하고 이 address를 `eax`에 넣습니다. paging structure를 구성하는 마지막 step은 `2-MByte` page를 가지는 `2048` page table entry를 구성하는 것입니다:

```assembly
	leal	pgtable + 0x2000(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:  movl	%eax, 0(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

여기서 이전 예와 거의 동일한 것을 하지만, 모든 entry의 flags는 `0x00000183` - `PRESENT + WRITE + MBZ`입니다. 마침내 `2-MByte` page를 가지는 `2048`개의 page - `4G` page table을 구성했습니다:

```python
>>> 2048 * 0x00200000
4294967296
```

이제 memory의 `4` gigabytes를 map하는 early page table structure를 구성했습니다. 이제 high-level page table - `PML4` - 의 address를 `cr3` control register에 넣습니다:

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

이것이 전부 입니다. 모든 준비는 끝났습니다. 이제 long mode로 천이를 살펴봅시다.

Transition to the 64-bit mode
--------------------------------------------------------------------------------

먼저 [MSR](http://en.wikipedia.org/wiki/Model-specific_register)에 `0xC0000080`로 `EFER_LME` flag를 설정해야 합니다:

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

([arch/x86/include/asm/msr-index.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/msr-index.h)에 정의되어 있는) `MSR_EFER` flag를 `ecx` register에 넣고 [MSR](http://en.wikipedia.org/wiki/Model-specific_register) register를 읽는 `rdmsr` instruction을 호출합니다. `rdmsr`이 실행된 후 `ecx` 값에 따라 결과 data는 `edx:eax`에 저장됩니다. `btsl` instruction으로 `EFER_LME` bit을 확인하고 `wrmsr` instruction으로 `eax`의 값을 `MSR` register로 씁니다.

다음 step으로 kernel segment code의 address를 (GDT에서 정의한) stack에 push하고 `startup_64` routine의 address를 `eax`에 넣습니다.

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
```

이후 이 address를 stack에 push하고 `cr0` register의 `PG`와 `PE` bits을 설정하여 paging을 enable 시킵니다:

```assembly
	pushl	%eax
    movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

그리고 다음 `lret` instruction을 수행합니다:

```assembly
lret
```

이전 step에서 `startup_64` 함수의 address를 stack에 push한 것을 기억하십시요. `lret` instruction 후 CPU는 그 address를 가져와서 그곳으로 jump합니다.

이 모든 step후에 마침내 64-bit mode로 진입되었습니다:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```

이것이 전부입니다.!

Conclusion
--------------------------------------------------------------------------------

이것이 linux kernel booting process의 네번째 part의 끝입니다. 질문이나 제안이 있으시면 [0xAX](https://twitter.com/0xAX)나 [email](anotherworldofworld@gmail.com)로 연락을 주시거나 [issue](https://github.com/0xAX/linux-insides/issues/new)를 생성해 주십시요.

다음 part에서 압축 해제와 더 많은 것을 살펴볼 것입니다.

**영어는 저의 모국어가 아닙니다. 불편한 점이 있으셨다면 먼저 사죄드립니다. 만약 잘못된 점을 발견하시면 [linux-insides](https://github.com/0xAX/linux-internals)로 PR을 보내주십시요.**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [GNU linker](http://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf)
* [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
* [Paging](http://en.wikipedia.org/wiki/Paging)
* [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [.fill instruction](http://www.chemie.fu-berlin.de/chemnet/use/info/gas/gas_7.html)
* [Previous part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)
* [Paging on osdev.org](http://wiki.osdev.org/Paging)
* [Paging Systems](https://www.cs.rutgers.edu/~pxk/416/notes/09a-paging.html)
* [x86 Paging Tutorial](http://www.cirosantilli.com/x86-paging/)
