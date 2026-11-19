# 3. Interrupts

## The full classification tree

The single question that matters most is **which of these are actually interrupts**, so that is the
tree's first branch: everything under **A** is an interrupt or an exception, and everything under **B** diverts the CPU without being either. Which IDT vector each one uses is in the table further down; A vector is marked inline only where the number identifies the item: every exception in branch (2), the NMI, and `int 0x80`.

```
Events that divert the CPU from the program it is running
│
├─ A. INTERRUPTS AND EXCEPTIONS
│  │
│  ├─ (1) Asynchronous -- from "outside," unrelated to the current instruction
│  │   │   = a hardware interrupt in the narrow sense
│  │   │
│  │   ├─ Maskable (IRQ) -- the CPU can disable these (cli/sti, the IF flag)
│  │   │   │
│  │   │   ├─ Device interrupts (raised by external devices)
│  │   │   │   ├─ Pin based, through the IO-APIC
│  │   │   │   │   ├─ Keyboard, mouse        (i8042)
│  │   │   │   │   ├─ Serial and others      (ttyS0 and so on)
│  │   │   │   │   ├─ Real-time clock        (rtc0)
│  │   │   │   │   └─ ACPI                   (acpi)
│  │   │   │   └─ MSI / MSI-X, delivered as a memory write
│  │   │   │       ├─ Storage                (nvme0q0, nvme0q1, ...)
│  │   │   │       ├─ Network                (ens5-Tx-Rx-0 .. -7, spread by RSS)
│  │   │   │       └─ Network management     (ena-mgmnt@pci)
│  │   │   │
│  │   │   ├─ Local APIC interrupts -- raised by the CPU's own APIC, not a device
│  │   │   │   ├─ Timer                      (LOC)  the heartbeat of the scheduler
│  │   │   │   ├─ Spurious                   (SPU)  a false interrupt
│  │   │   │   ├─ Performance monitoring     (PMI)  a perf counter overflowed
│  │   │   │   ├─ IRQ work                   (IWI)
│  │   │   │   ├─ Thermal event              (TRM)
│  │   │   │   ├─ Threshold                  (THR)  corrected errors crossed a limit
│  │   │   │   ├─ Deferred error             (DFR)
│  │   │   │   ├─ ICR read retry             (RTR)
│  │   │   │   └─ APIC error, missed         (ERR, MIS)
│  │   │   │
│  │   │   ├─ Inter-processor interrupts (IPI) -- CPUs signaling each other
│  │   │   │   ├─ Rescheduling               (RES)  ask another core to reschedule
│  │   │   │   ├─ Function call              (CAL)  run a function on another core
│  │   │   │   └─ TLB shootdown              (TLB)  invalidate another core's TLB
│  │   │   │
│  │   │   └─ Posted interrupts -- virtualization hardware delivers a device
│  │   │       └─ interrupt straight to a guest vCPU   (PIN, NPI, PIW)
│  │   │
│  │   └─ Non-maskable -- cannot be ignored, highest priority
│  │       ├─ NMI, vector 2: fatal hardware error, watchdog,
│  │       │                 send-diagnostic-interrupt            (NMI)
│  │       └─ SMI, no vector: a real interrupt, but firmware handles it in SMM below
│  │                          the OS (power, thermal, hardware errors). Invisible to
│  │                          Linux; look at the SMI column of turbostat, or at
│  │                          unaccounted time
│  │
│  ├─ (2) Synchronous -- detected by the CPU while executing an instruction
│  │   │   = an exception. Each one has its own fixed vector, shown inline below
│  │   │
│  │   ├─ Fault -- correctable. Fix it and "retry" the same instruction
│  │   │   ├─ Page fault, vector 14 (also a write to a read-only page)
│  │   │   │   │   see /proc/vmstat pgfault and pgmajfault, ps -o min_flt,maj_flt
│  │   │   │   ├─ Minor   in RAM but not in the page table. No disk I/O, so cheap
│  │   │   │   ├─ Major   must be read from disk. Storage latency, so expensive
│  │   │   │   └─ Invalid outside the address space, becomes a segmentation fault
│  │   │   ├─ General protection fault, vector 13 (privilege violations)      dmesg
│  │   │   ├─ Invalid opcode, vector 6 (an undefined instruction)             SIGILL
│  │   │   └─ Division by zero, vector 0                                      SIGFPE
│  │   │
│  │   ├─ Trap -- execution continues at the instruction "after" the current one
│  │   │   ├─ Debug, vector 1 (a debug register fired)                        SIGTRAP
│  │   │   ├─ Breakpoints, vector 3 (int3; kprobes and perf probes ride on it)
│  │   │   ├─ Overflow, vector 4 (the into instruction)
│  │   │   └─ System calls (on some architectures; the standard way there)
│  │   │
│  │   └─ Abort -- unrecoverable. The process or the system is terminated
│  │       ├─ Machine check exception, vector 18, an uncorrectable error
│  │       │                                                    (MCE)  mcelog
│  │       └─ Double fault, vector 8, raised when a handler itself faults (inconsistent system tables land here too)
│  │
│  └─ (3) Software interrupts proper -- an instruction dispatched through the IDT
│      ├─ int n     the vector is whatever the instruction names
│      └─ int 0x80  vector 128, the 32-bit system call path                   strace
│
└─ B. NOT INTERRUPTS -- they divert the CPU, but are neither an interrupt nor an
   │                    exception, so none of them has a vector at all
   │
   ├─ (4) Fast kernel entry -- a system call without raising anything
   │   ├─ syscall   x86-64. Jumps to the address in MSR_LSTAR                 strace
   │   └─ sysenter  the 32-bit fast path, via MSR_SYSENTER_EIP                strace
   │
   ├─ (5) Deferred kernel work -- the bottom half of a hardware interrupt
   │   ├─ softirq   statically defined, parallel across CPUs. /proc/softirqs
   │   │            (NET_RX / NET_TX / TIMER / BLOCK and others)
   │   ├─ tasklet   on top of softirqs, one instance of each at a time
   │   └─ workqueue a kernel thread, so it may sleep. ps shows kworker/*
   │
   ├─ (6) Outside the OS's reach -- it can neither classify nor control these
   │   ├─ VM exit / VM entry   the CPU hands control to the hypervisor
   │   │                       host side only: perf kvm, kvm_stat
   │   └─ Hypercall (vmcall)   the guest calls the hypervisor on purpose
   │
   └─ (7) Routinely mistaken for an interrupt
       ├─ Machine check poll   (MCP)  a periodic kernel poll, counted in
       │                       /proc/interrupts next to the real interrupts
       ├─ Signal               the kernel notifying a process. Usually the
       │                       consequence of an exception, as when a page
       │                       fault becomes SIGSEGV. strace; block it with
       │                       sigprocmask
       └─ vDSO call            runs code mapped into the process, such as
                               gettimeofday. It never enters the kernel
```
Two consequences worth stating outright:
- **"Software interrupt" names two unrelated things.** `int n` in branch (3) is a CPU mechanism
  dispatched through the IDT exactly like an exception; a softirq in branch (5) is a Linux mechanism
  with no vector at all. That is why `/proc/interrupts` and `/proc/softirqs` are separate files
- **A system call has four implementations and only one of them is an interrupt.** `int 0x80` really
  is a software-generated interrupt, `syscall` and `sysenter` bypass the IDT, and a vDSO call does not
  even enter the kernel. So on modern x86-64 the main entry is not an interrupt at all, which is why
  system calls never appear in `/proc/interrupts` despite happening thousands of times a second
### How this relates to the IDT (Interrupt Descriptor Table)
The IDT is a CPU table that maps **vector numbers 0 through 255** to the handler to jump to. This is
where each range belongs in the tree above:

| Vector | Reserved for | Which branch of the tree |
| --- | --- | --- |
| `0-31` | Exceptions. Fixed by the CPU, one vector per exception | (2) Synchronous |
| `2` | The NMI. Inside the exception range, yet asynchronous | (1) Non-maskable |
| `32-255` | Assigned freely by the OS | (1) Maskable: devices, local APIC, IPI, posted |
| `128 (0x80)` | Registered by Linux for the 32-bit `int 0x80` path | (3) Software interrupts proper |
| `n` | Whatever the `int n` instruction names | (3) Software interrupts proper |
| none | Never reaches the IDT | All of **B**, plus the SMI leaf of (1) |

- Everything under **A** appears in the IDT, except the SMI leaf; nothing under **B** does
- **The `syscall` instruction of x86-64 does not appear in the IDT.** It jumps straight to the address
  held in `MSR_LSTAR` (`entry_SYSCALL_64`), so it carries no vector number; the same is true of the
  32-bit `sysenter` path. Vector 128 is registered only for the 32-bit `int 0x80` path. In other words
  the main system call entry on a modern x86-64 Linux is not an interrupt at all in the CPU's sense
- **The softirqs, tasklets, and workqueues of branch (5) do not appear in the IDT** (they are software
  mechanisms the kernel manages itself)
  - The flow is: a hardware interrupt arrives, the IDT dispatches the top half, the top half
    schedules a softirq, and the softirq runs later with no involvement from the IDT
- Where to look
  - `/proc/interrupts` : counts for everything that comes through the IDT (exceptions, device interrupts, IPIs, the NMI, and so on)
  - `/proc/softirqs` : counts for softirqs (outside the IDT, a kernel-level concept)
> Sources: the x86 IDT holds 256 vectors, with 0 through 31 reserved for exceptions and the NMI
> (Intel 80386 reference, MIT 6.828). The fault/trap/abort classification of exceptions
> (OSDev, Exceptions).




### The same idea on ARM64 (Graviton)
The IDT is specific to x86. ARM has an equivalent, so **the two axes of the tree carry over**, but the
table is a different shape and the cause is identified differently. On Graviton instances this is the
difference you actually see.

| | x86-64 | ARM64 (Graviton) |
| --- | --- | --- |
| Dispatch table | The IDT: 256 descriptors, located by `IDTR` | The exception vector table: 16 slots of 128 bytes of code, located by `VBAR_EL1` |
| How the cause is identified | The vector number itself | The slot, plus the `EC` field of `ESR_EL1` |
| Table layout | One vector per cause | 4 groups (current EL with SP0 / with SPx, lower EL in AArch64 / in AArch32) x 4 kinds (Synchronous, IRQ, FIQ, SError) |
| Device interrupts | Vectors 32-255, one per IRQ | All funnel into the single IRQ slot; the GIC reports which one as an INTID: SGI 0-15 (= IPI), PPI 16-31 (per-core, e.g. the timer), SPI 32-1019 (devices), LPI 8192+ (MSI based) |
| NMI | Vector 2 | No true equivalent. Linux builds a pseudo-NMI out of GIC interrupt priorities. FIQ is normally routed to firmware rather than to Linux, and SError is the asynchronous-abort analogue of a machine check |
| Software interrupt | `int n` | No equivalent. `svc` / `hvc` / `smc` raise synchronous exceptions instead |
| System call | `syscall`, which bypasses the IDT, so it sits in branch **B** | `svc`, which is a synchronous exception, so it sits in branch **A**, under (2) |

> The last row is the one to remember: **the same system call lands on a different branch of the tree
> depending on the architecture.** On x86-64 it is not an interrupt; on ARM64 it is an exception.
### A closer look at the parts of the tree you cannot observe
Most of branch A is observable through `/proc/interrupts`, local APIC interrupts included. The SMI leaf of branch (1) and branch (6) are the two places where that breaks down, so they are worth a closer look:

- **SMI (System Management Interrupt)** -- the most important omission
  - Handled by firmware below the OS, in SMM (System Management Mode)
  - Used for power management, thermal control, hardware error handling, and similar duties
  - **Linux cannot even see it in the IDT and generally cannot control it** (the CPU enters SMM transparently)
- **Interrupts that come from virtualization** (EC2 and the like)
  - Posted interrupts (`PIN`/`NPI`/`PIW`), VM exit and entry, hypercalls

> Branch A is what you reason about in day-to-day operations and troubleshooting. The SMI leaf and
> branch (6) are in the tree to remind you that **some events never show up in any counter at all**.
> SMI above all occurs entirely outside the OS's jurisdiction.
### Reading `/proc/interrupts` from a real machine (AL2023, 16 vCPUs)
```
       (the CPU0..CPU15 columns are omitted; each row holds the running count per CPU)
  1:  ... IO-APIC   1-edge      i8042                  <- keyboard controller (legacy, pin based)
  4:  ... IO-APIC   4-edge      ttyS0                  <- serial console (155 on CPU3)
  8:  ... IO-APIC   8-edge      rtc0                   <- real-time clock
  9:  ... IO-APIC   9-fasteoi   acpi                   <- ACPI
 12:  ... IO-APIC  12-edge      i8042                  <- mouse (PS/2)
 24:  ... PCI-MSI 65536-edge    nvme0q0                <- NVMe admin queue (MSI)
 25:  ... PCI-MSI 65537-edge    nvme0q1                <- NVMe I/O queue 1 (16069 on CPU11)
 26:  ... PCI-MSI 65538-edge    nvme0q2                <- NVMe I/O queue 2 (14831 on CPU15)
 27:  ... PCI-MSI 81920-edge    ena-mgmnt@pci:...      <- ENA management queue
 28..35: PCI-MSI 81921..81928   ens5-Tx-Rx-0 .. -7     <- 8 ENA data queues (spread across CPUs = RSS)
```

Everything above is a **device interrupt with an IRQ number in the first column**. What follows has no IRQ number; these are architecture-specific interrupts, named by a mnemonic instead. They still use IDT vectors, at the top of the 32-255 range:

```
NMI:  0 ...        Non-maskable interrupts          <- the NMI (none occurred here)
LOC:  390k-560k    Local timer interrupts           <- the local timer. Frequent on every CPU (the heart of the scheduler)
SPU:  0 ...        Spurious interrupts              <- false interrupts
PMI:  0 ...        Performance monitoring interrupts <- performance monitoring (0, since perf is not in use)
IWI:  33-141       IRQ work interrupts              <- IRQ work
RTR:  0 ...        APIC ICR read retries
RES:  5k-9k        Rescheduling interrupts          <- IPI: reschedule
CAL:  50k-100k     Function call interrupts         <- IPI: run a function on another core
TLB:  around 6k    TLB shootdowns                   <- IPI: invalidate another core's TLB
TRM:  0 ...        Thermal event interrupts
THR:  0 ...        Threshold APIC interrupts
DFR:  0 ...        Deferred Error APIC interrupts
MCE:  0 ...        Machine check exceptions         <- an exception (abort class). None occurred = healthy
MCP:  73 (same on every CPU) Machine check polls    <- periodic polling (identical counts, since the interval is fixed)
PIN/NPI/PIW: 0 ... Posted-interrupt ...             <- the posted-interrupt counters used by virtualization
ERR:  0 / MIS: 0                                     <- the APIC error and missed counters
```
Observation points:
- **There are 8 queues, `ens5-Tx-Rx-0` through `-7`, and each takes its interrupts on a different CPU.**
  That is the evidence that RSS is spreading receive processing across several CPUs
- **`LOC` is orders of magnitude higher than anything else on every CPU.** Each CPU takes its own
  local timer interrupt and uses it to drive scheduling
- **`RES`, `CAL`, and `TLB` are spread over every CPU.** The cores are exchanging IPIs constantly,
  as expected on a multicore system
- **`MCP` holds the same value (73) on every CPU.** It is polled at a fixed interval, so the counts
  line up. `MCE` is 0, so the hardware is healthy
- **There is no row for `SMI`.** As described above, the OS cannot see SMIs. The very absence of that
  row is real-machine confirmation of the "no vector, invisible to Linux" note on the SMI leaf
- There *are* rows for `PIN/NPI/PIW`, meaning counters for posted interrupts are present, which is
  characteristic of virtualization and therefore of EC2
## What an interrupt is
- An event that diverts a program's normal flow of execution. Raised by hardware, or by the CPU itself
- With interrupts, the CPU can do other work instead of polling while it waits for an event, which
  is efficient
- **Interrupt handler (ISR: Interrupt Service Routine)**: the code that runs when an interrupt arrives
  - **First-Level Interrupt Handler (FLIH) = top half**
    - The hard, or fast, interrupt handler. Handles things quickly and schedules anything lengthy for the SLIH
    - It introduces execution jitter and it masks interrupts, which is exactly why the work is split in two
  - **Second-Level Interrupt Handler (SLIH) = bottom half**
    - The slow, or soft, interrupt handler. Runs lengthy work much as a process would
    - Handled by dedicated kernel threads or by a pool of worker threads
### Why interrupts exist at all
- They were introduced so that a polling loop would not waste CPU time waiting for an event
- Hardware interrupts originate in devices: a keystroke, an arriving packet, I/O completion, a hardware
  failure. Once the data is ready, the device sends an IRQ to tell the CPU
- A program raises a software interrupt when it needs the OS to do something for it
- **DMA (Direct Memory Access)**: the device accesses main memory directly, without going through the
  CPU. Once the transfer completes, a hardware interrupt notifies the CPU. Visible through `/proc/dma`
## Context switch
- Saving the state of a process or thread so it can be restored later
  - Store: write the process's structures to memory. Restore: load the registers back
- A prerequisite for sharing one CPU among several processes (the foundation of multitasking)
- Triggers
  - Multitasking (the scheduler switching tasks)
  - Interrupt handling
  - Transitions between user mode and kernel mode (though a mode transition by itself is not a context switch)
- The counts are visible through procfs
  - **voluntary**: the task had nothing to do and gave up the CPU on its own
  - **nonvoluntary**: the task wanted to continue but the OS switched away from it
```bash
# cat /proc/<PID>/status | grep ctxt
#   voluntary_ctxt_switches:    204
#   nonvoluntary_ctxt_switches:  56
```
## IRQ (Interrupt Request)
- The "deal with this now" signal sent from hardware to the CPU
- Each device is assigned one or more IRQ numbers
- `smp_affinity`: the affinity attribute that defines which CPUs may service that IRQ
```bash
# Which CPUs service this IRQ
# cat /proc/irq/1/smp_affinity
# The IRQ list (per-CPU interrupt counts, the type, and a description)
# cat /proc/interrupts
#   28: ... ens5-Tx-Rx-0   <- the network (ENA) interrupts are distributed across CPUs
```
## Scaling the network: RSS / RPS / RFS
Networking got fast enough that a single CPU can no longer keep up with interrupt processing, so the
work is spread across several CPUs.

| Mechanism | Implementation | Summary |
| --- | --- | --- |
| **RSS** (Receive-Side Scaling) | Hardware | Distributes work across several receive (Rx) queues. Maps a hash value to a queue through an indirection table. Requires NIC support |
| **RPS** (Receive Packet Steering) | Software | Achieves the equivalent of RSS in software. Usable with any NIC. Uses inter-processor interrupts |
| **RFS** (Receive Flow Steering) | Software | Extends RPS. Steers a packet toward the CPU where the consuming application runs, improving CPU cache efficiency |

- Without RSS, interrupts pile up on one CPU, and that CPU can no longer service its other interrupts
- Configuring RPS: `/sys/class/net/<device>/queues/<rx-queue>/rps_cpus` (a hexadecimal CPU bitmask)
  - `rx-` is a receive queue, `tx-` is a transmit queue
```bash
# ls /sys/class/net/eth0/queues       # rx-0 rx-1 tx-0 tx-1
# ls /sys/class/net/eth0/queues/rx-0/ # rps_cpus  rps_flow_cnt
```
### Enhanced networking and SR-IOV
- EC2 enhanced networking uses **SR-IOV** to deliver high performance
  - Enabled through either **ENA** or the Intel 82599 VF
  - Every current-generation instance supports it, except T2
- **SR-IOV**: a virtualization technology that presents one PCIe Ethernet controller as several PCIe devices
  - It lets a virtualized environment such as EC2 use hardware-based features like RSS
---
