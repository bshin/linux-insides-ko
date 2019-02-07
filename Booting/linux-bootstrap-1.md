Kernel booting process. Part 1.
================================================================================

From the bootloader to the kernel
--------------------------------------------------------------------------------

만약 저의 이전 [blog posts](https://0xax.github.io/categories/assembler/)를 읽으신 분이라면, 제가 얼마간 low-level programming에 관여하기 시작했다는 것을 아실 수 있을 겁니다. 저는 `x86_64` Linux를 위한 assembly programming에 대한 post를 썼고, 이와 동시에 Linux kernel source code로 뛰어 들었습니다.

저는 low-level이 어떻게 동작하는지, 내 computer에서 programs이 어떻게 실행되는지, memory에 그것들이 어떻게 위치되는지, kernel이 processes와 memory를 어떻게 관리하는지, low level에서 network stack이 어떻게 동작하는지 등에 관심이 많습니다. 그래서 저는 **x86_64** architecture의 Linux kernel에 대한 post의 series를 쓰기로 결심했습니다.

저는 professional kernel hacker가 아니며 일로써 kernel의 code를 작성하지 않습니다. 이것은 그저 취미입니다. 저는 그저 low-level stuff가 좋고, 그것들이 어떻게 동작하는지 보는 것에 관심이 있습니다. 만약 혼동되는 것이 있거나 질문 / 언급할 만한 것이 있으면 Twitter [0xAX](https://twitter.com/0xAX) 혹은 [email](anotherworldofworld@gmail.com)로 연락을 주시거나, [issue](https://github.com/0xAX/linux-insides/issues/new)를 생성해 주십시요. 감사드립니다.

모든 posts는 [github repo](https://github.com/0xAX/linux-insides)에서도 access 가능하며, 만약 영어 혹은 post의 content에서 잘못된 것을 찾으시면, pull request를 보내 주십시요.

*이것은 official documentation이 아니며, 단지 지식을 공유하는 것입니다.*

**Required knowledge**

* Understanding C code
* Understanding assembly code (AT&T syntax)

어째됐든, 만약 당신이 이러한 tools을 배우기 시작했다면, 이번과 이어지는 posts에서 이러한 부분을 설명할 것 입니다. 자, 이것이 간단한 소개의 끝입니다. 이제 우리는 Linux kernel과 low-level stuff으로 뛰어 들 수 있습니다.

저는 `3.18` Linux kernel과 같이 이 책을 쓰기 시작했습니다. 많은 것들이 바뀌었을 수 있습니다. 변경점이 있다면 저는 posts를 그에 맞게 update할 것입니다.

The Magical Power Button, What happens next?
--------------------------------------------------------------------------------

이것이 Linux kernel에 대한 posts의 series이지만, 적어도 이 문단에서는 kernel code로부터 바로 시작하지는 않을 겁니다. laptop 혹은 desktop computer의 magical power button을 누른 직후, 그것은 동작하기 시작합니다. motherboard는 [power supply](https://en.wikipedia.org/wiki/Power_supply) device로 signal을 보냅니다. power supply는 이 signal을 받고 적절한 양의 전력을 computer에 제공합니다. motherboard가 [power good signal](https://en.wikipedia.org/wiki/Power_good_signal)을 받으면, CPU를 시작시킵니다. CPU는 registers에 남아 있는 모든 data를 reset시키고 각각의 predefine된 값을 설정합니다.

[80386](https://en.wikipedia.org/wiki/Intel_80386) CPU와 그 이후 CPU는 computer reset후 CPU registers에 다음과 같은 predefine된 값을 정의합니다:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

processor는 [real mode](https://en.wikipedia.org/wiki/Real_mode)에서 동작하기 시작합니다. 잠시 이 mode에서의 [memory segmentation](https://en.wikipedia.org/wiki/Memory_segmentation)에 대해서 이해해 봅시다. real mode는 [8086](https://en.wikipedia.org/wiki/Intel_8086) CPU 부터 현재의 Intel 64-bit CPU까지 모든 x86-compatible processors에서 지원하고 있습니다. `8086` processor는 20-bit의 address bus를 가지고 있으며, 이는 `0-0xFFFFF` 혹은 `1 megabyte`의 address space에서 동작하는 것을 의미합니다. 그러나 그것은 오직 `16-bit` registers만 가지고 있어서, 최대 address는 `2^16 - 1` 혹은 `0xffff` (64 kilobytes)가 됩니다.

[Memory segmentation](https://en.wikipedia.org/wiki/Memory_segmentation)는 모든 address space를 사용 가능하도록 하기 위해 사용됩니다. 모든 memory는 작은, `65536` bytes (64 KB)의 고정된 크기의 segments로 나뉘어 집니다. 16-bit registers로는 `64 KB` 상위의 memory를 address할 수 없기 때문에 대안이 고안되었습니다.

address는 두 부분으로 구성됩니다: base address를 가지고 있는 segment selector와 이 base address로부터의 offset 입니다.real mode에서 segment selector의 base address는 `Segment Selector * 16` 입니다. 그래서 memory에서 physical address를 얻기 위해서는 segment selector 부분에 `16`을 곱하고 여기에 offset을 더해야 합니다:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

예를 들면, 만약 `CS:IP`가 `0x2000:0x0010`라면, 대응하는 physical address는 다음과 같습니다:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

하지만, 만약 가장 큰 segment selector와 offset을 선택하면, `0xffff:0xffff`, 대응하는 physical address는 다음과 같습니다:

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

이것은 첫번째 megabyte에서 `65520` bytes 넘은 것입니다. real mode에서는 오직 1 megabyte만 access 가능하기 때문에, `0x10ffef`는 [A20 line](https://en.wikipedia.org/wiki/A20_line)이 disable되어 `0x00ffef`가 됩니다.

자, 이제 real mode와 이 mode에서의 memory addressing에 대해 조금 알았습니다. 다시 reset후 register 값 논의로 돌아갑시다.

`CS` register는 두 부분으로 구성됩니다: visible segment selector와 hidden base address입니다. base address는 일반적으로 segment selector 값에 16을 곱한 형태이지만, hardware reset시에는 CS register의 segment selector는 `0xf0000`으로 load되고 base address는 `0xffff0000`로 load됩니다; processor는 `CS`가 바뀔 때까지 이 special base address를 사용하게 됩니다.

시작 address는 base address에 EIP register의 값을 더한 형태가 됩니다:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

`0xfffffff0`는 4GB에서 16 bytes 아래 입니다. 이 point는 [reset vector](https://en.wikipedia.org/wiki/Reset_vector)라고 불리웁니다. 이것은 reset 후 CPU가 수행할 첫번째 instruction을 찾는 memory location입니다. 이것은 일반적으로 BIOS의 entry point를 가르키는 [jump](https://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`) instruction을 포함합니다. 예를 들면, [coreboot](https://www.coreboot.org/)의 source code (`src/cpu/x86/16bit/reset16.inc`)에서 다음과 같은 부분을 볼 수 있습니다.

```assembly
    .section ".reset", "ax", %progbits
    .code16
.globl	_start
_start:
    .byte  0xe9
    .int   _start16bit - ( . + 2 )
    ...
```

여기서 `jmp` instruction [opcode](http://ref.x86asm.net/coder32.html#xE9)인 `0xe9`를 볼 수 있으며, 그것의 destination address는 `_start16bit - ( . + 2)`입니다.

`reset` section은 `16` bytes이고 `0xfffffff0` address부터 시작하는 것을 볼 수 있습니다. (`src/cpu/x86/16bit/reset16.ld`):

```
SECTIONS {
    /* Trigger an error if I have an unuseable start address */
    _bogus = ASSERT(_start16bit >= 0xffff0000, "_start16bit too low. Please report.");
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset);
        . = 15;
        BYTE(0x00);
    }
}
```

이제 BIOS가 시작합니다; initialization과 hardware의 check이후 BIOS는 bootable device를 찾아야 합니다. 어떤 device부터 boot를 시도할지 control하는 boot 순서는 BIOS의 configuration에 저장되어 있습니다. hard drive로 부터 boot를 시도할 때, BIOS는 boot sector를 찾습니다. [MBR partition layout](https://en.wikipedia.org/wiki/Master_boot_record)으로 partition되어 있는 hard drives에서는 sector가 `512` bytes이고, boot sector가 첫번째 sector의 첫번째 `446` bytes에 저장되어 있습니다. 첫번째 sector의 마지막 두 bytes는 `0x55`와 `0xaa`이며, 이는 BIOS에게 이 device가 bootable임을 나타냅니다.

예를 들면:

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

Build 하여 하기와 같이 수행합니다:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

이것은 [QEMU](http://qemu.org)가 방금 build한 `boot` binary를 disk image로 사용하도록 합니다. boot sector의 요구 사항 (~~시작 위치는`0x7c00`으로 set되어 있고~~ magic sequence로 끝남)을 모두 만족하는 상단의 assembly로 생성된 binary이기 때문에, QEMU는 이 binary를 disk image의 master boot record (MBR)로 취급합니다.

하기와 같은 출력을 볼 수 있습니다:

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

이 예에서, code가 `16-bit` real mode에서 수행되며 memory의 `0x7c00`에서 시작된다는 것을 알 수 있습니다. 시작 이후, [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt가 호출되고, 이것은 `!` symbol을 print 합니다; 나머지 `510` bytes는 zero로 채워지고 magic bytes인 `0xaa`와 `0x55`로 끝납니다.

`objdump` utility를 이용하여 binary dump하면 다음과 같은 결과를 볼 수 있습니다:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

실제 boot sector는 한 무더기의 0과 느낌표 대신에 boot process를 계속하기 위한 code와 partition table을 가지고 있습니다 :) 이 시점부터 BIOS는 bootloader로 control을 넘깁니다.

**NOTE**: 위에서 설명한 것처럼, CPU는 real mode에 있습니다; real mode에서, memory에서 physical address의 계산은 다음과 같이 수행됩니다:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

위에서 설명한 것과 같이, 16-bit general purpose registers만 있고; 이것의 최대 값은 `0xffff`이기 때문에, 최대값을 선택하면 그 결과는 다음과 같습니다:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

여기서 `0x10ffef`는 `1MB + 64KB - 16b`와 같습니다. 반면, [8086](https://en.wikipedia.org/wiki/Intel_8086) processor (real mode를 채용한 첫번째 processor)는 20-bit address line을 가지고 있습니다. `2^20 = 1048576`는 1MB, 이것은 실제 가용한 memory는 1MB라는 것을 의미합니다.

일반적으로, real mode의 memory map은 다음과 같습니다:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS
```

이 post의 시작으로 address `0xFFFFF` (1MB) 보다 훨씬 큰 `0xFFFFFFF0`에 위치된 CPU가 수행하는 첫번째 instruction에 대해 썼습니다. 어떻게 CPU는 real mode에서 이 address를 access할 수 있을까? 답은 [coreboot](https://www.coreboot.org/Developer_Manual/Memory_map) documentation에 있습니다:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

최초 수행시에 BIOS는 RAM에 있는 것이 아니라 ROM에 있는 것입니다.

Bootloader
--------------------------------------------------------------------------------

There are a number of bootloaders that can boot Linux, such as [GRUB 2](https://www.gnu.org/software/grub/) and [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). The Linux kernel has a [Boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt) which specifies the requirements for a bootloader to implement Linux support. This example will describe GRUB 2.

Continuing from before, now that the `BIOS` has chosen a boot device and transferred control to the boot sector code, execution starts from [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). This code is very simple, due to the limited amount of space available, and contains a pointer which is used to jump to the location of GRUB 2's core image. The core image begins with [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), which is usually stored immediately after the first sector in the unused space before the first partition. The above code loads the rest of the core image, which contains GRUB 2's kernel and drivers for handling filesystems, into memory. After loading the rest of the core image, it executes the [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) function.

The `grub_main` function initializes the console, gets the base address for modules, sets the root device, loads/parses the grub configuration file, loads modules, etc. At the end of execution, the `grub_main` function moves grub to normal mode. The `grub_normal_execute` function (from the `grub-core/normal/main.c` source code file) completes the final preparations and shows a menu to select an operating system. When we select one of the grub menu entries, the `grub_menu_execute_entry` function runs, executing the grub `boot` command and booting the selected operating system.

As we can read in the kernel boot protocol, the bootloader must read and fill some fields of the kernel setup header, which starts at the `0x01f1` offset from the kernel setup code. You may look at the boot [linker script](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) to confirm the value of this offset. The kernel header [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) starts from:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

The bootloader must fill this and the rest of the headers (which are only marked as being type `write` in the Linux boot protocol, such as in [this example](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354)) with values which it has either received from the command line or calculated during boot. (We will not go over full descriptions and explanations for all fields of the kernel setup header now, but we shall do so when we discuss how the kernel uses them; you can find a description of all fields in the [boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156).)

As we can see in the kernel boot protocol, the memory will be mapped as follows after loading the kernel:

```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

So, when the bootloader transfers control to the kernel, it starts at:

```
X + sizeof(KernelBootSector) + 1
```

where `X` is the address of the kernel boot sector being loaded. In my case, `X` is `0x10000`, as we can see in a memory dump:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

The bootloader has now loaded the Linux kernel into memory, filled the header fields, and then jumped to the corresponding memory address. We can now move directly to the kernel setup code.

The Beginning of the Kernel Setup Stage
--------------------------------------------------------------------------------

Finally, we are in the kernel! Technically, the kernel hasn't run yet; first, the kernel setup part must configure stuff such as the decompressor and some memory management related things, to name a few. After all these things are done, the kernel setup part will decompress the actual kernel and jump to it. Execution of the setup part starts from [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) at the [_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292) symbol.

It may looks a little bit strange at first sight, as there are several instructions before it. A long time ago, the Linux kernel used to have its own bootloader. Now, however, if you run, for example,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

then you will see:

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

Actually, the file `header.S` starts with the magic number [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (see image above), the error message that displays and, following that, the [PE](https://en.wikipedia.org/wiki/Portable_Executable) header:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

It needs this to load an operating system with [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) support. We won't be looking into its inner workings right now and will cover it in upcoming chapters.

The actual kernel setup entry point is:

```assembly
// header.S line 292
.globl _start
_start:
```

The bootloader (grub2 and others) knows about this point (at an offset of `0x200` from `MZ`) and makes a jump directly to it, despite the fact that `header.S` starts from the `.bstext` section, which prints an error message:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

The kernel setup entry point is:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

Here we can see a `jmp` instruction opcode (`0xeb`) that jumps to the `start_of_setup-1f` point. In `Nf` notation, `2f`, for example, refers to the local label `2:`; in our case, it is the label `1` that is present right after the jump, and it contains the rest of the setup [header](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156). Right after the setup header, we see the `.entrytext` section, which starts at the `start_of_setup` label.

This is the first code that actually runs (aside from the previous jump instructions, of course). After the kernel setup part receives control from the bootloader, the first `jmp` instruction is located at the `0x200` offset from the start of the kernel real mode, i.e., after the first 512 bytes. This can be seen in both the Linux kernel boot protocol and the grub2 source code:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

In my case, the kernel is loaded at `0x10000` address. This means that segment registers will have the following values after kernel setup starts:

```
gs = fs = es = ds = ss = 0x10000
cs = 0x10200
```

After the jump to `start_of_setup`, the kernel needs to do the following:

* Make sure that all segment register values are equal
* Set up a correct stack, if needed
* Set up [bss](https://en.wikipedia.org/wiki/.bss)
* Jump to the C code in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)

Let's look at the implementation.

Aligning the Segment Registers 
--------------------------------------------------------------------------------

First of all, the kernel ensures that the `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, `grub2` loads kernel setup code at address `0x10000` by default and `cs` at `0x10200` because execution doesn't start from the start of file, but from the jump here:

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

which is at a `512` byte offset from [4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46). We also need to align `cs` from `0x10200` to `0x10000`, as well as all other segment registers. After that, we set up the stack:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

which pushes the value of `ds` to the stack, followed by the address of the [6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602) label and executes the `lretw` instruction. When the `lretw` instruction is called, it loads the address of label `6` into the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and loads `cs` with the value of `ds`. Afterward, `ds` and `cs` will have the same values.

Stack Setup
--------------------------------------------------------------------------------

Almost all of the setup code is in preparation for the C language environment in real mode. The next [step](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575) is checking the `ss` register value and making a correct stack if `ss` is wrong:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

This can lead to 3 different scenarios:

* `ss` has a valid value `0x1000` (as do all the other segment registers beside `cs`)
* `ss` is invalid and the `CAN_USE_HEAP` flag is set     (see below)
* `ss` is invalid and the `CAN_USE_HEAP` flag is not set (see below)

Let's look at all three of these scenarios in turn:

* `ss` has a correct address (`0x1000`). In this case, we go to label [2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Here we set the alignment of `dx` (which contains the value of `sp` as given by the bootloader) to `4` bytes and a check for whether or not it is zero. If it is zero, we put `0xfffc` (4 byte aligned address before the maximum segment size of 64 KB) in `dx`. If it is not zero, we continue to use the value of `sp` given by the bootloader (0xf7f4 in my case). After this, we put the value of `ax` into `ss`, which stores the correct segment address of `0x1000` and sets up a correct `sp`. We now have a correct stack:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* In the second scenario, (`ss` != `ds`). First, we put the value of [_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) (the address of the end of the setup code) into `dx` and check the `loadflags` header field using the `testb` instruction to see whether we can use the heap. [loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) is a bitmask header which is defined as:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

and, as we can read in the boot protocol:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

If the `CAN_USE_HEAP` bit is set, we put `heap_end_ptr` into `dx` (which points to `_end`) and add `STACK_SIZE` (minimum stack size, `1024` bytes) to it. After this, if `dx` is not carried (it will not be carried, `dx = _end + 1024`), jump to label `2` (as in the previous case) and make a correct stack.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* When `CAN_USE_HEAP` is not set, we just use a minimal stack from `_end` to `_end + STACK_SIZE`:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSS Setup
--------------------------------------------------------------------------------

The last two steps that need to happen before we can jump to the main C code are setting up the [BSS](https://en.wikipedia.org/wiki/.bss) area and checking the "magic" signature. First, signature checking:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

This simply compares the [setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) with the magic number `0x5a5aaa55`. If they are not equal, a fatal error is reported.

If the magic number matches, knowing we have a set of correct segment registers and a stack, we only need to set up the BSS section before jumping into the C code.

The BSS section is used to store statically allocated, uninitialized data. Linux carefully ensures this area of memory is first zeroed using the following code:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

First, the [__bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) address is moved into `di`. Next, the `_end + 3` address (+3 - aligns to 4 bytes) is moved into `cx`. The `eax` register is cleared (using a `xor` instruction), and the bss section size (`cx`-`di`) is calculated and put into `cx`. Then, `cx` is divided by four (the size of a 'word'), and the `stosl` instruction is used repeatedly, storing the value of `eax` (zero) into the address pointed to by `di`, automatically increasing `di` by four, repeating until `cx` reaches zero). The net effect of this code is that zeros are written through all words in memory from `__bss_start` to `_end`:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Jump to main
--------------------------------------------------------------------------------

That's all - we have the stack and BSS, so we can jump to the `main()` C function:

```assembly
    calll main
```

The `main()` function is located in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). You can read about what this does in the next part.

Conclusion
--------------------------------------------------------------------------------

This is the end of the first part about Linux kernel insides. If you have questions or suggestions, ping me on Twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com), or just create an [issue](https://github.com/0xAX/linux-internals/issues/new). In the next part, we will see the first C code that executes in the Linux kernel setup, the implementation of memory routines such as `memset`, `memcpy`, `earlyprintk`, early console implementation and initialization, and much more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](https://en.wikipedia.org/wiki/Intel_8086)
  * [80386](https://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](https://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](https://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [coreboot developer manual](https://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](https://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](https://en.wikipedia.org/wiki/Power_good_signal)
