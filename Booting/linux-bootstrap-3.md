Kernel booting process. Part 3.
================================================================================

Video mode initialization and transition to protected mode
--------------------------------------------------------------------------------

이것은 `kernel booting process` series의 세번째 part입니다. 이전 [part](linux-bootstrap-2.md#kernel-booting-process-part-2)에서, [main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)에서 `set_video` routine을 호출하기 직전까지 살펴보았습니다.

이 part에서는 다음 사항을 살펴볼 것입니다:

* kernel setup code에서 video mode 초기화,
* protected mode로 천이되기 전에 이루어지는 준비,
* protected mode로의 천이

**NOTE** 만약 protected mode에 대해 알지 못한다면, 이전 [part](linux-bootstrap-2.md#protected-mode)에서 약간의 정보를 확인할 수 있습니다. 또한, 이에 대한 몇개의 [links](linux-bootstrap-2.md#links)도 있습니다.

이전에 쓴 것처럼, [arch/x86/boot/video.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video.c) source code file에 정의되어 있는 `set_video` 함수부터 시작할 것입니다. `boot_params.hdr` structure로부터 video mode를 얻는 것으로 시작하는 것을 보실 수 있습니다:

```C
u16 mode = boot_params.hdr.vid_mode;
```

그리고 이것은 (이전 post에서 볼 수 있듯이) `copy_boot_params` 함수에서 채웁니다. `vid_mode`는 bootloader가 채워야만 하는 의무적인 field입니다. kernel의 `boot_protocol`에서 이러한 정보를 찾을 수 있습니다:

```
Offset	Proto	Name		Meaning
/Size
01FA/2	ALL	    vid_mode	Video mode control
```

kernel boot protocol에서 다음과 같은 부분을 읽을 수 있습니다:

```
vga=<mode>
	<mode> here is either an integer (in C notation, either
	decimal, octal, or hexadecimal) or one of the strings
	"normal" (meaning 0xFFFF), "ext" (meaning 0xFFFE) or "ask"
	(meaning 0xFFFD).  This value should be entered into the
	vid_mode field, as it is used by the kernel before the command
	line is parsed.
```

grub (혹은 다른 bootloader)의 configuration에 `vga` option을 추가할 수 있으며, 이 option은 kernel command line으로 전달됩니다. 이 option은 description에 언급된 다른 값을 가질 수 있습니다. 예를 들면, 이 option은 integer number `0xFFFD` 혹은 `ask`가 될 수 있습니다. 만약 `vga`에 `ask`를 전달한다면, 다음과 같은 menu를 볼 수 있습니다:

![video mode setup menu](http://oi59.tinypic.com/ejcz81.jpg)

이것은 video mode의 선택을 요청하는 것입니다. 이 구현에 뛰어들 것입니다만, 그 전에 약간의 다른 것들을 살펴봐야 합니다.

Kernel data types
--------------------------------------------------------------------------------

이전에 kernel setup code에서 `u16`과 같은 상이한 data types의 정의를 살펴 보았습니다. kernel에서 제공하는 몇개의 data types을 살펴봅시다:


| Type | char | short | int | long | u8 | u16 | u32 | u64 |
|------|------|-------|-----|------|----|-----|-----|-----|
| Size |  1   |   2   |  4  |   8  |  1 |  2  |  4  |  8  |

kernel의 source code를 보실 때, 자주 이러한 것들을 보실 수 있으며 기억해 두면 좋을 것입니다.

Heap API
--------------------------------------------------------------------------------

`set_video` 함수에서 `boot_params.hdr`로부터 `vid_mode`를 가져온 이후, `RESET_HEAP` 함수를 호출하는 것을 볼 수 있습니다. `RESET_HEAP`은 [arch/x86/boot/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/boot.h) header file에 정의되어 있는 macro 입니다.

이 macro는 다음과 같이 정의되어 있습니다:

```C
#define RESET_HEAP() ((void *)( HEAP = _end ))
```

두번째 part를 읽으셨다면, [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) 함수로 heap을 초기화한 것을 기억하실 겁니다. `arch/x86/boot/boot.h` header file에 정의되어 있는 heap을 관리하기 위한 몇가지 utility macros와 함수가 있습니다.

그것들은 다음과 같습니다:

```C
#define RESET_HEAP()
```

위에서 본 것처럼, 이것은 `HEAP` variable에 `_end`를 설정하여 heap을 reset합니다. 여기서 `_end`는 단순히 `extern char _end[];`입니다.

다음은 heap allocation을 위한 `GET_HEAP` macro입니다:

```C
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))
```

이것은 3개의 parameters로 내부 함수 `__get_heap`을 호출합니다:

* allocate할 dataypte의 size
* 이 type의 variables이 어떻게 align될지 기술하는 `__alignof__(type)`
* 몇개의 items을 allocate할지 기술하는 `n`

`__get_heap`의 구현은 다음과 같습니다:

```C
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
```

그리고 하기와 같이 사용됩니다:

```C
saved.data = GET_HEAP(u16, saved.x * saved.y);
```

`__get_heap`이 어떻게 동작하는지 이해해 봅시다. 여기서 (`RESET_HEAP()`이후 `_end`와 같아진) `HEAP`이 `a` parameter에 따라 align된 memory의 address가 설정되는 것을 볼 수 있습니다. 이후 `HEAP`의 memory address를 `tmp` variable에 저장하고 `HEAP`을 allocate된 bloack의 끝으로 옮기고 allocate된 memory의 시작 address인 `tmp`를 return합니다.

그리고 마지막 함수는 다음과 같습니다:

```C
static inline bool heap_free(size_t n)
{
	return (int)(heap_end - HEAP) >= (int)n;
}
```

이것은 (이전 [part](linux-bootstrap-2.md)에서 계산한) `heap_end`에서 `HEAP`를 빼고, `n`을 위한 충분한 memory가 있으면 1을 return합니다.

이것이 전부입니다. 이제 heap을 위한 API를 이해했고 video mode를 setup할 수 있습니다.

Set up video mode
--------------------------------------------------------------------------------

이제 video mode 초기화로 바로 갈 수 있습니다. `set_video` 함수에서 `RESET_HEAP()` 호출하는 곳에서 멈췄었습니다. 다음은 [include/uapi/linux/screen_info.h](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/screen_info.h) header file에 정의된 `boot_params.screen_info` structure에 video mode parameters를 저장하는 `store_mode_params`를 호출합니다.

`store_mode_params` 함수를 보시면 `store_cursor_position` 함수를 호출하는 것으로 시작하는 것을 볼 수 있습니다. 함수 이름에서 알 수 있듯이, 이것은 cursor의 정보를 얻어 저장합니다.

무엇보다도 먼저, `store_cursor_position`은 `AH = 0x3`인 `biosregs` type의 2개의 variables를 초기화하고, `0x10` BIOS interrupt를 호출합니다. interrupt가 성공적으로 실행된 이후, row와 column이 `DL`과 `DH` registers로 return됩니다. row와 column은 `boot_params.screen_info` structure의 `orig_x`와 `orig_y` fields에 저장됩니다.

`store_cursor_position`이 실행된 후, `store_video_mode` 함수가 호출됩니다. 이것은 현재 video mode를 얻어서 `boot_params.screen_info.orig_video_mode`에 저장합니다.

이후 `store_mode_params`는 현재 video mode를 확인하고 `video_segment`를 설정합니다. BIOS가 boot sector로 control을 이전한 이후, 다음이 video memory를 위한 address입니다:

```
0xB000:0x0000 	32 Kb 	Monochrome Text Video Memory
0xB800:0x0000 	32 Kb 	Color Text Video Memory
```

만약 현재 video mode가 monochrome mode에서 MDA, HGC, 혹은 VGA라면 `video_segment` variable을 `0xb000`으로 설정하고, color mode라면 `0xb800`으로 설정합니다. video segment의 address를 설정한 후, 다음과 같이 `boot_params.screen_info.orig_video_points`에 font size를 저장해야 합니다:

```C
set_fs(0);
font_size = rdfs16(0x485);
boot_params.screen_info.orig_video_points = font_size;
```

무엇보다도 먼저 `set_fs` 함수에서 `FS` register에 0을 설정합니다. 이전 part에서 [arch/x86/boot/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/boot.h)에 정의된 `set_fs`와 같은 fucntion을 보았습니다. 다음으로 `0x485` address (이 memory는 font size를 얻기 위해 사용됩니다)의 값을 읽어 `boot_params.screen_info.orig_video_points`에 font size를 저장합니다.

```C
x = rdfs16(0x44a);
y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;
```

다음으로 `0x44a` address에서 column을 그리고 `0x484` address에서 row의 값을 얻어 `boot_params.screen_info.orig_video_cols`와 `boot_params.screen_info.orig_video_lines`에 저장합니다. 이후 `store_mode_params`의 실행을 끝냅니다.

다음으로 screen의 content를 heap에 저장하는 `save_screen` 함수를 볼 수 있습니다. 이 함수는 이전 함수에서 얻은 (row, columns과 같은) 모든 data를 수집하여 다음처럼 정의되어 있는 `save_screen` structure에 저장합니다:

```C
static struct saved_screen {
	int x, y;
	int curx, cury;
	u16 *data;
} saved;
```

그리고 다음과 같이 heap이 free space를 가지고 있는지 확인합니다:

```C
if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
		return;
```

공간이 충분하다면 heap에 space를 allocate하고 `saved_screen`을 저장합니다.

다음은 [arch/x86/boot/video-mode.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video-mode.c) source code file에 정의되어 있는 `probe_cards(0)`를 호출합니다. 이것은 모든 video_card를 돌면서 그 card에서 제공되는 mode의 개수를 수집합니다. 여기서 흥미로운 부분은 하기와 같은 loop 입니다:

```C
for (card = video_cards; card < video_cards_end; card++) {
  /* collecting number of modes here */
}
```

하지만 `video_cards`는 어디에서 선언되어 있지 않습니다. 답은 간단합니다: x86 kernel setup code에서 나타나는 모든 video mode는 하기와 같은 정의를 가지고 있습니다:

```C
static __videocard video_vga = {
	.card_name	= "VGA",
	.probe		= vga_probe,
	.set_mode	= vga_set_mode,
};
```

여기서 `__videocard`는 하기와 같은 macro입니다:

```C
#define __videocard struct card_info __attribute__((used,section(".videocards")))
```

즉, `card_info` structure를 의미합니다:

```C
struct card_info {
	const char *card_name;
	int (*set_mode)(struct mode_info *mode);
	int (*probe)(void);
	struct mode_info *modes;
	int nmodes;
	int unsafe;
	u16 xmode_first;
	u16 xmode_n;
};
```

이것은 `.videocards` segment에 저장됩니다. [arch/x86/boot/setup.ld](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) linker script를 살펴보면, 다음과 같은 부분을 찾을 수 있습니다:

```
	.videocards	: {
		video_cards = .;
		*(.videocards)
		video_cards_end = .;
	}
```

이것은 `video_cards`는 그냥 memory address이고 모든 `card_info` structure는 이 segment에 저장되어 있다는 것을 의미합니다. 이것은 모든 `card_info` structure는 `video_cards`와 `video_cards_end` 사이에 위치한다는 것을 의미하기 때문에 loop를 사용하여 그것을 순회할 수 있습니다. `probe_cards`가 실행된 후, `nmodes` (video modes의 수)가 채워진 복수개의 `static __videocard video_vga` structure를 얻게 됩니다.

`prove_cards` 함수가 완료된 후, `set_video` 함수의 main loop로 이동합니다. 여기에는 `set_mode` 함수로 video mode를 설정하거나, kernel command line으로 `vid_mode=ask`를 전달했거나 video mode가 undefine되어 있는 경우 menu를 출력하는 무한 loop가 있습니다.

`set_mode` 함수는 [video-mode.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video-mode.c)에 정의되어 있으며 `mode`라는 하나의 parameter만 받습니다. `mode`는 video mode의 개수로 menu에서 얻거나 kernel setup header의 `setup_video`의 시작 부분에서 얻습니다.

`set_mode` 함수는 `mode`를 확인하고 `raw_set_mode` 함수를 호출합니다. `raw_set_mode`는 선택된 card의 `set_mode` 함수 즉, `card->set_mode(struct mode_info *)`를 호출합니다. 이 함수는 `card_info` structure로부터 access할 수 있습니다. 모든 video mode는 video mode에 따라 값이 채워진 이 structure가 정의되어 있습니다. (예를 들면, `vga`의 경우 `video_vga.set_mode` 함수입니다. 위의 예에서 `vga`의 `card_info` structure를 참조해 주십시요.) `video_vga.set_mode`는 `vga_set_mode`로 vga mode를 확인하고 각각의 함수를 호출합니다:

```C
static int vga_set_mode(struct mode_info *mode)
{
	vga_set_basic_mode();

	force_x = mode->x;
	force_y = mode->y;

	switch (mode->mode) {
	case VIDEO_80x25:
		break;
	case VIDEO_8POINT:
		vga_set_8font();
		break;
	case VIDEO_80x43:
		vga_set_80x43();
		break;
	case VIDEO_80x28:
		vga_set_14font();
		break;
	case VIDEO_80x30:
		vga_set_80x30();
		break;
	case VIDEO_80x34:
		vga_set_80x34();
		break;
	case VIDEO_80x60:
		vga_set_80x60();
		break;
	}
	return 0;
}
```

videoo mode를 설정하는 각 함수는 특정 값을 `AH` register에 설정하고 `0x10` BIOS interrupt를 호출합니다.

video mode를 설정한 후, 그것을 `boot_params.hdr.vid_mode`에 저장합니다.

다음으로 `vesa_store_edid`가 호출됩니다. 이 함수는 단순히 kernel에서 사용하기 위해 [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) (**E**xtended **D**isplay **I**dentification **D**ata) 정보를 저장합니다. 이후 `store_mode_params`가 다시 호출됩니다. 마지막으로 `do_restore`가 설정되어 있으면, screen이 이전 state로 복원됩니다.

이것을 마치면 video mode 설정은 완료됩니다. 이제 protected mode로 천이시킬 수 있습니다.

Last preparation before transition into protected mode
--------------------------------------------------------------------------------

[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)에서 마지막 함수 호출인 `go_to_protected_mode`를 볼 수 있습니다. `나머지 것들을 하고 protected mode를 호출`이라는 comment 처럼, 나머지 것들이 무엇인지와 protected mode로 변경하는 것을 살펴봅시다.

`go_to_protected_mode` 함수는 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pm.c)에 정의되어 있습니다. 이것은 protected mode로 진입하기 전 마지막 준비를 수행하는 몇개의 함수를 포함하고 있습니다. 이것을 무엇을 어떻게 하는지 이해해 봅시다.

`go_to_protected_mode`에서 시작은 `realmode_switch_hook` 함수를 호출하는 것입니다. 이 함수는 real mode switch hook이 존재하고 [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt)가 disable되어 있으면 호출되도록 되어 있습니다. hook에 대해서는 [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)  (**ADVANCED BOOT LOADER HOOKS** 참조)에서 좀 더 자세히 읽어 볼 수 있습니다.

`realmode_switch` hook은 non-maskable interrupt를 disable하는 16-bit real mode far subroutine의 pointer 입니다. `realmode_switch` hook을 확인하고 (나의 경우에는 없었지만), Non-Maskable Interrupts(NMI)가 disable됩니다:

```assembly
asm volatile("cli");
outb(0x80, 0x70);	/* Disable NMI */
io_delay();
```

처음에 interrupt flag (`IF`)를 clear하는 `cli` instruction을 포함하는 inline assembly가 나옵니다. 이후, external interrupt가 disable됩니다. 그 다음 line이 NMI (non-maskable interrupt)를 disable시킵니다.

interrupt는 hardware 혹은 software가 발신하는 CPU에 전달되는 signal입니다. 그러한 signal이 전달되면, CPU는 현재의 instruction sequence를 중단하고, 현재 state를 저장하고 control을 interrupt handler로 옮깁니다. interrupt handler가 해당 작업을 완료한 후, control이 interrupt된 instruction으로 돌아옵니다. Non-maskable interrupts (NMI)는 permission과 무관하게 항상 처리되는 interrupt입니다. 이것은 무시될 수 없으며 non-recoverable hardware errors를 위해 사용됩니다. 지금 당장 interrupt에 대해 상세히 다루지는 않지만 이후 post에서 논의할 것입니다.

code로 돌아가 봅시다. 두번째 line에서 `0x80` (disabled bit)을 `0x70` (CMOOS address register)에 쓰는 것을 볼 수 있습니다. 이후, `io_delay` 함수 호출이 일어납니다. `io_delay`는 다음과 같이 작은 delay를 일으킵니다:

```C
static inline void io_delay(void)
{
	const u16 DELAY_PORT = 0x80;
	asm volatile("outb %%al,%0" : : "dN" (DELAY_PORT));
}
```

`0x80` port로 어떤 byte를 출력하기 위해서는 1 microsecond의 delay가 필요합니다. 그래서 어떤 값 (위에서는 `AL`의 값) 을 `0x80` port에 쓸 수 있습니다. 이 delay 이후 `realmode_switch_hook` 함수가 끝나고 다음 함수로 이동합니다.

다음 함수는 `enable_a20`이며, 이것은 [A20 line](http://en.wikipedia.org/wiki/A20_line)을 enable합니다. 이 함수는 [arch/x86/boot/a20.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/a20.c)에 정의되어 있으며 이것은 몇가지 다른 방법으로 A20 gate를 enable시킵니다. 먼저 `a20_test_short` 함수가 `a20_test` 함수를 통해 A20이 이미 enable되어 있는지를 확인합니다:

```C
static int a20_test(int loops)
{
	int ok = 0;
	int saved, ctr;

	set_fs(0x0000);
	set_gs(0xffff);

	saved = ctr = rdfs32(A20_TEST_ADDR);

        while (loops--) {
		wrfs32(++ctr, A20_TEST_ADDR);
		io_delay();	/* Serialize and make delay constant */
		ok = rdgs32(A20_TEST_ADDR+0x10) ^ ctr;
		if (ok)
			break;
	}

	wrfs32(saved, A20_TEST_ADDR);
	return ok;
}
```

먼저 `FS` register에 `0x0000`을 설정하고 `GS` register에 `0xffff`를 설정합니다. 다음으로 `A20_TEST_ADDR` (`0x200`) address의 값을 읽고 그 값을 변수 `saved`와 `ctr`에 저장합니다.

다음으로 `wrfs32` 함수로 update된 `ctr`의 값을 `fs:A20_TEST_ADDR` 혹은 `fs:0x200`에 쓰고 1ms의 delay를 주고 `GS` register로부터 값을 읽어 `A20_TEST_ADDR+0x10`에 씁니다. `a20` line이 disable되어 있다면 address가 overlap되고, 그렇지 않고 0이 아닌 경우 `a20` line은 이미 enable된 것입니다.

만약 A20이 disable되어 있으면, `a20.c`에서 찾을 수 있는 다른 방법으로 enable을 시도합니다. 예를 들면, `AH=0x2041`로 설정하고 `0x15` BIOS interrupt를 수행할 수 있습니다.

만약 `enable_a20` 함수가 실패로 끝나면, error message를 출력하고 `die` 함수를 호출합니다. 이것은 우리가 살펴본 첫 source code - [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S)에 정의되어 있습니다:

```assembly
die:
	hlt
	jmp	die
	.size	die, .-die
```

A20 gate가 성공적으로 enable된 후, `reset_coprocessor` 함수가 호출됩니다:

```C
outb(0, 0xf0);
outb(0, 0xf1);
```

이 함수는 `0`을 `0xf0`에 써서 Math Coprocessor를 clear하고 `0`을 `0xf1`에 써서 reset시킵니다.

이후, `mask_all_interrupts` 함수가 호출됩니다:

```C
outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
```

이것은 primary PIC (Programmable Interrupt Controller) 상의 IRQ2를 제외한 secondary PIC와 primary PIC의 모든 interrupt를 mask합니다.

이러한 모든 준비 후, 실제 protected mode로의 천이를 시작합니다.

Set up the Interrupt Descriptor Table
--------------------------------------------------------------------------------

이제 `setup_idt` 함수에서 Interrupt Descriptor Table (IDT)를 설정합니다:

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

이것은 (interrupt handler 등을 기술하는) Interrupt Descriptor Table을 설정합니다. 지금은 IDT는 설정되어 있지 않지만 (이후 살펴볼 것입니다), `lidtl` instruction으로 IDT를 load합니다. `null_idt`는 IDT의 address와 size를 포함하고 있으나 현재 그냥 0으로 되어 있습니다. `null_idt`는 `gdt_ptr` structure로 하기와 같이 정의되어 있습니다:

```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

이것은 IDT의 16-bit length (`len`)과 IDT를 가르키는 32-bit pointer로 구성되어 있습니다. (IDT와 interrupt에 대한 자세한 내용은 다음 post에서 살펴볼 것입니다) `__attribute__((packed))`는 `gdt_ptr`의 크기가 요구되어지는 최소 크기가 되는 것을 의미합니다. 그렇기 때문에 여기서 `gdt_ptr`의 크기는 6 bytes 혹은 48 bits가 됩니다. (다음에 `gdt_ptr`을 가르키는 pointer가 `GDTR` register에 load되는데, 이전 post에서 언급한 것처럼 이것은 48 bits 입니다)

Set up Global Descriptor Table
--------------------------------------------------------------------------------

다음은 Global Descriptor Table (GDT)의 설정입니다. GDT를 설정하는 `setup_gdt` 함수를 볼 수 있습니다. ([Kernel booting process. Part 2.](linux-bootstrap-2.md#protected-mode) post에서 살펴보았습니다) 이 함수에서는 3개의 segment의 선언을 포함하고 있는 `boot_gdt` array를 선언합니다:

```C
static const u64 boot_gdt[] __attribute__((aligned(16))) = {
	[GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
	[GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
};
```

이것들은 code, data 그리고 TSS (Task State Segment)를 위한 segment입니다. 여기서 Task State Segment는 사용되지 않지만, comment에서 볼 수 있는 것처럼 Intel VT를 만족하기 위해 추가되어 있습니다. (관련 commit은 [여기](https://github.com/torvalds/linux/commit/88089519f302f1296b4739be45699f06f728ec31)에 있습니다) `boot_gdt`를 살펴봅시다. 먼저 `__attribute__((aligned(16)))`의 attribute에 주목해 주십시요. 이것은 이 structure는 16 bytes에 align된다는 것을 의미합니다.

간단한 예제를 살펴봅시다:

```C
#include <stdio.h>

struct aligned {
	int a;
}__attribute__((aligned(16)));

struct nonaligned {
	int b;
};

int main(void)
{
	struct aligned    a;
	struct nonaligned na;

	printf("Not aligned - %zu \n", sizeof(na));
	printf("Aligned - %zu \n", sizeof(a));

	return 0;
}
```

기술적으로 `int` field를 포함하는 structure는 4 bytes가 되어야 하지만, `aligned` structure는 memory 상에서 16 bytes를 필요로 합니다:

```
$ gcc test.c -o test && test
Not aligned - 4
Aligned - 16
```

`GDT_ENTRY_BOOT_CS`는 index - 여기서는 2 - 를 가지고 있고, `GDT_ENTRY_BOOT_DS`는 `GDT_ENTRY_BOOT_CS + 1`입니다. 첫번째는 mandatory null descriptor (index - 0)이고 두번째는 사용되지 않기 때문에 (index - 1), 이것은 2부터 시작합니다.

`GDT_ENTRY`는 flags, base, limit를 받아 GDT entry를 생성하는 macro입니다. 예를 들면, code segment entry를 살펴봅시다. `GDT_ENTRY`는 다음과 같은 값을 받습니다:

* base  - 0
* limit - 0xfffff
* flags - 0xc09b

이것은 무엇을 의미할까요? segment의 base address는 0이고, limit (segment의 크기)는 `0xfffff` (1MB) 라는 것입니다. flags를 살펴봅시다. `0xc09b`는 2진수로 다음과 같습니다:

```
1100 0000 1001 1011
```

각각의 bit의 의미를 이해해 봅시다. 좌측에서 우측으로 모든 bit을 살펴보겠습니다:

* 1    - (G) granularity bit
* 1    - (D) 0이면 16-bit segment; 1이면 32-bit segment
* 0    - (L) 1이면 64-bit mode로 실행
* 0    - (AVL) system software로 사용
* 0000 - 4-bit length 19:16 bits in the descriptor
* 1    - (P) memory에서 segment의 presence
* 00   - (DPL) - privilege level, 0은 가장 높은 privilege
* 1    - (S) code 혹은 data segment, system segment는 아님
* 101  - segment type execute/read/
* 1    - accessed bit

이전 [post](linux-bootstrap-2.md) 혹은 [Intel® 64 and IA-32 Architectures Software Developer's Manuals 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)에서 각 bit에 대해 더 자세한 내용을 읽을 수 있습니다.

이후 하기와 같이 GDT의 크기를 얻습니다:

```C
gdt.len = sizeof(boot_gdt)-1;
```

`boot_gdt`의 크기를 얻고 1을 뺍니다. (GDT에서 마지막 valid address)

다음으로 하기와 같이 GDT의 pointer를 얻습니다:

```C
gdt.ptr = (u32)&boot_gdt + (ds() << 4);
```

여기서 단순히 `boot_gdt`의 address를 얻고 data segment의 address를 왼쪽으로 4 bits shift시킨 값을 더합니다. (현재 real mode라는 것을 기억하세요)

마지막으로 GDT를 GDTR register에 load하기 위해 `lgdtl` instruction을 실행합니다:

```C
asm volatile("lgdtl %0" : : "m" (gdt));
```

Actual transition into protected mode
--------------------------------------------------------------------------------

이것이 `go_to_protected_mode` 함수의 마지막입니다. IDT와 GDT를 load했고, interrupt를 disable했습니다. 이제 CPU를 protected mode로 전환할 수 있습니다. 마지막 step은 2개의 parameter로 `protected_mode_jump` 함수를 호출하는 것입니다:

```C
protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));
```

이것은 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S)에 정의되어 있습니다.

이것은 다음의 2개의 parameter를 받습니다:

* protected mode entry point의 address
* `boot_params`의 address

`protected_mode_jump`의 내부를 살펴봅시다. 위에서 쓴 것처럼 `arch/x86/boot/pmjump.S`에서 찾아볼 수 있습니다. 첫번째 parameter는 `eax` register에 있고 두번째 것은 `edx` register에 있을 것입니다.

먼저 `boot_params`의 address를 `esi` register에 넣고 code segment register `cs`의 address를 `bx`에 넣습니다. 이후 `bx`를 4 bits shift시키고 `2` (이것은 `(cs << 4) + in_pm32`로 32-bit mode로 천이한 후 jump할 physical address 입니다)라고 label된 memory location에 더하고 label `1`로 jump합니다. 이후 `2`로 label되어 있는 `in_pm32`는 `(cs << 4) + in_pm32`로 overwrite됩니다.

다음으로 data segment와 task state segment를 각각 `cx`와 `di` register에 넣습니다:

```assembly
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

위에서 보실수 있는 것처럼 `GDT_ENTRY_BOOT_CS`는 2개의 index를 가지고 있으며 각각의 GDT entry는 8 bytes이기 때문에 `CS`는 `2 * 8 = 16`, `__BOOT_DS`는 24가 됩니다.

다음으로 `CR0` control register의 `PE` (Protection Enable) bit을 설정합니다:

```assembly
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl
movl	%edx, %cr0
```

그리고 protected mode로 long jump를 합니다:

```assembly
	.byte	0x66, 0xea
2:	.long	in_pm32
	.word	__BOOT_CS
```

여기서:

* `0x66`는 16-bit와 32-bit code의 mix를 허용하는 operand-size prefix이고
* `0xea`는 jump opcode이고
* `in_pm32`는 protected mode에서 segment offset으로, 그 값은 real mode에서 얻은 `(cs << 4) + in_pm32`이고
* `__BOOT_CS`는 jump할 code segment입니다.

이제 protected mode로 진입하였습니다:

```assembly
.code32
.section ".text32","ax"
```

protected mode에서 수행하는 첫번째 step을 살펴봅시다. 먼저 data segment를 다음과 같이 설정합니다:

```assembly
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
```

`$__BOOT_DS`를 `cx` register에 저장한 것을 기억하실 수 있을 겁니다. `cs`를 제외한 모든 segment register를 이 값으로 채웁니다. (`cs`는 이미 `__BOOT_CS`입니다)

그리고 debugging을 위해 유효한 stack을 설정합니다:

```assembly
addl	%ebx, %esp
```

32-bit entry point로 jump하기 전 마지막 step은 general purpose register를 clear하는 것입니다:

```assembly
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

그리고 마침내 32-bit entry point로 jump합니다:

```
jmpl	*%eax
```

`eax`가 32-bit entry의 address를 가지고 있다는 것을 기억하십시요. (`protected_mode_jump`의 첫번째 parameter로 전달했습니다)

이것이 전부 입니다. protected mode로 진입했고 entry point에서 멈췄습니다. 다음 part에서 다음에 무엇이 일어날지 살펴볼 것입니다.

Conclusion
--------------------------------------------------------------------------------

이것이 linux kernel insides의 세번째 part의 끝입니다. 다음 part에서는 protected mode에서 수행하는 첫번째 step과 [long mode](http://en.wikipedia.org/wiki/Long_mode)로의 천이를 살펴볼 것입니다.

질문이나 제안이 있으시면 comment나 [twitter](https://twitter.com/0xAX)로 연락주십시요.

**영어는 저의 모국어가 아닙니다. 불편한 점이 있으셨다면 먼저 사죄드립니다. 만약 잘못된 점을 발견하시면 [linux-insides](https://github.com/0xAX/linux-internals)로 PR을 보내주십시요.**

Links
--------------------------------------------------------------------------------

* [VGA](http://en.wikipedia.org/wiki/Video_Graphics_Array)
* [VESA BIOS Extensions](http://en.wikipedia.org/wiki/VESA_BIOS_Extensions)
* [Data structure alignment](http://en.wikipedia.org/wiki/Data_structure_alignment)
* [Non-maskable interrupt](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [A20](http://en.wikipedia.org/wiki/A20_line)
* [GCC designated inits](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Designated-Inits.html)
* [GCC type attributes](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html)
* [Previous part](linux-bootstrap-2.md)
