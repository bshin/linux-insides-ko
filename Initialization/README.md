# Kernel initialization process

여기서는 kernel이 압축 해제된 이후부터 kernel의 첫번째 prcess의 시작까지 kernel initialization의 full cycel에 관련된 post를 찾아보실 수 있습니다.

*Note* 여기에 모든 kernel initialization steps이 있지 않습니다. 여기에는 interrupt handling, ACPI 등의 부분을 제외한 오직 일반적인 kernel part만 있습니다. 여기서 기술되지 않은 부분은 다른 chapters에서 기술할 것입니다.

* [First steps after kernel decompression](linux-initialization-1.md) - kernel에서 첫번째 step을 설명합니다.
* [Early interrupt and exception handling](linux-initialization-2.md) - early interrupts 초기화와 early page fault handler를 설명합니다.
* [Last preparations before the kernel entry point](linux-initialization-3.md) - `start_kernel` 호출 전 마지막 준비에 대하여 설명합니다.
* [Kernel entry point](linux-initialization-4.md) - kernel generic code의 첫번째 step에 대하여 설명합니다.
* [Continue of architecture-specific initializations](linux-initialization-5.md) - architecture-specific 초기화에 대하여 설명합니다.
* [Architecture-specific initializations, again...](linux-initialization-6.md) - architecture-specific 초기화 process에 대하여 계속하여 설명합니다.
* [The End of the architecture-specific initializations, almost...](linux-initialization-7.md) - `setup_arch`의 마지막에 관련된 내용을 설명합니다.
* [Scheduler initialization](linux-initialization-8.md) - scheduler 초기화 전 준비와 초기화에 대하여 설명합니다.
* [RCU initialization](linux-initialization-9.md) - [RCU](http://en.wikipedia.org/wiki/Read-copy-update)의 초기화에 대하여 설명합니다.
* [End of the initialization](linux-initialization-10.md) - linux kernel 초기화에 대한 마지막 part입니다.
