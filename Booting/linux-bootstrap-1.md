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

linux를 boot시킬 수 있는 [GRUB 2](https://www.gnu.org/software/grub/)이나 [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project) 같은 bootloader는 다수 존재합니다. linux kernel은 linux를 지원하는 bootloader 구현을 위한 요구 사항을 기술한 [Boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt)를 가지고 있습니다. 이 예는 GRUB 2에서 설명합니다.

이전에 계속하여, 이제 `BIOS`는 boot device를 선택하고 control을 [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD)로부터 실행을 시작하는 boot sector code로 넘깁니다. 이 code는 가용한 space의 제약으로 매우 간단한데, GRUB 2의 core image의 location으로 jump하기 위한 pointer를 가지고 있습니다. [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD)로부터 시작되는 core image는 일반적으로 첫번째 sector 직후에 첫번째 partition 전의 사용되지 않는 sector에 저장됩니다. 위의 code는 GRUB 2의 kernel과 filesystem을 처리하기 위한 driver를 포함하는 core image의 나머지를 memory로 load합니다. core image의 나머지를 load한 이후, [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) function을 수행합니다.

`grub_main`은 console을 초기화하고, modules의 base address을 얻고, root device를 설정하고, grub configuration file을 load/parse하고, modules을 load하는 등의 일을 합니다. 수행의 마지막에, `grub_main` function은 grub을 normal mode로 천이시킵니다. `grub_normal_execute` function (source code file의 `grub-core/normal/main.c`)은 마지막 준비를 마치고 operating system 선택 menu를 보여줍니다. grub menu entries중 하나를 선택하면, `grub_menu_execute_entry` function가 실행되어, grub `boot` command를 수행하고 선택된 operating system을 boot합니다.

kernel boot protocol에서 읽을 수 있는 것처럼, bootloader는 kernel setup code로부터 offset `0x01f1`에서 시작되는 kernel setup header의 일부 fields를 읽고 채워줘야 합니다. 이 offset의 값을 확인하기 위해 boot [linker script](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)를 봐도 됩니다. kernel header [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S)는 다음과 같이 시작됩니다:

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

bootloader는 이것과 headers의 나머지([이 예](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354)와 같이 linux boot protocol에서 type이 `write`로 표기되어 있는 부분만)를 command line에서 받거나 boot동안 계산한 값으로 채워줘야 합니다. (지금 kernel setup header의 모든 fields에 대한 전체 기술과 설명은 하지 않을 것입니다. 하지만, kernel이 그것들을 어떻게 사용하는지 설명할 때 그렇게 할 것입니다; 전체 fields의 설명은 [boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156)에서 볼 수 있습니다.)

kernel boot protocl에서 볼 수 있는 것 처럼 kernel load후 memory는 다음과 같이 map되어 있습니다:

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

bootloader가 kernel로 control을 넘길 때, 하기에서 시작합니다:

```
X + sizeof(KernelBootSector) + 1
```

여기서 `X`는 load된 kernel boot sector의 address입니다. memory dump에서 볼 수 있는 것처럼 `X`는 `0x10000`입니다:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

이제 bootloader가 linux kernel을 memory에 load하고, header fields를 채우고, 대응하는 memory address로 jump했습니다. 이제 kernel setup code로 바로 이동할 수 있습니다.

The Beginning of the Kernel Setup Stage
--------------------------------------------------------------------------------

드디어, kernel에 들어왔습니다! 엄밀히 말하면, kernel은 하직 실행되지 않았습니다; 먼저 kernel setup part가 decompressor와 memory management 관련된 것들을 설정해야 합니다. 이러한 것들이 모두 끝난 후에, kernel setup part는 실제 kernel을 decompress하고 kernel로 jump 합니다. setup part의 실행은 [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S)의 [_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292) symbol에서 시작합니다.

처음 볼 때는 조금 이상해 보일 수도 있습니다만, 몇개의 instructions이 그것 앞에 있습니다. 오래 전에는 linux kernel은 자신의 bootloader를 가지고 있었습니다. 하지만 지금 하기 예와 같이 실행해 보면,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

다음과 같은 출력을 볼 수 있습니다:

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

실제, `header.S` file은 magic number [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (위 image를 참조)로 시작하고, error message를 출력하고, 이어서 [PE](https://en.wikipedia.org/wiki/Portable_Executable) header가 나옵니다:

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

[UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)를 지원하는 operating system을 load하기 위해 이러한 것이 필요합니다. 지금 당장 내부 동작을 보지는 않지만 이어지는 chapters에서 다룰 것입니다.

실제 kernel setup entry point는 다음과 같습니다:

```assembly
// header.S line 292
.globl _start
_start:
```

bootloader (grub2와 다른 것들)는 이 point (`MZ`으로 부터 `0x200`의 offset을 가짐)를 알고 있고, `header.S`가 error message를 출력하는 `.bstext` section으로 시작함에도 불구하고, 적절한 위치로 jump합니다:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

kernel setup entry point는 다음과 같습니다:

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

여기서 `start_of_setup-1f`로 jump하는 `jmp` instruction opcode (`0xeb`)를 볼 수 있습니다. `Nf` notation은 예를 들면 `2f`는 local label `2:`를 참조합니다; 위의 경우, jump 직후에 나오는 label `1`을 참조하고, 그것은 setup [header](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156)의 나머지 부분을 포함합니다. setup header 직후 `start_of_setup` label로 시작하는 `.entrytext` section을 볼 수 있습니다.

이것이 실제 실행되는 최초의 code (물론, 이전 jump instructions을 제외하고) 입니다. bootloader로부터 kernel setup part가 control을 받은 후, 첫번째 `jmp` instruction은 kernel real mode의 시작으로부터 `0x200` 즉, 첫 512 bytes offset을 가르킵니다. 이것은 linux kernel boot protocol과 grub2 source code에서 볼 수 있습니다:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```
위의 예에서, kernel은 `0x10000` address에 load되었습니다. 이것은 kernel setup이 시작된 후 segment registers가 다음과 같은 값을 가진다는 것을 의미합니다.

```
gs = fs = es = ds = ss = 0x10000
cs = 0x10200
```

`start_of_setup`로 jump한 후, kernel은 다음과 같은 합니다:

* 모든 segment register 값을 동일하게 설정
* 필요시 적절한 stack을 setup
* [bss](https://en.wikipedia.org/wiki/.bss)를 setup
* [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)의 C code로 jump

구현을 살펴봅시다.

Aligning the Segment Registers 
--------------------------------------------------------------------------------

무엇보다도 먼저, kernel은 `ds`와 `es` segment registers가 같은 address를 가르키도록 합니다. 다음으로 `cld` instruction을 사용하여 direction flag를 clear합니다:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

위에서 기술한것 처럼, `grub2`는 default로 kernel setup code를 `0x10000` address에 load하고 file의 선두에서 실행이 시작되는 것이 아니기 때문에 `cs`를 `0x10200`으로 설정합니다. 여기서 jump합니다:

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

이것은 [4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46) 로부터 `512` byte offset의 위치입니다. 또한 `cs`와 모든 다른 segment registers를 `0x10200`에서 `0x10000`으로 align해야 합니다. 이후 stack을 setup합니다:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

`ds`의 값을 stack에 push하고 이어서 [6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602) label의 address를 push하고 `lretw` instruction을 수행합니다. `lretw` instruction이 호출되면, label `6`의 address를 [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register에 load하고 `ds`의 값을 `cs`에 load합니다. 이후 `ds`와 `cs`는 같은 값을 가지게 됩니다.

Stack Setup
--------------------------------------------------------------------------------

setup code의 대부분은 real mode에서 C language environment를 위한 준비입니다. 다음 [step](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575)은 `ss` register의 값을 확인하고, 만약 `ss`가 잘못되어 있다면 적절한 stack을 만듭니다:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

이것은 3개의 다른 secnarios를 만들 수 있습니다:

* `ss`가 valid한 값 `0x1000`을 가지고 있음 (`cs`를 제외한 모든 다른 segment registers도 동일)
* `ss`가 invalid하고 `CAN_USE_HEAP` flag가 set되어 있음 (하기 참조)
* `ss`가 invalid하고 `CAN_USE_HEAP` flag가 set되어 있지 않음 (하기 참조)

이 3개의 scenarios를 순서대로 봅시다:

* `ss`가 올바른 address (`0x1000`)를 가지고 있습니다. 이 경우, label [2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589)로 갑니다:

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

여기서 (bootloader에서 `sp`의 값을 설정한) `dx`를 `4` bytes에 alignment를 set하고 0이 되는지 check합니다. 만약 0이라면 `dx`에 (64 KB의 최대 segment size에서 직전에 4 byte align된 address인) `0xfffc`를 설정합니다. 만약 0이 아니라면 bootloader에서 설정한 `sp`의 값을 계속해서 사용합니다 (위의 예의 경우 0xf7f4). 이후 `0x1000`의 올바른 segment address가 저장되어 있는 `ax`의 값을 `ss`에 설정하고 올바른 `sp`를 설정합니다. 이제 올바른 stack이 setup되었습니다:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* 두번째 scenario에서 (`ss` != `ds`) 먼저 [_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) (setup code의 마지막 address)를 `dx`에 설정하고, heap을 사용할 수 있는지 확인하기 위해 `testb` instruction을 사용하여 `loadflags` header field를 check합니다. [loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320)는 bitmask header로 하기와 같이 define되어 있습니다:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

boot protocol에서 하기와 같이 확인할 수 있습니다:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

만약 `CAN_USE_HEAP` bit이 설정되어 있다면, (`_end`를 가르키고 있는) `heap_end_ptr`을 `dx`에 설정하고 `STACK_SIZE` (최소 stack size, 1024 bytes)를 더해줍니다. 이후 `dx`가 carry가 발생하지 않았다면 (일반적으로 발생하지 않음, `dx = _end + 1024`), label `2` (이전 case)로 jump하고 올바른 stack을 setup합니다.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* `CAN_USE_HEAP`이 set되어 있지 않은 경우, `_end`부터 `_end + STACK_SIZE`까지를 minimal stack으로 사용합니다:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSS Setup
--------------------------------------------------------------------------------

main C code로 jump하기 전에 진행해야 하는 마지막 2개의 steps은 [BSS](https://en.wikipedia.org/wiki/.bss) area를 setup하는 것과 "magic" signature를 check하는 것입니다. 먼저 signature check는 다음과 같이 수행합니다:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

단순히 [setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)와 magic number `0x5a5aaa55`를 비교합니다. 만약 이들이 다르다면 fatal error를 report합니다.

만약 magic number가 일치한다면, segment registers와 stack이 올바르게 설정되었다고 판단합니다. 이제 C code로 jump하기 전에 BSS section만 설정하면 됩니다.

BSS section은 static하게 alloc되고 initialize되지 않은 data를 저장하기 위해 사용됩니다. linux는 다음의 code를 사용하여 이 영역을 0으로 초기화 시킵니다:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

먼저 [__bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) address가 `di`에 설정됩니다. 다음으로 `_end + 3` address (+3 - 4 bytes에 align)가 `cx`에 설정됩니다. `eax` register가 (`xor` instruction을 사용하여) clear되고, bss section size (`cx`-`di`)가 계산되어 `cx`에 설정됩니다. 그리고 `cx`가 4 ('word'의 크기)로 나뉘어 지고, `stosl` instruction이 반복적으로 사용되어 `eax`의 값 (0)를 `di`가 가르키는 address에 저장합니다. `di`의 값은 `cx`가 0이 될때까지 4씩 자동적으로 증가합니다. 이 code의 최종적인 영향은 `__bss_start`와 `_end`사이의 모든 word가 0으로 write되는 것입니다:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Jump to main
--------------------------------------------------------------------------------

이것이 전부입니다 - stack과 BSS가 올바로 설정되었고, 이제 `main()` C function으로 jump할 수 있습니다:

```assembly
    calll main
```

`main()` function은 [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)에 위치해 있습니다. 이것이 무엇을 하는지 다음 part에서 읽으실 수 있습니다.

Conclusion
--------------------------------------------------------------------------------

이것으로 linux kernel insides의 첫번째 part를 마칩니다. 질문이나 제안이 있으시면 Twitter [0xAX](https://twitter.com/0xAX)나 [email](anotherworldofworld@gmail.com)로 연락을 주시거나, [issue](https://github.com/0xAX/linux-internals/issues/new)를 생성해 주십시요. 다음 part에서는 linux kernel setup에서 첫 C code 수행과 `memset`, `memcpy`, `earlyprintk`와 같은 memory routines 구현과 early console의 구현과 초기화 등을 살펴 볼 것입니다.

**영어는 저의 모국어가 아닙니다. 불편한 점이 있으셨다면 먼저 사죄드립니다. 만약 잘못된 점을 발견하시면 [linux-insides](https://github.com/0xAX/linux-internals)로 PR을 보내주십시요.**

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
