Kernel initialization. Part 1.
================================================================================

First steps in the kernel code
--------------------------------------------------------------------------------

이전 [post](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-6.html)는 linux kernel [booting process](https://0xax.gitbooks.io/linux-insides/content/Booting/index.html) chapter의 마지막 part였습니다. 이제 linux kernel의 initialization process로 뛰어 들어볼 차례입니다. linux kernel image가 압축 해제되고 memory의 적절한 곳에 위치된 후 동작하기 시작합니다. 이전 모든 parts는 linux kernel code의 첫번째 bytes가 실행되기 전 준비하는 linux kernel setup code의 동작을 설명하였습니다. 이제 kernel로 진입하였고 이 chapter의 모든 part는 kernel이 [pid](https://en.wikipedia.org/wiki/Process_identifier) `1`로 process를 실행시키기 전 kernel의 초기화 과정을 설명할 것입니다. kernel이 첫번째 `init` process를 시작하기 전 많은 것들을 해야 합니다. 이 큰 chapter에서 kernel이 시작하기 전 모든 준비 과정을 살펴보기를 희망합니다. [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)에 있는 kernel entry point에서 시작하여 하나씩 진행할 것입니다. [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)의 `start_kernel`이 호출되는 것일 보기 전, early page tables 초기화, kernel space에서 새로운 descriptor로 전환 등 첫번째 준비 작업을 살펴 볼 것입니다.

이전 [chapter](https://0xax.gitbooks.io/linux-insides/content/Booting/index.html)의 마지막 [part](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-6.html)에서 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) assembly source code file의 jmp instruction에서 멈추었습니다:

```assembly
jmp	*%rax
```

이 시점에서 `rax` register는 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c) source code file의 `decompress_kernel` 함수를 호출하여 얻은 linux kernel entry point의 address를 가지고 있습니다. kernel setup code의 마지막 instruction은 kernel entry point로 jump하는 것입니다. linux kernel의 entry point가 define되어 있는 곳을 알고 있기 때문에 시작 후 linux kernel이 무엇을 하는지 살펴볼 수 있습니다.

First steps in the kernel
--------------------------------------------------------------------------------

자, `decompress_kernel` 함수에서 압축 해제된 kernel image의 주소를 `rax` register에 얻었고 그곳으로 jump했습니다. 이미 아시는 것처럼 압축 해제된 kernel image의 entry point는 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) assembly source code file에서 시작하고, 그것의 시작 부분에서 다음과 같은 정의를 볼 수 있습니다:

```assembly
    .text
	__HEAD
	.code64
	.globl startup_64
startup_64:
	...
	...
	...
```

`__HEAD` section에 정의되어 있는 `startup_64` routine의 정의를 볼 수 있는데, `__HEAD`는 executable `.head.text` section으로 확장되는 macro입니다:

```C
#define __HEAD		.section	".head.text","ax"
```

이 section의 정의는 [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S) linker script에서 찾아볼 수 있습니다:

```
.text : AT(ADDR(.text) - LOAD_OFFSET) {
	_text = .;
	...
	...
	...
} :text = 0x9090
```

`.text` section의 정의 외에도 linker script로부터 default virtual과 physical address를 알 수 있습니다. `_text`의 address는 [x86_64](https://en.wikipedia.org/wiki/X86-64)에서는 다음과 같이 location counter로 정의되어 있다는 것에 주목하십시요:

```
. = __START_KERNEL;
```

`__START_KERNEL` macro는 [arch/x86/include/asm/page_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_types.h) header file에 정의되어 있고 kernel mapping의 virtual address와 physical start의 합을 나타냅니다:

```C
#define __START_KERNEL	(__START_KERNEL_map + __PHYSICAL_START)

#define __PHYSICAL_START  ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)
```

즉:

* linux kernel의 base physical address - `0x1000000`;
* linux kernel의 base virtual address - `0xffffffff81000000`.

CPU configuration을 제거 후, [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)에 정의된 `__startup_64` 함수를 호출합니다:

```assembly
	leaq	_text(%rip), %rdi
	pushq	%rsi
	call	__startup_64
	popq	%rsi
```

```C
unsigned log __head __startup_64(unsigned long physaddr,
				 struct boot_params *bp)
{
	unsigned long load_delta, *p;
	unsigned long pgtable_flags;
	pgdval_t *pgd;
	p4dval_t *p4d;
	pudval_t *pud;
	pmdval_t *pmd, pmd_entry;
	pteval_t *mask_ptr;
	bool la57;
	int i;
	unsigned int *next_pgt_ptr;
	...
	...
	...
}
```

[kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization#Linux)이 enable되어 있기 때문에, `startup_64` routine은 compile된 address와 다른 곳에 load되어 있습니다. 그래서 다음 code로 delta를 계산해야 합니다:

```C
	load_delta = physaddr - (unsigned long)(_text - __START_KERNEL_map);
```

결과적으로, `load_delta`는 compile된 address와 실제 load된 address의 delta를 가집니다.

delta를 얻은 후 `_text` address가 2 megabytes에 align되어 있는지 다음의 code로 확인합니다:

```C
	if (load_delta & ~PMD_PAGE_MASK)
		for (;;);
```

만약 `_text` address가 `2` megabytes에 align되어 있지 않으면, 무한 loop로 진입합니다. `PMD_PAGE_MASK`는 `Page middle directory`([Paging](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html) 참조)를 위한 mask를 나타내고 하기와 깉이 정의되어 있습니다:

```C
#define PMD_PAGE_MASK           (~(PMD_PAGE_SIZE-1))
```

여기서 `PMD_PAGE_SIZE` macro는 하기와 같이 정의되어 있습니다:

```C
#define PMD_PAGE_SIZE           (_AC(1, UL) << PMD_SHIFT)
#define PMD_SHIFT		21
```

쉽게 계산할 수 있는 것처럼 `PMD_PAGE_SIZE`는 `2` megabytes입니다.

만약 [SME](https://en.wikipedia.org/wiki/Zen_%28microarchitecture%29#Enhanced_security_and_virtualization_support)가 지원되고 enable되어 있다면, 이것을 activate시키고 `load_delta`에 SME encryption mask를 포함시킵니다:

```C
	sme_enable(bp);
	load_delta += sme_get_me_mask();
```

자, 이제 early check를 했고 다음으로 진행할 수 있습니다.

Fix base addresses of page tables
--------------------------------------------------------------------------------

다음 step으로 page table의 physical address를 수정합니다:

```C
	pgd = fixup_pointer(&early_top_pgt, physaddr);
	pud = fixup_pointer(&level3_kernel_pgt, physaddr);
	pmd = fixup_pointer(level2_fixmap_pgt, physaddr);
```

자, 전달된 argument의 physical address를 return하는 `fixup_pointer` 함수의 정의를 살펴봅시다:

```C
static void __head *fixup_pointer(void *ptr, unsigned long physaddr)
{
	return ptr - (void *)_text + (void *)physaddr;
}
```

다음으로 `early_top_pgt`와 위에서 본 다른 page table symbols에 집중할 것입니다. 이 symbols들이 어떤 의미인지 이해해 봅시다. 먼저 정의를 살펴봅시다:

```assembly
NEXT_PAGE(early_top_pgt)
	.fill	512,8,0
	.fill	PTI_USER_PGD_FILL,8,0

NEXT_PAGE(level3_kernel_pgt)
	.fill	L3_START_KERNEL,8,0
	.quad	level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE_NOENC
	.quad	level2_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC

NEXT_PAGE(level2_kernel_pgt)
	PMDS(0, __PAGE_KERNEL_LARGE_EXEC,
		KERNEL_IMAGE_SIZE/PMD_SIZE)

NEXT_PAGE(level2_fixmap_pgt)
	.fill	506,8,0
	.quad	level1_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC
	.fill	5,8,0

NEXT_PAGE(level1_fixmap_pgt)
	.fill	512,8,0
```

어려워 보입니다만 그렇지 않습니다. 먼저 `early_top_pgt`를 살펴봅시다. 이것은 `4906` bytes의 0으로 시작하고, (만약 `CONFIG_PAGE_TABLE_ISOLATION`이 enable되어 있으면 `8192` bytes) 이것은 처음 `512` entries를 사용하지 않는다는 것을 의미합니다. 이후 `level3_kernel_pgt` entry를 볼 수 있습니다. 이 정의의 시작에 `4080` bytes(`L3_START_KERNEL`이 `510`과 같음)를 0으로 채우는 것을 볼 수 있습니다. 그 뒤에, kernel space를 map하는 2개의 entries를 저장합니다. `level2_kernel_pgt`와 `level2_fixmap_pgt`에서 `__START_KERNEL_map`을 빼는 것을 주목하십시요. 알고 있는 것처럼 `__START_KERNEL_map`은 kernel text의 base virtual address이고, `__START_KERNEL_map`을 뺀다는 것은 `level2_kernel_pgt`와 `level2_fixmap_pgt`의 physical address를 얻는다는 것입니다.

다음으로 `_KERNPG_TABLE_NOENC`와 `_PAGE_TABLE_NOENC`를 살펴봅시다. 이것들은 page entry access 권한입니다:

```C
#define _KERNPG_TABLE_NOENC   (_PAGE_PRESENT | _PAGE_RW | _PAGE_ACCESSED | \
			       _PAGE_DIRTY)
#define _PAGE_TABLE_NOENC     (_PAGE_PRESENT | _PAGE_RW | _PAGE_USER | \
			       _PAGE_ACCESSED | _PAGE_DIRTY)
```

`level2_kernel_pgt`는 kernel space를 map하는 page middle directory의 pointer를 가지는 page table entry입니다. 이것은 kernel `.text`를 위해 `__START_KERNEL_map`에서 `512` megabytes를 생성하는 `PDMS` macro를 호출합니다. (이 `512` megabytes이후는 module memory space가 될겁니다)

`level2_fixmap_pgt`는 kernel space도 포함하여 모든 physical address를 access할 수 있는 virtual address입니다. 이것은 `4048` bytes의 0, `level1_fixmap_pgt` entry, [vsyscalls](https://lwn.net/Articles/446528/) mapping를 위한 `8` megabytes의 reserve, `2` megabytes의 hole로 나타납니다.

이에 대해서는 [Paging](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html) part에서 자세히 볼 수 있습니다.

이제, 이 symbols에 대해서 살펴보았고, code로 돌아가 봅시다. 다음으로 `pgd`의 마지막 entry를 `level3_kernel_pgt`로 초기화합니다:

```C
	pgd[pgd_index(__START_KERNEL_map)] = level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC;
```

만약 `startup_64`가 default인 `0x1000000` address가 이나리면 모든 `p*d` address는 잘못된 것일 수 있습니다. `load_delta`가 `startup_64` symbol의 kernel [linking](https://en.wikipedia.org/wiki/Linker_%28computing%29)시와 실제 address의 delta를 가지고 있음을 기억하십시요. 그래서 `p*d`에 이 delta를 더합니다.

```C
	pgd[pgd_index(__START_KERNEL_map)] += load_delta;
	pud[510] += load_delta;
	pud[511] += load_delta;
	pmd[506] += load_delta;
```

이 모든 작업 후, 하기와 같은 값을 가지게 됩니다:

```
early_top_pgt[511] -> level3_kernel_pgt[0]
level3_kernel_pgt[510] -> level2_kernel_pgt[0]
level3_kernel_pgt[511] -> level2_fixmap_pgt[0]
level2_kernel_pgt[0]   -> 512 MB kernel mapping
level2_fixmap_pgt[506] -> level1_fixmap_pgt
```

`early_top_pgt`와 다른 page table directory의 base address를 수정하지 않았다는 것에 주목하십시요. 이것은 이 page table의 structure를 build/fill할때 볼 것이기 때문입니다. page table의 base address를 맞게 수정하였으니 build를 시작할 수 있습니다.

Identity mapping setup
--------------------------------------------------------------------------------

이제 early page tables의 identity mapping 설정을 살펴볼 수 있습니다. Identity Mapping Paging에서 virtual address는 physical address와 동일하게 map됩니다. 자세하게 살펴 봅시다. 먼저 `pud`와 `pmd`를 `early_dynamic_pgts`의 첫번째와 두번째 entry로 교체합니다:

```C
	next_pgt_ptr = fixup_pointer(&next_early_pgt, physaddr);
	pud = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);
	pmd = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);
```

`early_dynamic_pgts` 정의를 살펴봅시다:

```assembly
NEXT_PAGE(early_dynamic_pgts)
	.fill	512*EARLY_DYNAMIC_PAGE_TABLES,8,0
```

이것은 early kernel을 위한 임시 page tables을 저장합니다.

다음으로 후에 `p*d` entries를 초기화할때 사용되는 `pgtable_flags`를 초기화합니다:

```C
	pgtable_flags = _KERNPG_TABLE_NOENC + sme_get_me_mask();
```

`sme_get_me_mask` 함수는 `sme_enable` 함수에서 초기화되는 `sme_me_mask`를 return 합니다.

다음으로 `pgd`의 두개의 entries를 `pud` + 위에서 초기화한 `pgtable_flags`로 채웁니다:

```C
	i = (physaddr >> PGDIR_SHIFT) % PTRS_PER_PGD;
	pgd[i + 0] = (pgdval_t)pud + pgtable_flags;
	pgd[i + 1] = (pgdval_t)pud + pgtable_flags;
```
`PGDIR_SHFT`는 virtual address에서 page global directory bits의 mask를 나타냅니다. 여기서 (`512`로 확장되는) `PTRS_PER_PGD`로 modulo연산을 하여 index가 `512`보다 크지 않게 합니다. 모든 type의 page directories를 위한 macro가 있습니다:

```C
#define PGDIR_SHIFT     39
#define PTRS_PER_PGD	512
#define PUD_SHIFT       30
#define PTRS_PER_PUD	512
#define PMD_SHIFT       21
#define PTRS_PER_PMD	512
```

위와 거의 동일한 것을 수행합니다:

```C
	i = (physaddr >> PUD_SHIFT) % PTRS_PER_PUD;
	pud[i + 0] = (pudval_t)pmd + pgtable_flags;
	pud[i + 1] = (pudval_t)pmd + pgtable_flags;
```

다음으로 `pmd_entry`를 초기화하고 지원되지 않는 `__PAGE_KERNEL_*` bit을 clear합니다:

```C
	pmd_entry = __PAGE_KERNEL_LARGE_EXEC & ~_PAGE_GLOBAL;
	mask_ptr = fixup_pointer(&__supported_pte_mask, physaddr);
	pmd_entry &= *mask_ptr;
	pmd_entry += sme_get_me_mask();
	pmd_entry += physaddr;
```

다음으로 kernel의 전체 size를 cover하기 위해 모든 `pmd` entries를 채웁니다:

```C
	for (i = 0; i < DIV_ROUND_UP(_end - _text, PMD_SIZE); i++) {
		int idx = i + (physaddr >> PMD_SHIFT) % PTRS_PER_PMD;
		pmd[idx] = pmd_entry + i * PMD_SIZE;
	}
```

다음으로 text+data virtual address를 수정합니다. kernel이 relocate되어 있다면 invalid한 pmds를 쓸수 있다는 것을 주목하십시요. (`_end`후의 mapping은 `cleanup_highmap` 함수에서 수정합니다)

```C
	pmd = fixup_pointer(level2_kernel_pgt, physaddr);
	for (i = 0; i < PTRS_PER_PMD; i++) {
		if (pmd[i] & _PAGE_PRESENT)
			pmd[i] += load_delta;
	}
```

다음으로 실제 physical address를 얻기 위해 memory encryption mask를 제거합니다 (`load_delta`가 mask를 포함하고 있다는 것을 기억하십시요):

```C
	*fixup_long(&phys_base, physaddr) += load_delta - sme_get_me_mask();
```

`phys_base` must match the first entry in `level2_kernel_pgt`.

`__startup_64` 함수의 마지막 step으로 (만약 SME가 active라면) kernel을 encrypt하고 `cr3` register에 program될 초기 page directory entry를 위한 modifier로 사용될 SME encryption mask를 return합니다:

```C
	sme_encrypt_kernel(bp);
	return sme_get_me_mask();
```

이제 assembly code로 돌아가 봅시다. 다음 문단을 위해 다음 code로 준비를 합니다:

```assembly
	addq	$(early_top_pgt - __START_KERNEL_map), %rax
	jmp 1f
```

`early_top_pgt`의 physical address에 `rax` register를 더하여, `rax` register가 address와 SME encryption mask를 가지게 합니다.

이것이 현재까지 전부입니다. early paging은 준비되었고 kernel entry point로 jump하기전 마지막 분비를 끝마치면 됩니다.

Last preparation before jump at the kernel entry point
--------------------------------------------------------------------------------

label `1`로 jump한 후, `PAE`와 `PGE` (Paging Global Extension)을 enable하고 `phys_base` (위 참조)위 내용을 `rax` register에 넣고 그것으로 `cr3` register를 채웁니다:

```assembly
1:
	movl	$(X86_CR4_PAE | X86_CR4_PGE), %ecx
	movq	%rcx, %cr4

	addq	phys_base(%rip), %rax
	movq	%rax, %cr3
```

다음으로 다음 code로 CPU가 [NX](http://en.wikipedia.org/wiki/NX_bit) bit을 지원하는지 확인합니다:

```assembly
	movl	$0x80000001, %eax
	cpuid
	movl	%edx,%edi
```

`0x80000001` 값을 `eax`에 넣고 extended processor info와 feature bits을 얻기 위해 `cpuid` instruction을 수행합니다. 결과는 `edx` register에 저장되고 이것을 `edi`에 넣습니다.

이제 `0xc0000080` 혹은 `MSR_EFER`를 `ecx`에 넣고 model specific register를 읽기 이해 `rdmsr` instruction을 수행합니다.

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
```

결과는 `edx:eax`에 저장됩니다. `EFER`의 일반적인 모습은 다음과 같습니다:

```
63                                                                              32
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved MBZ                                   |
|                                                                               |
 --------------------------------------------------------------------------------
31                            16  15      14      13   12  11   10  9  8 7  1   0
 --------------------------------------------------------------------------------
|                              | T |       |       |    |   |   |   |   |   |   |
| Reserved MBZ                 | C | FFXSR | LMSLE |SVME|NXE|LMA|MBZ|LME|RAZ|SCE|
|                              | E |       |       |    |   |   |   |   |   |   |
 --------------------------------------------------------------------------------
```

여기서 모든 fields를 자세히 살펴보지는 않겠지만, 특별한 part에서 이것과 다른 `MSRs`에 대해서 배울 것입니다. `EFER`을 `edx:eax`로 읽었으니 `btsl` instruction을 사용하는 `System Call Extensions`을 나타내는 `_EFER_SCE`를 확인하고 이를 1로 설정합니다. `SCE`를 설정함으로써 `SYSCALL`과 `SYSRET` instructions이 enable됩니다. 다음 step으로 `cpuid`의 결과를 저장하고 있는 `edi` register (위 참조)의 20번째 bit을 확인합니다. `20` bit (`NX` bit)이 set되어 있다면 `EFER_SCE`를 model specific register에 써줍니다.

```assembly
	btsl	$_EFER_SCE, %eax
	btl	$20,%edi
	jnc     1f
	btsl	$_EFER_NX, %eax
	btsq	$_PAGE_BIT_NX,early_pmd_flags(%rip)
1:	wrmsr
```

만약 [NX](https://en.wikipedia.org/wiki/NX_bit) bit이 지원되면 `_EFER_NX`를 enable하고 `wrmsr` instruction으로 이를 write합니다. [NX](https://en.wikipedia.org/wiki/NX_bit) bit이 set된 이후, 다음 assembly code로 `cr0` [control register](https://en.wikipedia.org/wiki/Control_register)에 일부 bits을 set합니다:

```assembly
	movl	$CR0_STATE, %eax
	movq	%rax, %cr0
```

특히 다음 bits를 set합니다:

* `X86_CR0_PE` - system이 protected mode에 있습니다;
* `X86_CR0_MP` - WAIT/FWAIT instructions와 CR0의 TS flag 사이의 동작을 controls 합니다;
* `X86_CR0_ET` - 386에서 external math coprocessor가 80287인지 80387인지 기술합니다;
* `X86_CR0_NE` - set되면 internal x87 floating point error reporting을 enable하고, 그렇지 않으면 PC style x87 error detection을 enable합니다;
* `X86_CR0_WP` - set되면, privilege level이 0일때 CPU는 read-only pages에 write할 수 없습니다;
* `X86_CR0_AM` - AM이 set되면 alignment check가 enable되고, (EFLAGS register의) AC flag가 set되면, privilege level이 3이 됩니다;
* `X86_CR0_PG` - paging을 enable합니다.

assembly에서 [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) code를 수행시키기 위해서는 stack을 설정해야 합니다. 이것은 [stack pointer](https://en.wikipedia.org/wiki/Stack_register)에 memory의 적절한 장소를 설정하는 것으로 이루어지고 이후 [flags](https://en.wikipedia.org/wiki/FLAGS_register) register를 다시 설정합니다:

```assembly
	movq initial_stack(%rip), %rsp
	pushq $0
	popfq
```

여기서 가장 흥미로운 것은 `initial_stack`입니다. 이 symbol은 [source](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) code file에 정의되어 있고 다음과 같습니다:

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union + THREAD_SIZE - SIZEOF_PTREGS
```

`THREAD_SIZE` macro는 [arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_64_types.h) header file에 정의되어 있고 `KASAN_STACK_ORDER` macro의 값에 따라 바뀝니다:

```C
#ifdef CONFIG_KASAN
#define KASAN_STACK_ORDER 1
#else
#define KASAN_STACK_ORDER 0
#endif

#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

[kasan](https://github.com/torvalds/linux/blob/master/Documentation/dev-tools/kasan.rst)이 disable되어 있고 `PAGE_SIZE`가 `4096` bytes인 경우를 고려해 봅시다. `THREAD_SIZE`는 `16` kilobytes로 확장될 것이고 이는 thread의 stack의 size를 나타냅니다. 왜 `thread` 일까요? 이미 알고 계실 수도 있지만 각 [process](https://en.wikipedia.org/wiki/Process_%28computing%29)는 [parent processes](https://en.wikipedia.org/wiki/Parent_process)와 [child processes](https://en.wikipedia.org/wiki/Child_process)를 가질 수 있습니다. 실제로 parent process와 child process는 stack에서 다릅니다. 새로운 process를 위해서는 새로운 kernel stack이 할당됩니다. linux kernel에서는 이 stack이 `thread_info` structure의 [union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B)으로 표현됩니다.

`init_thread_union`은 `thread_union`으로 표현됩니다. 그리고 `thread_union`은 [include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h) file에 정의되어 있고 다음과 같습니다:

```C
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
	struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

`CONFIG_ARCH_TASK_STRUCT_ON_STACK` kernel configuration option은 오직 `ia64` architecture에서만 enable되고, `CONFIG_THREAD_INFO_IN_TASK` kernel configuration option은 `x86_64` architecture에서 enable됩니다. 그러므로 `thread_info` structure는 `task_struct` structure의 `thread_union` union 대신 위치할 것입니다.

`init_thread_union`은 [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/blob/master/include/asm-generic/vmlinux.lds.h) file에 `INIT_TASK_DATA` macro의 일부로 위치하고 다음과 같습니다:

```C
#define INIT_TASK_DATA(align)  \
	. = ALIGN(align);      \
	...                    \
	init_thread_union = .; \
	...
```

이 macro는 [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S) file에서 다음과 같이 사용됩니다:

```
.data : AT(ADDR(.data) - LOAD_OFFSET) {
	...
	INIT_TASK_DATA(THREAD_SIZE)
	...
} :data
```

`init_thread_union`은 `16` kilobytes인 `THREAD_SIZE`에 align된 address로 초기화됩니다.

이제 다음 표현을 이해할 수 있을겁니다:

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union + THREAD_SIZE - SIZEOF_PTREGS
```

`initial_stack` symbol은 `thread_union.stack` array + `THREAD_SIZE` (16 kilobytes) - `SIZEOF_PTREGS`(in-kernel에서 stack의 끝을 확실하게 detect하게 해주는 관습)의 시작을 가르킵니다.

early boot stack이 설정된 후, `lgdt` instruction을 사용하여 [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)를 update합니다:

```assembly
lgdt	early_gdt_descr(%rip)
```

여기서 `early_gdt_descr`은 다음과 같이 정의되어 있습니다:

```assembly
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	.quad	INIT_PER_CPU_VAR(gdt_page)
```

현재 kernel이 low userspace address에서 동작하고 있지만, 곧 kernel space에서 동작할 것이기 때문에 `Global Descriptor Table`을 reload해야 합니다.

이제 `early_gdt_descr`의 정의를 살펴봅시다. `GDT_ENTRIES`는 `32`로 확장되고 Global Descriptor Table은 kernel code, data, thread local storage segments 등을 위해 `32`개의 entries를 가집니다.

이제 `early_gdt_descr_base`의 정의를 살펴봅시다. `gdt_page` structure는 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)에 다음과 같이 정의되어 있습니다:

```C
struct gdt_page {
	struct desc_struct gdt[GDT_ENTRIES];
} __attribute__((aligned(PAGE_SIZE)));
```

이것은 다음과 같이 정의되어 있는 `desc_struct` struct의 array인 `gdt`의 하나의 field를 가지고 있습니다:

```C
struct desc_struct {
         union {
                 struct {
                         unsigned int a;
                         unsigned int b;
                 };
                 struct {
                         u16 limit0;
                         u16 base0;
                         unsigned base1: 8, type: 4, s: 1, dpl: 2, p: 1;
                         unsigned limit: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8;
                 };
         };
 } __attribute__((packed));
```

이것은 `GDT` descriptor와 유사합니다. `gdt_page` structure는 `PAGE_SIZE` 즉 `4096` bytes에 align되어 있다는 것에 주목하십시요. 이것은 `gdt`가 하나의 page를 점유한다는 것을 의미합니다.

이제 `INIT_PER_CPU_VAR`이 무엇인지 이해해 봅시다. `INIT_PER_CPU_VAR`은 [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/percpu.h)에 정의되어 있는 macro로 `init_per_cpu__`와 전달받은 parameter를 이어줍니다:

```C
#define INIT_PER_CPU_VAR(var) init_per_cpu__##var
```

`INIT_PER_CPU_VAR` macro가 확장되면 `init_per_cpu__gdt_page`가 됩니다. [linker script](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S)에서 `init_per_cpu__gdt_page`의 초기화를 볼 수 있습니다:

```
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(gdt_page);
```
`INIT_PER_CPU_VAR`에서 `init_per_cpu__gdt_page`를 얻었고 linker script의 `INIT_PER_CPU` macro는 확장되기 때문에, `__per_cpu_load`로 부터의 offset을 얻을 것입니다. 이 계산 이후, 새로운 GDT의 정확한 base address를 가지게 됩니다.

일반적으로 per-CPU variables는 2.6 kernel의 feature입니다. 이것이 무엇인지는 이름에서 알 수 있습니다. `per-CPU` variable을 생성하면, 각 CPU는 이 variable의 자신만의 copy를 가지게 됩니다. 여기서 `gdt_page` per-CPU variable을 생성하였습니다. 이러한 type의 variable은 각 CPU는 variable의 자신만의 copy를 사용하기 때문에 lock이 필요없다는 등 여러가지 장점이 있습니다. 그래서 multiprocessor의 각 core는 자신만이 `GDT` table을 가지고 이 table의 각 entry는 해당 core에서 수행되는 thread에서 access될 수 있는 memory segment를 나타냅니다. `per-CPU` variable에 자세한 사항은 [Concepts/per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)에서 읽어 볼 수 있습니다.

새로운 Global Descriptor Table을 load했기 때문에, 매번 그러하듯이 segments를 reload합니다:

```assembly
	xorl %eax,%eax
	movl %eax,%ds
	movl %eax,%ss
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs
```

이 모든 step이후 [interrupts](https://en.wikipedia.org/wiki/Interrupt)가 handle되는 곳에서 사용될 special stack을 나타내는 `irqstack`을 가르키는 `gs` register를 설정합니다:

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

여기서 `MSR_GS_BASE`는 다음과 같습니다:

```C
#define MSR_GS_BASE             0xc0000101
```

`MSR_GS_BASE`를 `ecx` register에 넣고 `wrmsr` instruction으로 `eax`와 (`initial_gs`를 가르키고 있는) `edx`로부터 data를 load합니다. 64-bit mode에서는 address를 하기 위해 `cs`, `fs`, `ds`, `ss` segment registers를 사용하지 않지만, `fs`와 `gs` registers는 사용될 수 있습니다. `fs`와 `gs`는 숨겨진 part이고 (real mode에서 `cs`를 본 것처럼) 이 part는 [Model Specific Registers](https://en.wikipedia.org/wiki/Model-specific_register)에 map된 descriptor를 가집니다. 위에서 볼 수 있는 것처럼 `0xc0000101`이 `gs.base` MSR address입니다. [system call](https://en.wikipedia.org/wiki/System_call)이나 [interrupt](https://en.wikipedia.org/wiki/Interrupt)가 발생하면 entry point에 kernel stack이 없습니다. 그래서 `MSR_GS_BASE`의 값이 interrupt stack의 address를 저장합니다.

다음 step으로 real mode의 bootparam structure의 address를 `rdi`에 넣습니다. (시작부터 `rsi`가 이 structure의 pointer를 가지고 있었음을 기억하십시요) 그리고 다음과 같이 C code로 jump합니다:

```assembly
	pushq	$.Lafter_lret	# put return address on stack for unwinder
	xorq	%rbp, %rbp	# clear frame pointer
	movq	initial_code(%rip), %rax
	pushq	$__KERNEL_CS	# set correct cs
	pushq	%rax		# target address in negative space
	lretq
.Lafter_lret:
```

여기서 `initial_code`의 address를 `rax`에 넣고 return address, `__KERNEL_CS,`, `initial_code`의 address를 stack에 push합니다. 이후 stack에서 return address를 추출하여 jump하는 `lretq` instruction을 볼 수 있습니다. (이제 stack에는 `initial_code`의 address가 있습니다) `initial_code`는 동일한 source code file에 정의되어 있고 다음과 같습니다:

```assembly
	.balign	8
	GLOBAL(initial_code)
	.quad	x86_64_start_kernel
	...
	...
	...
```

`initial_code`는 [arch/x86/kerne/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)에 정의되어 있는 `x86_64_start_kernel`의 address를 가지고 있으며 다음과 같습니다:

```C
asmlinkage __visible void __init x86_64_start_kernel(char * real_mode_data)
{
	...
	...
	...
}
```

이것은 `real_mode_data`라는 하나의 argument를 가지고 있습니다. (이전에 real mode data의 address를 `rdi` register에 전달했다는 것을 기억하십시요)

Next to start_kernel
--------------------------------------------------------------------------------

[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)에 정의되어 있는 "kernel entry point"인 start_kernel을 보기 전에 마지막 준비가 필요합니다.

먼저 `x86_64_start_kernel` 함수에서 몇가지 확인하는 것을 볼 수 있습니다:

```C
BUILD_BUG_ON(MODULES_VADDR < __START_KERNEL_map);
BUILD_BUG_ON(MODULES_VADDR - __START_KERNEL_map < KERNEL_IMAGE_SIZE);
BUILD_BUG_ON(MODULES_LEN + KERNEL_IMAGE_SIZE > 2*PUD_SIZE);
BUILD_BUG_ON((__START_KERNEL_map & ~PMD_MASK) != 0);
BUILD_BUG_ON((MODULES_VADDR & ~PMD_MASK) != 0);
BUILD_BUG_ON(!(MODULES_VADDR > __START_KERNEL));
MAYBE_BUILD_BUG_ON(!(((MODULES_END - 1) & PGDIR_MASK) == (__START_KERNEL & PGDIR_MASK)));
BUILD_BUG_ON(__fix_to_virt(__end_of_fixed_addresses) <= MODULES_END);
```

module space의 virtual address가 kernel text의 base address - `__START_KERNEL_map` 보다 작지 않은지, modules과 kernel text가 kernel image보다 작지 않은지 등과 같은 몇가지 다른 것들에 대한 확인을 합니다. `BUILD_BUG_ON`은 다음과 같은 macro입니다:

```C
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
```

이 trick이 어떻게 동작하는지 살펴봅시다. 첫번째 조건을 예로 살펴봅시다: `MODULES_VADDR < __START_KERNEL_map`. `!!conditions`는 `condition != 0`과 같습니다. 그래서 `MODULES_VADDR < __START_KERNEL_map`이면 true로 `!!(condition)`에서 `1`을 얻고 그렇지 않으면 0을 얻는다는 것을 의미합니다. `2*!!(condition)`로 `2` 또는 `0`을 얻게 됩니다. 계산을 끝마치면 2개의 다른 동작이 됩니다:

* 음수 index를 가진 char array의 size를 얻으려고 했기 때문에 compile error가 발생합니다. (위의 경우 `MODULES_VADDR`이 `__START_KERNEL_map`보다 작아질 수 없기 때문에 이렇게 될 수 있습니다);
* compile error가 발생하지 않습니다.

이것이 전부입니다. 상수 값에 따라 compile error를 발생시키는 흥미로운 C trick입니다.

다음 step으로 cpu별로 `cr4`의 shadown copy를 저장하는 `cr4_init_shadow` 함수 호출을 볼 수 있습니다. context switches가 `cr4`의 bits을 바꿀수 있기 때문에 각 CPU가 `cr4`를 저장해야 합니다. 이후 모든 page global directory entries를 reset하고 PGT의 새로운 pointer를 `cr3`에 쓰는 `reset_early_page_tables` 함수 호출을 볼 수 있습니다:

```C
	memset(early_top_pgt, 0, sizeof(pgd_t)*(PTRS_PER_PGD-1));
	next_early_pgt = 0;
	write_cr3(__sme_pa_nodebug(early_top_pgt));
```

곧 새로운 page table을 생성할 것입니다. 여기서 모든 Page Global Directory entries를 0으로 채우는 것을 볼 수 있습니다. 이후 `next_early_pgt`를 0으로 설정하고 (이에 대한 자세한 내용은 다음 post에서 볼 것입니다) `early_top_pgt`의 physical address를 `cr3`에 씁니다.

이후 `__bss_stop`부터 `__bss_start`까지 `_bss`를 clear하고 `init_top_pgt`도 clear합니다. `init_top_pgt`는 [arch/x86/kerne/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)에 다음과 같이 정의되어 있습니다:

```assembly
NEXT_PGD_PAGE(init_top_pgt)
	.fill	512,8,0
	.fill	PTI_USER_PGD_FILL,8,0
``` 

이것은 정확하게 `early_top_pgt`와 동일한 정의입니다.

다음 step은 early `IDT` handler를 설정하는 것이 될 것입니다. 하지만 이것은 큰 concept로 다음 post에서 살펴볼 것입니다.

Conclusion
--------------------------------------------------------------------------------

이것이 linux kernel initialization에 대한 첫번째 part의 끝입니다.

질문이나 제안이 있으시면 Twitter [0xAX](https://twitter.com/0xAX)나 [email](anotherworldofworld@gmail.com)로 연락을 주시거나, [issue](https://github.com/0xAX/linux-internals/issues/new)를 생성해 주십시요.

다음 part에서는 early interrupt handler, kernel space memory mapping 등의 초기화를 살펴볼 것입니다.

**영어는 저의 모국어가 아닙니다. 불편한 점이 있으셨다면 먼저 사죄드립니다. 만약 잘못된 점을 발견하시면 [linux-insides](https://github.com/0xAX/linux-internals)로 PR을 보내주십시요.**

Links
--------------------------------------------------------------------------------

* [Model Specific Register](http://en.wikipedia.org/wiki/Model-specific_register)
* [Paging](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html)
* [Previous part - kernel load address randomization](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-6.html)
* [NX](http://en.wikipedia.org/wiki/NX_bit)
* [ASLR](http://en.wikipedia.org/wiki/Address_space_layout_randomization)
