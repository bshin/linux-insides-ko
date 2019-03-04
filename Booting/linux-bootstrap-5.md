Kernel booting process. Part 5.
================================================================================

Kernel decompression
--------------------------------------------------------------------------------

이것은 `Kernel booting process` series의 다섯번째 part입니다. 이전 [part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-4.md#transition-to-the-long-mode)에서 64-bit mode로 천이하는 것을 살펴보았고 이 part에서는 여기서부터 이어 나갈 것입니다. kernel code로 jump하기 전 step인 kernel 압축 해제 준비, relocation과 kernel 압축 해제 등을 살펴 볼 것입니다. 다시 kernel code로 뛰어 들어 봅시다.

Preparation before kernel decompression
--------------------------------------------------------------------------------

`64-bit` entry point - [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) source code file에 위치해 있는 `startup_64`로 jump하기 직전에 멈췄습니다. `startup_32`에서 `startup_64`로 jump하는 것은 이전 part에서 이미 살펴보았습니다:

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
	...
	...
	...
	pushl	%eax
	...
	...
	...
	lret
```

새로운 `Global Descriptor Table`이 load되었고 CPU가 다른 mode (이 경우 `64-bit` mode)로 천이되었기 때문에, `startup_64`의 시작 부분에서 data segments를 setup하는 것을 볼 수 있습니다:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
	xorl	%eax, %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
	movl	%eax, %fs
	movl	%eax, %gs
```

이제 `cs` register를 제외한 모든 segment registers들이 `long mode`로 천이되었을 때와 같이 reset되었습니다.

다음 step은 kernel이 compile된 address와 load된 address의 차이를 계산하는 것입니다:

```assembly
#ifdef CONFIG_RELOCATABLE
	leaq	startup_32(%rip), %rbp
	movl	BP_kernel_alignment(%rsi), %eax
	decl	%eax
	addq	%rax, %rbp
	notq	%rax
	andq	%rax, %rbp
	cmpq	$LOAD_PHYSICAL_ADDR, %rbp
	jge	1f
#endif
	movq	$LOAD_PHYSICAL_ADDR, %rbp
1:
	movl	BP_init_size(%rsi), %ebx
	subl	$_end, %ebx
	addq	%rbp, %rbx
```

`rbp`는 압축 해제된 kernel의 start address를 가지고 있고 이 code가 실행된 후 `rbx` register는 압축 해제를 위해 kernel code가 relocate될 address를 가지게 될것 입니다. 이미 이와 유사한 code를 `startup_32`에서 살펴보았습니다. (이전 part - [Calculate relocation address](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-4.md#calculate-relocation-address)에서 관련 내용을 확인할 수 있습니다) 하지만, bootloader가 64-bit boot protocol을 사용해서 `startup_32`가 실행되지 않았을 수도 있기 때문에 이 계산을 다시 해야 합니다.

다음 step은 stack pointer의 설정입니다. bootloader가 `64-bit` protocol을 사용하여 `32-bit` code segment가 누락되었을 수 있기 때문에 flags register를 reset하고 `GDT`를 다시 설정합니다:

```assembly
    leaq	boot_stack_end(%rbx), %rsp

    leaq	gdt(%rip), %rax
    movq	%rax, gdt64+2(%rip)
    lgdt	gdt64(%rip)

    pushq	$0
    popfq
```

`lgdt gdt64(%rip)` instruction 이후의 linux kernel source code를 살펴보면 추가적인 code가 있는 것을 알 수 있습니다. 이 code는 필요시 trampoline을 생성하고 [5-level pagging](https://lwn.net/Articles/708526/)을 enable합니다. 이 책에서는 오직 4-level paging만을 고려하기 때문에 이 code는 생략하겠습니다.

위에서 보시는 것과 같이 `rbx` register는 kernel 압축 해제를 위한 code의 시작 address를 가지고 있고 이 address의 `boot_stack_end` offset의 값을 stack의 top을 가르키는 pointer로서 `rsp` register에 넣습니다. 이 step 후에 stack은 적절하게 설정된 것입니다. `boot_stack_end`는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) assembly source code file의 마지막 부분에 정의되어 있습니다:

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

이것은 `.bss` section의 마지막 부분에  `.pgtable` 직전에 위치해 있습니다. [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/vmlinux.lds.S) linker script를 보시면 `.bss`와 `.pgtable`의 정의를 찾아볼 수 있습니다.

stack을 설정하였으니, 이제 압축된 kernel을 위에서 계산한 압축 해제된 kernel의 relocation address로 복사할 수 있습니다. 자세한 내용을 다루기 전에 하기의 assembly code를 살펴봅시다:

```assembly
	pushq	%rsi
	leaq	(_bss-8)(%rip), %rsi
	leaq	(_bss-8)(%rbx), %rdi
	movq	$_bss, %rcx
	shrq	$3, %rcx
	std
	rep	movsq
	cld
	popq	%rsi
```

먼저 `rsi`를 stack에 push합니다. `rsi` register는 booting 관련 data를 가지는 real mode structure인 `boot_params` (기억하시겠지만, kernel setup의 시작 부분에서 이 structure에 값을 채웠습니다)의 pointer를 저장할 것이기 때문에 이 register의 값을 보존해야 합니다. 이 code의 끝에 `boot_params`의 pointer를 가진 `rsi`의 값을 다시 복구할겁니다.

다음 두개의 `leaq` instructions은 `_bss - 8` offset으로 `rip`와 `rbx`의 유효한 address를 계산하고 이것을 `rsi`와 `rdi`에 넣습니다. 왜 이 address를 계산해야 할까요? 실제 압축된 kernel image는 이 복사하는 code (`startup_32`부터 현재 code까지)와 압축을 해제하는 code 사이에 위치합니다. linker script - [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/vmlinux.lds.S)에서 확인해 볼 수 있습니다:

```
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
	.rodata..compressed : {
		*(.rodata..compressed)
	}
	.text :	{
		_text = .; 	/* Text */
		*(.text)
		*(.text.*)
		_etext = . ;
	}
```

`.head.text` section이 `startup_32`를 포함한다는 것에 주목하십시요. 이전 part에서 하기의 code를 기억하실 겁니다:

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
...
...
...
```

`.text` section은 압축 해제 code를 포함합니다:

```assembly
	.text
relocated:
...
...
...
/*
 * Do the decompression, and jump to the new kernel..
 */
...
```

그리고 `.rodata..compressed`는 압축된 kernel image를 포함합니다. `rsi`는 `_bss - 8`의 절대 address를 가지고 있고 `rdi`는 `_bss - 8`의 relocation 상대 address를 가지고 있습니다. 이러한 address를 register에 저장할때 `rcx` register에 `_bss`의 address를 저장했습니다. `vmlinux.lds.S` linker script에서 보시는 것처럼 이것은 setup/kernel code와 모든 section의 뒤에 위치합니다. 이제 `rsi`에서 `rdi`로 `movsq` instruction을 사용하여 한번에 8 bytes씩 data 복사를 시작할 수 있습니다.

data를 복사하기 전 `std` instruction이 있다는 것에 주목하십시요: 이것은 `rsi`와 `rdi`가 감소한다는 것을 의미하는 `DF` flag를 설정합니다. 즉, 복사는 뒤에서 부터 진행됩니다. 마지막으로 `cld` instruction을 사용하여 `DF` flag를 clear하고 `boot_params` structure를 `rsi`로 복구합니다.

이제 relocation후 `.text` section의 address를 확인했고, 그곳으로 jump합니다:

```assembly
	leaq	relocated(%rbx), %rax
	jmp	*%rax
```

Last preparation before kernel decompression
--------------------------------------------------------------------------------

이전 문단에서 `relocated` label로 시작되는 `.text` section을 보았습니다. 그것이 하는 첫번째 일은 `bss` section을 clear하는 것입니다:

```assembly
	xorl	%eax, %eax
	leaq    _bss(%rip), %rdi
	leaq    _ebss(%rip), %rcx
	subq	%rdi, %rcx
	shrq	$3, %rcx
	rep	stosq
```

이제 곧 [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) code로 jump할 것이기 때문에 `.bss` section을 초기화해야 합니다. 여기서 `eax`를 clear하고 `_bss`의 address를 `rdi`에 넣고 `_ebss`의 address를 `rcx`에 넣고 `rep stosq` instruction을 사용하여 0으로 채웁니다.

마침내 `extract_kernel` 함수를 호출하는 것을 볼 수 있습니다:

```assembly
	pushq	%rsi
	movq	%rsi, %rdi
	leaq	boot_heap(%rip), %rsi
	leaq	input_data(%rip), %rdx
	movl	$z_input_len, %ecx
	movq	%rbp, %r8
	movq	$z_output_len, %r9
	call	extract_kernel
	popq	%rsi
```

다시 `rdi`에 `boot_param` structure를 가르키는 pointer를 넣고 그것을 stack에 보존합니다. 동시에 `rsi`에 kernel 압축 해제에 사용될 영역의 pointer를 넣습니다. 마지막 step은 `extract_kernel`의 parameters를 준비하고 kernel의 압축을 해제하는 이 함수를 호출하는 것입니다. `extract_kernel` 함수는 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c) source code file에 정의되어 있고 6개의 arguments를 받습니다:

* `rmode` - bootloader나 early kernel initialization에서 채워지는 [boot_params](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h) structure를 가르키는 pointer;
* `heap` - early boot heap의 시작 address를 나타내는 `boot_heap`을 가르키는 pointer;
* `input_data` - 압축된 kernel의 시작을 가르키는 pointer 즉 `arch/x86/boot/compressed/vmlinux.bin.bz2`을 가르키는 pointer;
* `input_len` - 압축된 kernel의 size;
* `output` - 압으로 압축 해제된 kernel의 시작 address;
* `output_len` - 압축 해제된 kernel의 size;

[System V Application Binary Interface](http://www.x86-64.org/documentation/abi.pdf)에 따라 모든 arguments는 register로 전달됩니다. 모든 준비가 끝났고 이제 kernel 압축 해제를 살펴볼 수 있습니다.

Kernel decompression
--------------------------------------------------------------------------------

이전 문단에서 살펴본 것처럼, `extract_kernel` 함수는 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c) source code file에 정의되어 있고 6개의 arguments를 받습니다. 이 함수는 이전 parts에서 이미 살펴본 video/console 초기화로 시작합니다. 이것은 [real mode](https://en.wikipedia.org/wiki/Real_mode)에서 시작되었는지, bootloader가 사용되었는지, bootloader가 `32` 혹은 `64-bit` boot protocol을 사용했는지 알 수 없기 때문에 필요합니다.

첫번째 초기화 step 이후, free memory의 처음과 마지막의 pointers를 저장합니다:

```C
free_mem_ptr     = heap;
free_mem_end_ptr = heap + BOOT_HEAP_SIZE;
```

여기서 `heap`은  `extract_kernel` 함수의 두번째 parameter이고 이것은 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)에서 얻었습니다:

```assembly
leaq	boot_heap(%rip), %rsi
```

위에서 보는 것처럼 `boot_heap`이 정의됩니다:

```assembly
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
```

여기서 `BOOT_HEAP_SIZE`는 `0x10000` (`bzip2` kernel이 경우 `0x400000`)로 확장되는 macro이며 heap의 size를 나타냅니다.

heap pointers가 초기화된 후, 다음 step은 [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr.c) source code file의 `choose_random_location` 함수를 호출하는 것입니다. 함수 이름에서 추측할 수 있는 것처럼, 이것은 압축 해제된 kernel image의 memory location을 선택합니다. 압축된 kernel image를 압축 해제하는 찾는다거나 심지어 `choose`해야 한다는 것이 이상해 보일 수 있습니다만, linux kernel은 security 이유로 random한 address에 kernel의 압축 해제를 허용하는 [kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization)을 지원하기 때문입니다.

이 part에서는 linux kernel load address의 randomization을 고려하지는 않을 것입니다만, 다음 part에서 살펴볼 것입니다.

이제 [misc.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c)로 돌아가 봅시다. kernel image를 위한 address를 얻은 후, 검색한 random address가 제대로 align되어 있는지와 address가 잘못되지 않았는지 확인이 필요합니다:

```C
if ((unsigned long)output & (MIN_KERNEL_ALIGN - 1))
	error("Destination physical address inappropriately aligned");

if (virt_addr & (MIN_KERNEL_ALIGN - 1))
	error("Destination virtual address inappropriately aligned");

if (heap > 0x3fffffffffffUL)
	error("Destination address too large");

if (virt_addr + max(output_len, kernel_total_size) > KERNEL_IMAGE_SIZE)
	error("Destination virtual address is beyond the kernel mapping area");

if ((unsigned long)output != LOAD_PHYSICAL_ADDR)
    error("Destination address does not match LOAD_PHYSICAL_ADDR");

if (virt_addr != LOAD_PHYSICAL_ADDR)
	error("Destination virtual address changed when not relocatable");
```

이 모든 확인 후 익숙한 message를 볼 수 있습니다:

```
Decompressing Linux... 
```

그리고 kernel의 압축을 해제하는 `++decompress` 함수를 호출합니다:

```C
__decompress(input_data, input_len, NULL, NULL, output, output_len, NULL, error);
```

`__decompress` 함수의 구현은 kernel compilation시 어떤 압축 해제 algorithm이 선택되었는지에 따라 달라집니다:

```C
#ifdef CONFIG_KERNEL_GZIP
#include "../../../../lib/decompress_inflate.c"
#endif

#ifdef CONFIG_KERNEL_BZIP2
#include "../../../../lib/decompress_bunzip2.c"
#endif

#ifdef CONFIG_KERNEL_LZMA
#include "../../../../lib/decompress_unlzma.c"
#endif

#ifdef CONFIG_KERNEL_XZ
#include "../../../../lib/decompress_unxz.c"
#endif

#ifdef CONFIG_KERNEL_LZO
#include "../../../../lib/decompress_unlzo.c"
#endif

#ifdef CONFIG_KERNEL_LZ4
#include "../../../../lib/decompress_unlz4.c"
#endif
```

kernel이 압축 해제된 후, 마지막 두 함수는 `parse_elf`와 `handle_relocations`입니다. 이 두 함수의 주 목적은 압축 해제된 kernel image를 적절한 memory 위치로 옮기는 것입니다. 사실 압축 해제는 [in-place](https://en.wikipedia.org/wiki/In-place_algorithm)에서 이루어지기 때문에 kernel을 적절한 address로 옮겨야 합니다. 이미 알고 있듯이 kernel image는 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) executable로 `parse_elf` 함수는 loadable segments를 적절한 address로 옮깁니다. loadable segments는 `readelf` program의 출력에서 볼 수 있습니다:

```
readelf -l vmlinux

Elf file type is EXEC (Executable file)
Entry point 0x1000000
There are 5 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x0000000000893000 0x0000000000893000  R E    200000
  LOAD           0x0000000000a93000 0xffffffff81893000 0x0000000001893000
                 0x000000000016d000 0x000000000016d000  RW     200000
  LOAD           0x0000000000c00000 0x0000000000000000 0x0000000001a00000
                 0x00000000000152d8 0x00000000000152d8  RW     200000
  LOAD           0x0000000000c16000 0xffffffff81a16000 0x0000000001a16000
                 0x0000000000138000 0x000000000029b000  RWE    200000
```

`parse_elf` 함수의 목적은 이 segments를 `choose_random_location` function의 `output` address로 load하는 것입니다. 이 함수는 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) signature를 확인하는 것으로 시작합니다:

```C
Elf64_Ehdr ehdr;
Elf64_Phdr *phdrs, *phdr;

memcpy(&ehdr, output, sizeof(ehdr));

if (ehdr.e_ident[EI_MAG0] != ELFMAG0 ||
    ehdr.e_ident[EI_MAG1] != ELFMAG1 ||
    ehdr.e_ident[EI_MAG2] != ELFMAG2 ||
    ehdr.e_ident[EI_MAG3] != ELFMAG3) {
        error("Kernel is not a valid ELF file");
        return;
}
```

만약 signature가 valid하지 않다면, error message를 출력하고 halt합니다. 만약 valid한 `ELF` file이라면, `ELF` file의 모든 program headers를 지나 2 megabytes에 align된 모든 loadable segments를 output buffer로 copy합니다:

```C
	for (i = 0; i < ehdr.e_phnum; i++) {
		phdr = &phdrs[i];

		switch (phdr->p_type) {
		case PT_LOAD:
#ifdef CONFIG_X86_64
			if ((phdr->p_align % 0x200000) != 0)
				error("Alignment of LOAD segment isn't multiple of 2MB");
#endif                
#ifdef CONFIG_RELOCATABLE
			dest = output;
			dest += (phdr->p_paddr - LOAD_PHYSICAL_ADDR);
#else
			dest = (void *)(phdr->p_paddr);
#endif
			memmove(dest, output + phdr->p_offset, phdr->p_filesz);
			break;
		default:
			break;
		}
	}
```

이것이 전부입니다.

이 순간부터 모든 loadable segments는 적절한 위치에 있습니다.

`parse_elf` 함수 이후 다음 step은 `handle_relocations` 함수를 호출하는 것입니다. 이 함수의 구현은 `CONFIG_X86_NEED_RELOCS` kernel configuration option에 따라 달라집니다. 만약 이것이 enable되어 있으면 이 함수는 kernel image에서 address를 조정하는데, 이것은 kernel configuration에서 `CONFIG_RANDOMIZE_BASE` configuration option이 enable되어 있을 때만 호출됩니다. `handle_relocations` 함수의 구현은 굉장히 간단합니다. 이 함수는 kernel의 base load address의 값에서 `LOAD_PHYSICAL_ADDR`의 값을 빼서 kernel이 link될 때의 address와 실제 load된 address의 차를 얻습니다. 이후 kernel이 load된 실제 address와 수행되도록 link된 address와 kernel image의 마지막에 있는 relocation table을 알고 있기 때문에 kernel relocation을 수행할 수 있습니다.

kernel을 relocate한 후 `extract_kernel`에서 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)로 돌아갑니다.

kernel의 address는 `rax` register에 있고 그곳으로 jump합니다:

```assembly
jmp	*%rax
```

이것이 전부입니다. 이제 kernel에 진입하였습니다!

Conclusion
--------------------------------------------------------------------------------

이것이 linux kernel booting process의 다섯번째 part의 끝입니다. 이제 더이상 kernel booting에 대한 post는 없을 것입니다. (이전 post에 대한 update는 있을 수 있습니다) 하지만 다른 kernel internals에 대한 많은 post가 있을 것입니다.

다음 chapter에서는 load address randomization 같은 linux kernel booting process의 고급 세부 사항을 기술할 것입니다.

질문이나 제안이 있으시면 comment를 남기시거나 [twitter](https://twitter.com/0xAX)로 알려 주십시요.

**영어는 저의 모국어가 아닙니다. 불편한 점이 있으셨다면 먼저 사죄드립니다. 만약 잘못된 점을 발견하시면 [linux-insides](https://github.com/0xAX/linux-internals)로 PR을 보내주십시요.**

Links
--------------------------------------------------------------------------------

* [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [long mode](https://en.wikipedia.org/wiki/Long_mode)
* [bzip2](http://www.bzip.org/)
* [RdRand instruction](https://en.wikipedia.org/wiki/RdRand)
* [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [Programmable Interval Timers](https://en.wikipedia.org/wiki/Intel_8253)
* [Previous part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-4.md)
