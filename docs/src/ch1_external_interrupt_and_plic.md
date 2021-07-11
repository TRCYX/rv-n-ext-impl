# 外部中断与平台级中断控制器（PLIC）

## 外部中断

外部中断主要用于处理与外设相关的中断，如 GPIO、UART、DMA 等。PLIC 接收所有外设的中断信号，根据程序配置的优先级、阈值和上下文规则，在相应的核上触发外部中断。程序需要从 PLIC 的 MMIO 寄存器中进一步读取具体的中断外设源信息。

`mip.MEIP` 位对程序是只读的，只能由 PLIC 写入或清除。 `mip.SEIP` 可读写， M 态程序可以将该位置 1 ，而是否产生 S 态外部中断由该位的值和 PLIC 相应的信号逻辑或的结果决定，二者中任一为 1 即产生中断； `csrr` 、 `csrrs` 和 `csrrc` 指令在该位上的行为略有不同，具体可见规范。 `sip.SEIP` （对 S 态程序）是只读的。对于用户态中断可以设计类似的约束条件。

## PLIC

目前的 [PLIC 规范](https://github.com/riscv/riscv-plic-spec/blob/master/riscv-plic.adoc) 支持至多 1024 个（外设）中断源和 15872 套上下文（具体含义将在下文分析），每个中断源可配置 2^32 种优先级，每套上下文可配置 2^32 种优先级阈值，以及在每套上下文中可配置每个中断源是否使能。

### 中断触发条件

1. 中断源产生中断等待信号；
2. 在某套上下文中，该中断源被使能；
3. 该中断源的优先级高于该上下文的优先级阈值；

上述条件均满足时，PLIC 会在核中触发外部中断，核编号与中断的特权级由上下文的设计决定。

### 上下文

规范中并未给出 “上下文” 的严格定义，但在[一个 Issue 中有人提到](https://github.com/riscv/riscv-plic-spec/issues/10#issuecomment-641632618)，在实践中一套上下文通常指代每个核心和特权级的组合，如果 CPU 中有三个核和两个可以处理中断的特权级（ M 和 S ），那么就存在六套 PLIC 上下文（在某个核上触发某个特权级的外部中断）。在该场景下，不妨按以下方法对上下文编号：

|             | 核心 1   | 核心 2   | 核心 3   |
| ----------- | -------- | -------- | -------- |
| 运行在 M 态 | 上下文 1 | 上下文 2 | 上下文 3 |
| 运行在 S 态 | 上下文 4 | 上下文 5 | 上下文 6 |

设某个中断源在上下文 1 和 6 中符合中断触发条件，那么当核心 1 运行在 M 态时，PLIC 会在核心 1 上触发 MEI ；核心 3 运行在 S 态时，会在核心 3 上触发 SEI ；在其他特权级，以及核心 2 上，该中断源不会触发外部中断。

### 中断领取与完成

PLIC 中每套上下文具有一个领取/完成（`claim/complete`）寄存器。程序读取该寄存器时，PLIC 会返回该上下文中优先级最高、等待信号有效且被使能的中断源编号（该中断源的优先级可以低于上下文阈值），同时清除该中断源的等待位，并屏蔽其信号。
程序向该寄存器中写入一个中断编号以通知 PLIC 该中断处理完成，若相应中断源在其上下文中被使能，则 PLIC 解除相应的屏蔽，否则忽略本次写入。

## 用户态外部中断设计

实现了 N 扩展的系统可以将外部中断发送至用户态程序进行处理，以提高用户态驱动程序的效率。我们认为，这种场景对外部中断有如下需求：

1. 实时性：外部中断由外设触发，触发时机和传递的信息基本不受核心控制，程序应及时处理，降低丢失信息的风险；
2. 隔离性：不开启虚拟化的情况下，机器中通常只存在一个内核，不需要考虑中断如何分发；但用户态通常有多个进程同时（或分时复用）运行，A 外设不应导致 B 外设的用户态驱动或其他不相关的进程进入中断；
3. 安全性：中断源的优先级和不同上下文中的使能应当由内核设置，用户态程序不能任意修改；
4. 兼容性：外部中断应当与现有操作系统的若干机制能够良好兼容，如基于页表的地址空间映射、基于时间片的分时复用等。

基于以上需求和现有的外部中断与 PLIC 规范，我们提出如下设计方案：

1. 对于每个核心，M 态和 S 态各自占有一套 PLIC 上下文，分别产生 MEI 和 SEI； U 态占有多套上下文，其中至少一套产生 SEI ，用于非驱动进程，其余产生 UEI ，用于驱动进程。具体上下文数目大致与中断源数目成正比，但不需要一一对应。一种上下文编号方案示例如下：

   |            | 核心 1            | 核心 2            | ... | 核心 m        |
   | ---------- | ----------------- | ----------------- | --- | ------------- |
   | M 态       | 上下文 1          | 上下文 2          | ... | 上下文 m      |
   | S 态       | 上下文 (m+1)      | 上下文 (m+2)      | ... | 上下文 2m     |
   | U 态驱动 1 | 上下文 (2m+1)     | 上下文 (2m+2)     | ... | 上下文 3m     |
   | U 态驱动 2 | 上下文 (3m+1)     | 上下文 (3m+2)     | ... | 上下文 4m     |
   | ...        | ...               | ...               | ... | ...           |
   | U 态驱动 n | 上下文 [(n+1)m+1] | 上下文 [(n+1)m+2] | ... | 上下文 (n+2)m |

2. 进程切换时，内核将驱动编号写入一个每个核心独有、PLIC 可见的寄存器中，用于 PLIC 区分上下文；这个编号不一定等于进程的 PID ；
3. 非驱动进程因 SEI 进入内核后，内核应尽快将相应的驱动进程调度运行；可能需要由内核完成中断领取/完成过程，取出中断源编号，再通过将 sip.UEIP 置 1 实现中断转发；
4. PLIC 规范中各部分地址空间基本对齐到 4KB 边界，可以简单地将中断源优先级、等待位、使能位对应的地址空间映射到内核中，而每个上下文的阈值和领取/完成寄存器映射到对应的驱动程序地址空间中；
5. 对于实时性要求较高的外设，内核可以提高其驱动进程优先级，甚至将其长期绑定在一个核上轮询中断是否产生。