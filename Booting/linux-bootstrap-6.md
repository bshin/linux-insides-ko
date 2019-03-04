Kernel booting process. Part 6.
================================================================================

Introduction
--------------------------------------------------------------------------------

이것은 `kernel booting process` series의 여섯번째 part입니다. 이전 [part](linux-bootstrap-5.md)에서 kernel boot process의 마지막을 살펴보았습니다. 하지만 일부 중요한 고급 part를 skip했습니다.

기억하시겠지만 linux kernel의 entry point는 `start_kernel` 함수입니다. 이 함수는 [main.c](https://github.com/torvalds/linux/blob/v4.16/init/main.c) source code file에 정의되어 있고 `LOAD_PHYSICAL_ADDR` address에서 수행됩니다. 이 address는 `CONFIG_PHYSICAL_START` kernel configuration option에 의해 결정되며 default는 `0x1000000` 입니다:

```
config PHYSICAL_START
	hex "Physical address where the kernel is loaded" if (EXPERT || CRASH_DUMP)
	default "0x1000000"
	---help---
	  This gives the physical address where the kernel is loaded.
      ...
      ...
      ...
```

이 값은 kernel configuration에서 변경될 수 있고 또한 load address도 random value로 선택될 수 있습니다. 이것을 위해서 kernel configuration시 `CONFIG_RANDOMIZE_BASE` kernel configuration option을 enable하면 됩니다.

이 경우 linux kernel image가 압축 해제되고 load될 physical address가 randomize됩니다. 이 part에서는 이 option이 enable되었을 경우를 살펴볼 것입니다. kernel image의 load address는 [security 이유](https://en.wikipedia.org/wiki/Address_space_layout_randomization)로 randomize됩니다.

Initialization of page tables
--------------------------------------------------------------------------------

kernel decompressor가 압축 kernel을 압축 해제하고 load할 random memory range를 찾기 전에 identity mapped page tables이 초기화되어야 합니다. 만약 [bootloader](https://en.wikipedia.org/wiki/Booting)가 [16-bit or 32-bit boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt)를 사용한다면 이미 page tables이 존재합니다. 하지만 kernel decompressor가 이것 외의 memory range를 선택한다면 필요에 따라 새로운 page가 필요할 수도 있습니다. 이것이 새로운 identity mapped page tables를 만들어야 하는 이유입니다.

네, identity mapped page tables을 생성하는 것이 load address의 randomization의 첫번째 step입니다. 하지만 그것을 살펴보기 전에 어디서 여기로 왔는지 기억해 봅시다.

이전 [part](linux-bootstrap-5.md)에서 [long mode](https://en.wikipedia.org/wiki/Long_mode) 천이와 kernel decompressor entry point - `extract_kernel` 함수로의 jump하는 것을 살펴보았습니다. randomization은 여기서 `choose_random_location` 함수를 호출하는 것으로 시작됩니다:

```C
void choose_random_location(unsigned long input,
                            unsigned long input_size,
                            unsigned long *output,
                            unsigned long output_size,
                            unsigned long *virt_addr)
{}
```

보시는 바와 같이 이 함수는 다음의 5개의 parameter를 받습니다:

  * `input`;
  * `input_size`;
  * `output`;
  * `output_isze`;
  * `virt_addr`.

이 parameter가 무엇인지 이해해 봅시다. 첫번째 `input` parameter는 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c) source code file의 `extract_kernel` 함수의 parameter에서 옵니다:

```C
asmlinkage __visible void *extract_kernel(void *rmode, memptr heap,
				                          unsigned char *input_data,
				                          unsigned long input_len,
				                          unsigned char *output,
				                          unsigned long output_len)
{
  ...
  ...
  ...
  choose_random_location((unsigned long)input_data, input_len,
                         (unsigned long *)&output,
				         max(output_len, kernel_total_size),
				         &virt_addr);
  ...
  ...
  ...
}
```

이 parameter는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)의 assembler code에서 전달됩니다:

```C
leaq	input_data(%rip), %rdx
```

`input_data`는 작은 [mkpiggy](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/mkpiggy.c) program에서 생성됩니다. linux kernel source code를 compile하면 이 program이 생성한 file을 `linux/arch/x86/boot/compressed/piggy.S`에서 찾을 수 있습니다. 저의 경우 이 file은 다음과 같았습니다:

```assembly
.section ".rodata..compressed","a",@progbits
.globl z_input_len
z_input_len = 6988196
.globl z_output_len
z_output_len = 29207032
.globl input_data, input_data_end
input_data:
.incbin "arch/x86/boot/compressed/vmlinux.bin.gz"
input_data_end:
```

보시는 것처럼 이것은 4개의 global symbols을 포함합니다. 처음 두개는 `z_input_len`과 `z_output_len`으로 `vmlinux.bin.gz`의 압축된 것과 압축 해제된 것의 size입니다. 세번째는 `input_data`로 raw binary format (모든 debugging symbols, comment, relocation 정보가 strip된 것)의 linux kernel image을 가르킵니다. 그리고 마지막 `input_data_end`는 압축된 linux image의 마지막을 가르킵니다.

`choose_random_location` 함수의 첫번째 parameter는 `piggy.o` object file을 포함하고 있는 압축된 kernel image를 가르킵니다.

`choose_random_location` 함수의 두번째 parameter는 방금 본 `z_input_len` 입니다.

`choose_random_location` 함수의 세번째와 네번째 parameters는 압축 해제한 kernel image를 위치시킬 address와 압축 해제한 kernel image의 크기입니다. 압축 해제한 kernel을 저장할 address는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)에서 오고 그것은 2 megabytes boundary에 align된 `startup_32`의 address입니다. 압축 해제한 kernel의 size는 `piggy.S`에서 오고 그것은 `z_output_len`입니다.

`choose_random_location` 함수의 마지막 parameter는 kernel load address의 virtual address입니다. 하기에서 보시는 것과 같이 default로 그것은 default physical load address와 일치합니다:

```C
unsigned long virt_addr = LOAD_PHYSICAL_ADDR;
```

그리고 그것은 kernel configuration에 의해 결정됩니다:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

지금까지 `choose_random_location` 함수의 parameters를 살펴보았습니다. 이제 구현을 살펴봅시다. 이 함수는 kernel command line의 `nokaslr` option을 확인하는 것으로 시작합니다:

```C
if (cmdline_find_option_bool("nokaslr")) {
	warn("KASLR disabled: 'nokaslr' on cmdline.");
	return;
}
```

만약 이 option이 주어졌다면 `choose_random_location` 함수를 나가고 kernel load address는 randomize되지 않습니다. 관련된 command line option은 [kernel documentation](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst)에서 찾아볼 수 있습니다:

```
kaslr/nokaslr [X86]

Enable/disable kernel and module base offset ASLR
(Address Space Layout Randomization) if built into
the kernel. When CONFIG_HIBERNATION is selected,
kASLR is disabled by default. When kASLR is enabled,
hibernation will be disabled.
```

kernel command line에 `nokaslr`을 전달하지 않고 `CONFIG_RANDOMIZE_BASE` kernel configuration option이 enable되어 있다고 가정해 봅시다. 이 경우 kernel load flags에 `kASLR` flag를 추가합니다:

```C
boot_params->hdr.loadflags |= KASLR_FLAG;
```

다음 step은 `initialize_identity_maps` 함수를 호출하는 것입니다:

```C
initialize_identity_maps();
```

이 함수는 [arch/x86/boot/compressed/kaslr_64.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr_64.c) source code file에 정의되어 있습니다. 이 함수는 `x86_mapping_info` structure의 instance인 `mapping_info`를 초기화 하는 것으로 시작합니다:

```C
mapping_info.alloc_pgt_page = alloc_pgt_page;
mapping_info.context = &pgt_data;
mapping_info.page_flag = __PAGE_KERNEL_LARGE_EXEC | sev_me_mask;
mapping_info.kernpg_flag = _KERNPG_TABLE;
```

`x86_mapping_info` structure는 [arch/x86/include/asm/init.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/init.h) header file에 정의되어 있고 다음과 같습니다:

```C
struct x86_mapping_info {
	void *(*alloc_pgt_page)(void *);
	void *context;
	unsigned long page_flag;
	unsigned long offset;
	bool direct_gbpages;
	unsigned long kernpg_flag;
};
```

이 structure는 memory mappings에 대한 정보를 제공합니다. 이전 part에서 기억하시는 것처럼, 이미 0부터 `4G`까지의 page tables을 setup하였습니다. 여기서 load kernel의 random position에 따라 `4G` 상위의 memory를 access해야 할 수도 있습니다. 그래서 `initialize_identity_maps` 함수는 새로운 page table이 필요한 경우를 위해 memory region을 초기화 합니다. 먼저 `x86_mapping_info` structure의 정의를 살펴봅시다.

`alloc_pgt_page`는 page table entry를 위한 space를 allocate하기 위해 호출되는 callback 함수입니다. `context` field는 allocate된 page table을 track하기 위한 `alloc_pgt_data` structure의 instance입니다. `page_flag`와 `kernpg_flag` field는 page flags입니다. 첫번째 것은 `PMD` 혹은 `PUD` entries를 위한 flag를 나타냅니다. 두번째 `kernpg_flag` field는 나중에 overridden될 수 있는 kernel pages를 위한 flags를 나타냅니다. `direct_gbpage` field는 huge pages를 위한 flag를 나타냅니다. 마지막 `offset` field는 kernel virtual address와 physical address의 `PMD` level까지의 offset을 나타냅니다.

`alloc_pgt_page` callback은 새로운 page를 위한 공간이 있는지 확인하고 새로운 page를 할당합니다:

```C
entry = pages->pgt_buf + pages->pgt_buf_offset;
pages->pgt_buf_offset += PAGE_SIZE;
```

`alloc_pgt_data` structure로부터 page가 할당됩니다:

```C
struct alloc_pgt_data {
	unsigned char *pgt_buf;
	unsigned long pgt_buf_size;
	unsigned long pgt_buf_offset;
};
```

`alloc_pgt_page` callback은 새로운 page의 address를 return합니다. `initialize_identity_maps` 함수의 최종 목적은 `pgdt_buf_size`와 `pgt_buf_offset`을 초기화하는 것입니다. 초기화 phase로 `initialize_identity_maps` 함수는 `pgt_buf_offset`을 0으로 설정합니다:

```C
pgt_data.pgt_buf_offset = 0;
```

`pgt_data.pgt_buf_size`는 bootloader에서 어떤 boot protocol (64-bit 혹은 32-bit)을 사용하였는지에 따라 `77824` 혹은 `69632`로 설정됩니다. `pgt_data.pgt_buf`도 동일합니다. bootloader가 kernel을 `startup_32`에 load했다면 `pgdt_data.pgdt_buf`는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)에서 이미 초기화된 page table의 마지막을 가르키게 됩니다:

```C
pgt_data.pgt_buf = _pgtable + BOOT_INIT_PGT_SIZE;
```

여기서 `_pgtable`은 page table [_pgtable](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/vmlinux.lds.S)의 시작을 가르킵니다. 다른 방식으로 bootloader가 64-bit boot protocol을 사용하여 kernel을 `startup_64`에 load하였다면 early page table은 bootloader가 생성하고 `_pgtable`이 이를 overwrite합니다:

```C
pgt_data.pgt_buf = _pgtable
```

새로운 page tables을 위한 buffer가 초기화되면 `choose_random_location` 함수로 돌아갑니다.

Avoid reserved memory ranges
--------------------------------------------------------------------------------

identity page table 관련 작업이 초기화된 후, 압축 해제한 kernel image를 위치시킬 random location의 선택을 시작합니다. 하지만 예상하신 바와 같이 아무 address나 선택할 수는 없습니다. memory range에는 일부 reserved address가 있습니다. 이러한 address는 [initrd](https://en.wikipedia.org/wiki/Initial_ramdisk), kernel command line 등의 중요한 것에 점유되어 있습니다.

```C
mem_avoid_init(input, input_size, *output);
```

`mem_avoid_init` 함수는 모든 non-safe memory를 mem_avoid array에 모읍니다:

```C
struct mem_vector {
	unsigned long long start;
	unsigned long long size;
};

static struct mem_vector mem_avoid[MEM_AVOID_MAX];
```

여기서 `MEM_AVOID_MAX`는 reserved memory region의 다른 type을 나타내는 `mem_avoid_index` [enum](https://en.wikipedia.org/wiki/Enumerated_type#C)에서 옵니다:

```C
enum mem_avoid_index {
	MEM_AVOID_ZO_RANGE = 0,
	MEM_AVOID_INITRD,
	MEM_AVOID_CMDLINE,
	MEM_AVOID_BOOTPARAMS,
	MEM_AVOID_MEMMAP_BEGIN,
	MEM_AVOID_MEMMAP_END = MEM_AVOID_MEMMAP_BEGIN + MAX_MEMMAP_REGIONS - 1,
	MEM_AVOID_MAX,
};
```

둘 다 [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr.c) source code file에 정의되어 있습니다.

`mem_avoid_init` 함수의 구현을 살펴봅시다. 이 함수의 주 목적은 `mem_avoid_index` enum에 기술되어 있는 reserved memory region이 정보를 `mem_avoid` array에 저장하는 것입니다. 그리고 새로운 identity mapped buffer에 이러한 region을 위한 새로운 page를 생성합니다. `mem_avoid_index` 함수의 여러 부분이 비슷합니다. 그 중 하나를 살펴봅시다:

```C
mem_avoid[MEM_AVOID_ZO_RANGE].start = input;
mem_avoid[MEM_AVOID_ZO_RANGE].size = (output + init_size) - input;
add_identity_map(mem_avoid[MEM_AVOID_ZO_RANGE].start,
		 mem_avoid[MEM_AVOID_ZO_RANGE].size);
```

`mem_avoid_init` 함수의 시작에서 현재 kernel 압축 해제를 위한 memory region을 회피하기 위한 시도합니다. `mem_avoid` array의 entry를 이러한 region의 시작 위치와 size를 채우고 이 region을 위한 identity mapped page를 생성하는 `add_identity_map` 함수를 호출합니다. `add_identity_map` 함수는 [arch/x86/boot/compressed/kaslr_64.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr_64.c)에 정의되어 있고 다음과 같습니다:

```C
void add_identity_map(unsigned long start, unsigned long size)
{
	unsigned long end = start + size;

	start = round_down(start, PMD_SIZE);
	end = round_up(end, PMD_SIZE);
	if (start >= end)
		return;

	kernel_ident_mapping_init(&mapping_info, (pgd_t *)top_level_pgt,
				  start, end);
}
```

보시는 바와 같이 이것은 2 megabytes boundary에 align시키고 전달받은 시작과 끝 address를 확인합니다.

마지막으로 [arch/x86/mm/ident_map.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/mm/ident_map.c) source code file에 정의되어 있는 `kernel_ident_mapping_init` 함수를 호출하고 위에서 초기화된 `mapping_info` instance, top level page table의 address, 새로운 identity mapping이 생성될 memory region의 address를 전달합니다.

`kernel_ident_mapping_init` 함수는 flag가 주어지지 않으면 새로운 page에 default flag를 설정합니다:

```C
if (!info->kernpg_flag)
	info->kernpg_flag = _KERNPG_TABLE;
```

그리고 전달받은 address에 관련된 새로운 2-megabytes (`mapping_info.page_flag`에 `PSE` bit 때문) page entries ([five-level page tables](https://lwn.net/Articles/717293/) 인 경우 `PGD -> P4D -> PUD -> PMD` 혹은 [four-level page tables](https://lwn.net/Articles/117749/) 인 경우 `PGD -> PUD -> PMD`) 를 생성합니다.

```C
for (; addr < end; addr = next) {
	p4d_t *p4d;

	next = (addr & PGDIR_MASK) + PGDIR_SIZE;
	if (next > end)
		next = end;

    p4d = (p4d_t *)info->alloc_pgt_page(info->context);
	result = ident_p4d_init(info, p4d, addr, next);

    return result;
}
```

먼저 전달받은 address의 `Page Global Directory`의 다음 entry를 찾고 이것이 전달받은 memory region의 `end`보다 크면 그것을 `end`로 설정합니다. 이후 이미 위에서 살펴본 `x86_mapping_info` callback을 사용하여 새로운 page를 할당받고 `ident_p4d_init` 함수를 호출합니다. `ident_p4d_init` 함수는 low-level page directories (`p4d` -> `pud` -> `pmd`)를 위해 동일한 작업을 수행합니다.

이것이 전부입니다.

reserved address 관련된 새로운 page entries가 page table에 생성되었습니다. 이것이 `mem_avoid_init` 함수의 끝이 아닙니다. 하지만 다른 부분은 비슷합니다. 이것은 [initrd](https://en.wikipedia.org/wiki/Initial_ramdisk), kernel command line 등을 위한 page를 생성합니다.

이제 `choose_random_location` 함수로 돌아가봅시다.

Physical address randomization
--------------------------------------------------------------------------------

reserved memory region이 `mem_avoid` array에 저장되고 이것들을 위한 identity mapping pages가 생성된 후, kernel을 압축 해제하기 위한 random memory region을 선택하기 위해 최소 가능 address를 선택합니다:

```C
min_addr = min(*output, 512UL << 20);
```

보시는 바와 같이 `512` megabytes보다 작습니다. 이 `512` megabytes는 하위 memory의 알수 없는 것들을 회피하기 위해 선택되었습니다.

다음 step은 kernel을 load하기 위한 random physical과 virtual address를 선택하는 것입니다. 먼저 physical address입니다:

```C
random_addr = find_random_phys_addr(min_addr, output_size);
```

`find_random_phys_addr` 함수는 [same](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr.c) source code file에 정의되어 있습니다:

```
static unsigned long find_random_phys_addr(unsigned long minimum,
                                           unsigned long image_size)
{
	minimum = ALIGN(minimum, CONFIG_PHYSICAL_ALIGN);

	if (process_efi_entries(minimum, image_size))
		return slots_fetch_random();

	process_e820_entries(minimum, image_size);
	return slots_fetch_random();
}
```

`process_efi_entries` 함수의 주 목적은 전체 accessible memory에서 kernel을 load하기 위한 모든 가능한 memory rabge를 찾는 것입니다. kernel이 [EFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) 지원없이 system에서 compile되고 수행되었다면, [e820](https://en.wikipedia.org/wiki/E820) regions에서 이러한 memory를 찾습니다. 발견된 모든 memroy region은 `slot_areas`에 저장됩니다:

```C
struct slot_area {
	unsigned long addr;
	int num;
};

#define MAX_SLOT_AREA 100

static struct slot_area slot_areas[MAX_SLOT_AREA];
```

kernel이 압축 해제될 곳을 위해 이 array에서 random index를 선택합니다. 이 선택은 `slots_fetch_random` 함수에서 수행됩니다. `slots_fetch_random` 함수의 주 목적은 `kaslr_get_random_long` 함수를 통해 `slot_areas` array에서 random한 memory range를 선택하는 것입니다:

```C
slot = kaslr_get_random_long("Physical") % slot_max;
```

`kaslr_get_random_long` 함수는 [arch/x86/lib/kaslr.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/lib/kaslr.c) source code file에 정의되어 있고 단순히 random number를 return합니다. kernel configuration과 system opportunities에 따라 다른 방법으로 random number를 얻는 것에 주목하십시요. ([time stamp counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter), [rdrand](https://en.wikipedia.org/wiki/RdRand) 등을 통해 얻음)

이것이 전부입니다. random한 memory range가 선택될 것입니다.

Virtual address randomization
--------------------------------------------------------------------------------

kernel decompressor에 의해 random한 memory region이 선택된 후, 필요에 따라 이 영역을 위한 identity mapped pages가 생성됩니다:

```C
random_addr = find_random_phys_addr(min_addr, output_size);

if (*output != random_addr) {
		add_identity_map(random_addr, output_size);
		*output = random_addr;
}
```

여기부터 `output`은 kernel이 압축 해제될 memory region의 base address를 저장할 것입니다. 하지만 지금 이 순간, 기억하시는 것처럼 오직 physical address만 randomize되었습니다. [x86_64](https://en.wikipedia.org/wiki/X86-64) architecture에서는 virtual address도 randomize되어야 합니다:

```C
if (IS_ENABLED(CONFIG_X86_64))
	random_addr = find_random_virt_addr(LOAD_PHYSICAL_ADDR, output_size);

*virt_addr = random_addr;
```

`x86_64` architecture가 아닌 경우에서 보시는 것처럼, virtual address의 randomize는 physical address의 randomize와 동시에 일어납니다. `find_random_virt_addr` 함수가 kernel image를 저장할 virtual memory ranges의 양을 계산하고 이전 random `physical` address를 찾을때 살펴본 `kaslr_get_random_long` 함수를 호출합니다.

이 순간부터 base physical address (`*output`)와 base virtual address (`*virt_addr`)가 모두 randomize되었습니다.

이것이 전부입니다.

Conclusion
--------------------------------------------------------------------------------

이것의 linux kernel booting process의 여섯번째 그리고 마지막 part의 끝입니다. 이제 더 이상 kernel booting에 관한 post는 없을 것입니다. (이전 post에 대한 update는 있을 수 있습니다) 하지만 다른 kernel internals에 대한 많은 post가 있을 것입니다.

다음 chapter는 kernel initialization에 관한 것이 될 것입니다. linux kernel initialization code의 첫번째 step을 살펴볼 것입니다.

질문이나 제안이 있으시면 comment를 남기시거나 [twitter](https://twitter.com/0xAX)로 알려 주십시요.

**영어는 저의 모국어가 아닙니다. 불편한 점이 있으셨다면 먼저 사죄드립니다. 만약 잘못된 점을 발견하시면 [linux-insides](https://github.com/0xAX/linux-internals)로 PR을 보내주십시요.**

Links
--------------------------------------------------------------------------------

* [Address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [Linux kernel boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt)
* [long mode](https://en.wikipedia.org/wiki/Long_mode)
* [initrd](https://en.wikipedia.org/wiki/Initial_ramdisk)
* [Enumerated type](https://en.wikipedia.org/wiki/Enumerated_type#C)
* [four-level page tables](https://lwn.net/Articles/117749/)
* [five-level page tables](https://lwn.net/Articles/717293/)
* [EFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
* [e820](https://en.wikipedia.org/wiki/E820)
* [time stamp counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [rdrand](https://en.wikipedia.org/wiki/RdRand)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Previous part](linux-bootstrap-5.md)
