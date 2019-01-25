# Kernel Boot Process

이 chapter는 linux kernel의 boot process를 설명합니다. Kernel의 loading process의 전체 cycle을 설명하는 일련의 posts를 볼 수 있습니다:

* [Bootloader부터 kernel까지](linux-bootstrap-1.md) - computer의 전원을 켜는 것부터 kernel의 첫번째 instruction을 실행하는 모든 stages를 설명합니다.
* [Kernel setup code에서 첫번째 steps](linux-bootstrap-2.md) - kernel setup code의 첫번째 steps을 설명합니다. You will see heap 초기화, EDD, IST 같은 다른 parameters의 query를 볼 수 있습니다.
* [Video mode 초기화와 protected mode로의 천이](linux-bootstrap-3.md) - kernel setup code에서 video mode 초기화와 protected mode로 천이를 설명합니다.
* [64-bit mode로의 천이](linux-bootstrap-4.md) - 64-bit mode로 천이를 위한 준비와 천이의 세부 내용을 설명합니다.
* [Kernel 압축 해제](linux-bootstrap-5.md) - kernel 압축 해제 전 준비와 직접 압축 해제의 세부 내용을 설명합니다.
* [Kernel random 주소 randomization](linux-bootstrap-6.md) - Linux kernel이 적제되는 주소의 randomization을 설명합니다.

이 chapter는 `Linux kernel v4.17`과 일치합니다.
