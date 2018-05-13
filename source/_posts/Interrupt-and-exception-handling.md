---
title: Interrupt and exception handling
date: 2018-05-13 16:38:39
tags:
- OSDI
- note
---
# Interrupt and exception handling

This note is based on [intel software development manual](https://software.intel.com/sites/default/files/managed/39/c5/325462-sdm-vol-1-2abcd-3abcd.pdf) Volume 3A chap 6

## Interrupt and exception overview
- Interrupts occur at random times during the execution of a program, in response to signals from hardware. System hardware uses interrupts to handle events external to the processor, such as requests to service peripheral devices. Software can also generate interrupts by executing the `INT n` instruction.
- Exceptions occur when the processor detects an error condition while executing an instruction
    - division by zero
    - protection violations
    - page faults
    - internal machine faults
## Exception and interrupt vectors
- The processor uses the vector number assigned to an exception or interrupt as an index into the interrupt descriptor table (IDT). The table provides the entry point to an exception or interrupt handler
- 0 through 31 are reserved
- 32 to 255 are designated as user-defined interrupts

![](https://i.imgur.com/mr3BGdh.png =500x)

## Sources of interrupts
- External Interrupts
External interrupts are received through pins on the processor or through the local APIC(Advanced Programmable Interrupt Controller)
- Maskable Hardware Interrupts
Any external interrupt that is delivered to the processor by means of the INTR pin or through the local APIC is called a maskable hardware interrupt
- Software-Generated Interrupts
The `INT n` instruction permits interrupts to be generated from within software by supplying an interrupt vector
number as an operand. For example, the INT 35 instruction forces an implicit call to the interrupt handler for interrupt 35.
## Source of exceptions
- Processor-detected program-error exceptions
- Software-generated exceptions
The `INTO`, `INT 3`, and `BOUND` instructions permit exceptions to be generated in software. These instructions allow checks for exception conditions to be performed at points in the instruction stream. For example,` INT 3` causes a breakpoint exception to be generated.
- Machine-check exceptions
## Exception classifications
- Faults
A fault is an exception that can generally be corrected and that, once corrected, allows the program to be restarted with no loss of continuity. When a fault is reported, the processor restores the machine state to the state prior to the beginning of execution of the faulting instruction. The return address (saved contents of the CS and EIP registers) for the fault handler points to the **faulting instruction**, rather than to the instruction following the faulting instruction.
- Traps
A trap is an exception that is reported immediately following the execution of the trapping instruction. Traps allow execution of a program or task to be continued without loss of program continuity. The return address for the trap handler points to the **instruction to be executed after the trapping instruction**.
    - If a trap is detected during an instruction which transfers execution, the return instruction pointer reflects the transfer.
    - If a trap is detected while executing a `JMP` instruction, the return instruction pointer points to the destination of the `JMP` instruction, not to the next address past the `JMP` instruction.
    - overflow exception is a trap exception. Here, the return instruction pointer points to the instruction following the `INTO` instruction that tested EFLAGS.OF (overflow) flag. The trap handler for this exception resolves the overflow condition. Upon return from the trap handler, program or task execution continues at the instruction following the `INTO` instruction.
- Aborts
An abort is an exception that does not always report the precise location of the instruction causing the exception and does not allow a restart of the program or task that caused the exception. Aborts are used to report severe errors, such as hardware errors and inconsistent or illegal values in system tables.

## Enabling and disabling interrupts
- The processor inhibits the generation of some interrupts, depending on the state of the processor and of the `IF` and `RF` flags in the `EFLAGS` register
- When the `IF` flag is clear, the processor inhibits interrupts delivered to the `INTR` pin or through the local APIC from generating an internal interrupt request
- when the `IF` flag is set, interrupts delivered to the `INTR` or through the local APIC pin are processed as normal external interrupts.
- The `IF` flag does not affect non-maskable interrupts (NMIs) delivered to the NMI pin or delivery mode NMI messages delivered through the local APIC
- The `IF` flag can be set or cleared with the `STI` (set interrupt-enable flag) and `CLI` (clear interrupt-enable flag) instructions
## Interrupt descriptor table
The interrupt descriptor table (IDT) associates each exception or interrupt vector with a gate descriptor for the procedure or task used to service the associated exception or interrupt.

The base addresses of the IDT should be aligned on an 8-byte boundary to maximize performance of cache line fills. The limit value is expressed in bytes and is added to the base address to get the address of the last valid byte. A limit value of 0 results in exactly 1 valid byte. Because IDT entries are always eight bytes long, the limit should always be one less than an integral multiple of eight (that is, 8N – 1).

The `LIDT` instruction loads the IDTR register with the base address and limit held in a memory operand. This instruction can be executed only when the CPL is 0. It normally is used by the initialization code of an operating system when creating an IDT

![](https://i.imgur.com/2Cxiovn.png)

## IDT DESCRIPTORS
- The task gate contains the segment selector for a TSS for an exception and/or interrupt handler task
- Interrupt and trap gates contain a far pointer (segment selector and offset) that the processor uses to transfer program execution to a handler procedure in an exception- or interrupt-handler code segment
![](https://i.imgur.com/5TXaMZs.png =400x)

## Exception and interrupt handling
An interrupt gate or trap gate references an exception- or interrupt-handler procedure that runs in the context of the currently executing task. The segment selector for the gate points to a segment descriptor for an executable code segment in either the GDT or the current LDT. The offset field of the gate descriptor points to the beginning of the exception- or interrupt-handling procedure.
![](https://i.imgur.com/H3a5tMC.png)

### Exception- or Interrupt-Handler Procedures
When the processor performs a call to the exception- or interrupt-handler procedure:
* If the handler procedure is going to be executed at a numerically lower privilege level, a stack switch occurs. When the stack switch occurs:
    1. The segment selector and stack pointer for the stack to be used by the handler are obtained from the TSS for the currently executing task. On this new stack, the processor pushes the stack segment selector and stack pointer of the interrupted procedure.
    2. The processor then saves the current state of the EFLAGS, CS, and EIP registers on the new stack
    3. If an exception causes an error code to be saved, it is pushed on the new stack after the EIP value
* If the handler procedure is going to be executed at the same privilege level as the interrupted procedure
    1. The processor saves the current state of the EFLAGS, CS, and EIP registers on the current stack
    2. If an exception causes an error code to be saved, it is pushed on the current stack after the EIP value
![](https://i.imgur.com/cz81uJd.png)
- To return from an exception- or interrupt-handler procedure, the handler must use the IRET (or IRETD) instruction.The IRET instruction is similar to the RET instruction except that it restores the saved flags into the EFLAGS register
#### Protection
The processor does not permit transfer of execution to an exception- or interrupt-handler procedure in a less privileged code segment than the CPL.
- The processor checks the DPL of the interrupt or trap gate only if an exception or interrupt is generated with an INT n, INT 3, or INTO instruction. Here, the CPL must be less than or equal to the DPL of the gate.
    -  prevents application programs or procedures running at privilege level 3 from using a software interrupt to access critical exception handlers, such as the page-fault handler
- Hardware-generated interrupts and processor-detected exceptions, the processor ignores the DPL of interrupt and trap gates.
