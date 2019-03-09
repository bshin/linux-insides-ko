Per-CPU variables
================================================================================
per-CPU variables는 kernel features중 하나입니다. 그 이름에서 이 feature의 의미를 이해할 수 있습니다. variable을 생성하고 각 processor core는 이 variable의 자신만의 copy를 가집니다. 이 part에서는 이 feature를 자세히 살펴보고 어떻게 구현되었고 어떻게 동작하는지 이해해 봅시다.

kernel은 per-cpu variables을 위한 API를 제공합니다 - `DEFINE_PER_CPU` macro:

```C
#define DEFINE_PER_CPU(type, name) \
        DEFINE_PER_CPU_SECTION(type, name, "")
```

이 macro는 per-cpu variables과 같이 동작하는 다른 많은 macros와 같이 [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/percpu-defs.h)에 정의되어 있습니다. 이제 이 feature가 어떻게 구현되어 있는지 살펴봅시다.

`DEFINE_PER_CPU` 정의를 살펴봅시다. 이것은 2개의 parameters를 받습니다: `type`과 `name`, 그리고 이것을 per-cpu variables를 생성하는데 사용합니다. 예를 들면:

```C
DEFINE_PER_CPU(int, per_cpu_n)
```

생성할 variable의 type과 name을 전달합니다. `DEFINE_PER_CPU`는 `DEFINE_PER_CPU_SECTION` macro를 호출하고 동일한 2개의 parameters와 빈 string을 전달합니다. `DEFINE_PER_CPU_SECTION`의 정의를 살펴봅시다:

```C
#define DEFINE_PER_CPU_SECTION(type, name, sec)    \
         __PCPU_ATTRS(sec) PER_CPU_DEF_ATTRIBUTES  \
         __typeof__(type) name
```

```C
#define __PCPU_ATTRS(sec)                                                \
         __percpu __attribute__((section(PER_CPU_BASE_SECTION sec)))     \
         PER_CPU_ATTRIBUTES
```

여기서 `section`은 다음과 같습니다:

```C
#define PER_CPU_BASE_SECTION ".data..percpu"
```

모든 macros가 확장된 후 global per-cpu variable을 얻을 수 있습니다:

```C
__attribute__((section(".data..percpu"))) int per_cpu_n
```

이것은 `.data.percpu` section에 `per_cpu_n` variable이 있다는 것을 의미합니다. 이 section은 `vmlinux`에서 찾아볼 수 있습니다:

```
.data..percpu 00013a58  0000000000000000  0000000001a5c000  00e00000  2**12
              CONTENTS, ALLOC, LOAD, DATA
```

자, 이제 `DEFINE_PER_CPU` macro를 사용하면 `.data..percpu` section에 per-cpu variable이 생성된다는 것을 알았습니다. kernel이 초기화될때 `.data..percpu` section을 load하는 `setup_per_cpu_areas`를 CPU당 한 section씩 여러차례 호출합니다.

per-CPU areas의 초기화 과정을 살펴봅시다. 이것은 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에서 [arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup_percpu.c)에 정의되어 있는 `setup_per_cpu_areas` 함수를 호출하는 것으로부터 시작합니다.

```C
pr_info("NR_CPUS:%d nr_cpumask_bits:%d nr_cpu_ids:%d nr_node_ids:%d\n",
        NR_CPUS, nr_cpumask_bits, nr_cpu_ids, nr_node_ids);
```

`setup_per_cpu_areas`는 kernel configuration에서 `CONFIG_NR_CPUS` configuration option으로 설정된 최대 CPU 개수, 실제 CPUs의 개수, (새로운 `cpumask` operators에서 `NR_CPUS`와 같은) `nr_cpumask_bits`, `NUMA` nodes의 개수에 대한 정보를 출력하는 것으로부터 시작합니다.

이 출력은 dmesg에서 볼 수 있습니다:

```
$ dmesg | grep percpu
[    0.000000] setup_percpu: NR_CPUS:8 nr_cpumask_bits:8 nr_cpu_ids:8 nr_node_ids:1
```

다음 step으로 `purcpu` 첫번째 chunk allocator를 확인합니다. 모든 percpu areas는 chunk로 allocate됩니다. 첫번째 chunk는 static percpu variable을 위해 사용됩니다. linux kernel은 첫번째 chunk allocator의 type을 설정하는 `percpu_alloc`이라는 command line parameter를 가지고 있습니다. kernel documentation에서 찾아볼 수 있습니다:

```
percpu_alloc=	Select which percpu first chunk allocator to use.
		Currently supported values are "embed" and "page".
		Archs may support subset or none of the	selections.
		See comments in mm/percpu.c for details on each
		allocator.  This parameter is primarily	for debugging
		and performance comparison.
```

[mm/percpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/percpu.c)에 이 command line option에 대한 handler가 있습니다:

```C
early_param("percpu_alloc", percpu_alloc_setup);
```

여기서 `percpu_alloc_setup` 함수는 `percpu_alloc` parameter 값에 따라 `pcpu_chosen_fc` 값을 설정합니다. default로 첫번째 chunk allocator은 `auto` 입니다:

```C
enum pcpu_fc pcpu_chosen_fc __initdata = PCPU_FC_AUTO;
```

만약 `percpu_alloc` parameter가 kernel command line에서 전달되지 않으면, 첫번째 percpu chunk를 [memblock](https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html)으로 bootmem에 할당하는 `embed` allocator가 사용됩니다. 마지막 allocator는 첫번째 chunk를 `PAGE_SIZE` pages에 map하는 첫번째 chunk `page` allocator입니다.

위에 쓴것 처럼, 먼저 `setup_per_cpu_areas`에서 첫번째 chunk allocator type을 확인합니다. 첫번째 chunk allocator가 page가 아닌 것을 확인합니다:

```C
if (pcpu_chosen_fc != PCPU_FC_PAGE) {
    ...
    ...
    ...
}
```

`PCPU_FC_PAGE`가 아니면, `embed` allocator를 사용하고 첫번째 chunk를 위한 공간을 `pcpu_embed_first_chunk` 함수를 사용하여 allocate합니다:

```C
rc = pcpu_embed_first_chunk(PERCPU_FIRST_CHUNK_RESERVE,
					    dyn_size, atom_size,
					    pcpu_cpu_distance,
					    pcpu_fc_alloc, pcpu_fc_free);
```

위에서 본 것처럼, `pcpu_embed_first_chunk` 함수는 첫번째 percpu chunk를 bootmem에 넣고 몇가지 parameters를 `pcpu_embed_first_chunk`에 전달합니다. 이것들은 다음과 같습니다:

* `PERCPU_FIRST_CHUNK_RESERVE` - static `percpu` variables을 위한 reserved space의 크기;
* `dyn_size` - dynamic allocation을 위한 최소 free size (bytes);
* `atom_size` - 모든 allocations은 이 parameter의 배수가 되고 align 됩니다;
* `pcpu_cpu_distance` - cpus간의 distance를 알아내기 위한 callback;
* `pcpu_fc_alloc` - `percpu` page를 할당하기 위한 함수;
* `pcpu_fc_free` - `percpu` page를 해제하기 위한 함수.

`pcpu_embed_first_chunk`를 호출하기 전에 모든 parameters를 계산합니다:

```C
const size_t dyn_size = PERCPU_MODULE_RESERVE + PERCPU_DYNAMIC_RESERVE - PERCPU_FIRST_CHUNK_RESERVE;
size_t atom_size;
#ifdef CONFIG_X86_64
		atom_size = PMD_SIZE;
#else
		atom_size = PAGE_SIZE;
#endif
```

만약 첫번째 chunk allocator가 `PCPU_FC_PAGE`이면, `pcpu_embed_first_chunk` 대신 `pcpu_page_first_chunk`를 사용합니다. `percpu` areas가 올라온 후, `percpu` offset과 각 CPU를 위한 segment를 `setup_percpu_segment` 함수(오직 `x86` system에서만)를 이용하여 설정하고 일부 early data를 arrays에서 `percpu` variables로 옮깁니다. (`x86_cpu_to_apicid`, `irq_stack_ptr`, 등) kernel이 초기화 작업을 마친 후, CPUs 개수와 같은 N개의 `.data..percpu` section을 가지고, bootstrap processor가 사용한 section은 `DEFINE_PER_CPU` macro로 생성된 초기화 되지 않은 variable을 가지게 됩니다.

kernel은 per-cpu variables 조작을 위한 API를 제공합니다:

* get_cpu_var(var)
* put_cpu_var(var)


`get_cpu_var`의 구현을 살펴봅시다:

```C
#define get_cpu_var(var)     \
(*({                         \
         preempt_disable();  \
         this_cpu_ptr(&var); \
}))
```

linux kernel은 preemptible하고 per-cpu variable을 접근할 때 kernel이 어떤 processor에서 동작 중인지 알아야 합니다. 그래서 현재 code는 preempt되지 말아야 하고 per-cpu variable을 접근할 때 다른 CPU로 이동되지 말아야 합니다. 이것이 다음과 같이 먼저 `preempt_disable` 함수를 호출하고 다음에 `this_cpu_ptr` macro를 호출하는 이유입니다:

```C
#define this_cpu_ptr(ptr) raw_cpu_ptr(ptr)
```

그리고

```C
#define raw_cpu_ptr(ptr)        per_cpu_ptr(ptr, 0)
```

여기서 `per_cpu_ptr`은 전달한 cpu (두번째 parameter)의 per-cpu variable의 pointer를 return 합니다. per-cpu variable을 생성하고 수정한 후, `preempt_enable` 함수를 호출하여 preemption을 enable시키는 `put_cpu_var` macro를 호출해야 합니다. 그래서 per-cpu variable의 전형적인 사용 방법은 다음과 같습니다:

```C
get_cpu_var(var);
...
//Do something with the 'var'
...
put_cpu_var(var);
```

`per_cpu_ptr` macro를 살펴봅시다:

```C
#define per_cpu_ptr(ptr, cpu)                             \
({                                                        \
        __verify_pcpu_ptr(ptr);                           \
         SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu)));  \
})
```

위에 쓴것 처럼, 이 macro는 전달받은 cpu의 per-cpu variable을 return합니다. 먼저 `__verify_pcpu_ptr`을 호출합니다:

```C
#define __verify_pcpu_ptr(ptr)
do {
	const void __percpu *__vpp_verify = (typeof((ptr) + 0))NULL;
	(void)__vpp_verify;
} while (0)
```

이것은 전달받은 `ptr`을 `const void __percpu *` type으로 만듭니다,

이후 2개의 parameters로 `SHIFT_PERCPU_PTR` macro를 호출하는 것을 볼 수 있습니다. `per_cpu_offset` macro로 첫번째 parameter로 ptr을 전달하고 두번째 parameter로 cpu number를 전달합니다:

```C
#define per_cpu_offset(x) (__per_cpu_offset[x])
```

이것은 `__per_cpu_offset` array로부터 `x` element를 받는 것으로 확장됩니다:

```C
extern unsigned long __per_cpu_offset[NR_CPUS];
```

여기서 `NR_CPUS`는 CPUs의 number입니다. `__per_cpu_offset` array는 cpu-variable copies 간의 distances로 채워집니다. 예를 들면 모든 per-cpu data의 size가 `X` bytes이고 `__per_cpu_offset[Y]`를 접근하면, `X*Y`가 접근됩니다. `SHIFT_PERCPU_PTR` 구현을 살펴봅시다:

```C
#define SHIFT_PERCPU_PTR(__p, __offset)                                 \
         RELOC_HIDE((typeof(*(__p)) __kernel __force *)(__p), (__offset))
```

`RELOC_HIDE`는 단지 offset `(typeof(ptr)) (__ptr + (off))`를 return하고 이것은 variable을 가르키는 pointer를 return합니다.

이것이 전부입니다! 물론 이것은 전체 API가 아니라 general overview입니다. 어려울 수 있습니다만 per-cpu variables를 이해하기 위해서는 [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/percpu-defs.h) magic를 이해해야 합니다.

per-cpu variable의 pointer를 가져오는 algorithm을 다시 살펴봅시다:

* kernel은 초기화 과정에서 복수개의 `.data..percpu` section (cpu당 하나)를 생성합니다;
* `DEFINE_PER_CPU` macro로 생성되는 모든 variables는 첫번째 section 혹은 CPU0를 위한 section으로 relocate될 것입니다;
* `__per_cpu_offset` array는 `.data..percpu` sections간의 distance (`BOOT_PERCPU_OFFSET`)으로 채워집니다;
* `per_cpu_ptr`이 호출되면, 예를 들어 세번째 CPU를 위한 어떤 per-cpu variable의 pointer를 얻기 위해, `__per_cpu_offset` array가 접근되고, 여기서 모든 index는 요청한 CPU를 가르킵니다.

이것이 전부입니다.
