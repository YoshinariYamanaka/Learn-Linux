https://www.brendangregg.com/blog/2017-11-29/aws-ec2-virtualization-2017.html

| Method                    | Label in Gregg's diagram     | Guest OS                  | Overhead (Gregg's article) |
| ------------------------- | ---------------------------- | ------------------------- | -------------------------- |
| Emulated in software (SW) | VS (Virtualized in software) | Unaware it is virtualized | 2x to 10x slower           |
| Para-virtualization (PV)  | P (Para-virtualization)      | Aware it is virtualized   | 10% to 50%                 |
| Hardware-assisted (HW)    | VH (Virtualized in Hardware) | Unaware it is virtualized | 0.1% to 1.5%               |

The three methods are chosen separately for each component: CPU/memory, network, storage, interrupts/timers and boot. "PV" and "HVM" are just names for a combination.

| Generation                         | CPU/Memory        | Network    | Storage  | Interrupts/timers | Boot |
| ---------------------------------- | ----------------- | ---------- | -------- | ----------------- | ---- |
| Fully emulated (early VMware)      | SW                | SW         | SW       | SW                | SW   |
| Xen PV (m1.small)                  | PV                | PV         | PV       | PV                | PV   |
| Xen HVM 3.0                        | HW                | PV         | PV       | SW                | SW   |
| Xen PV-HVM                         | HW                | PV         | PV       | PV                | SW   |
| Xen 2013 (c3, enhanced networking) | HW                | HW(SR-IOV) | PV       | PV                | SW   |
| Xen 2017 (c4, i3)                  | HW                | HW         | HW(NVMe) | PV                | SW   |
| Nitro (c5, m5)                     | HW                | HW         | HW       | HW(APICv)         | SW   |
| Bare metal                         | No virtualization |            |          |                   |      |

Reading top to bottom, each component is replaced with HW one at a time, starting from the left (CPU).

# What SW / PV / HW mean in each column

|                                           | SW (VS)                         | PV (P)                                        | HW (VH)                  |
| ----------------------------------------- | ------------------------------- | --------------------------------------------- | ------------------------ |
| [CPU](cpu.md)                             | Binary translation              | Hypercalls                                    | VT-x/AMD-V               |
| [Memory](cpu.md)                          | shadow page table               | `mmu_update` hypercall                        | EPT / NPT                |
| [Network](network.md)                     | Emulated NIC (e1000 and others) | netfront/netback, virtio-net                  | SR-IOV (ixgbevf, ENA)    |
| [Storage](storage.md)                     | Emulated disk (IDE and others)  | blkfront/blkback, virtio-blk                  | NVMe                     |
| [Interrupts/timers](interrupts-timers.md) | Emulated PIC/IO-APIC, PIT/HPET  | Event channels, PV clock (`xen`, `kvm-clock`) | APICv, posted interrupts |
| [Boot](boot.md)                           | Emulated BIOS/UEFI              | PV-GRUB (boots the kernel directly, no BIOS)  | None                     |
