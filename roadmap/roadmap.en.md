[ 🇷🇺 Русский ](roadmap.ru.md) | [ 🇬🇧 English ](roadmap.en.md)

# Wunix Development Roadmap and Milestone Status

This document defines the development phases of the Wunix kernel and base system, current milestone progress, and detailed task checklists.

---

## 1. Summary of Development Phases

| Phase | Subsystem / Focus | Core Components | Status | Deliverable / Completion Criteria |
|:---:|:---|:---|:---:|:---|
| **1** | **Hardware Bring-Up & CPU** | GDT, TSS, IST0, IDT, PIC 8259, Serial COM1, Limine, Localization | ✅ **Completed** | Kernel boots stably into 64-bit Long Mode, dispatches CPU faults, supports runtime localization. |
| **2** | **Memory Management** | Frame Allocator, Paging (HHDM), Heap (1 MiB), `alloc` | ✅ **Completed** | Dynamic memory (`Box`, `Vec`, `String`), Higher-Half heap (`0xFFFF_9000_...`), UNIX autoconfiguration log. |
| **3** | **Input & System Timers** | PS/2 Keyboard (Ring Buffer, Alt+Shift), PIT 8254 Timer | ✅ **Completed** | Thread-safe input queue, system uptime tracking, `Ctrl+C` interrupt handling. |
| **4** | **Kernel Multitasking** | TCB, thread stacks, `context_switch`, scheduler | 🔄 **In Progress** | Preemptive multitasking triggered by PIT timer ticks, isolated kernel thread stacks. |
| **5** | **VFS & Storage** | VFS traits, Ramdisk / Initramfs (TAR), DeviceManager | ⏳ *Planned* | Root filesystem (`/`) mounting, loading initial modules and files from ramdisk. |
| **6** | **Userland (Ring 3)** | Process address space isolation, `syscall` / `sysret`, ELF parser | ⏳ *Planned* | CPU transition to Ring 3 with privilege separation and fast system call handling. |
| **7** | **First Process & WSH** | `/bin/init` process, Wunix Shell, basic system utilities | ⏳ *Planned* | Interactive userland command shell running in Ring 3. |

---

## 2. Architectural Pipeline

```
[1. Hardware Bring-up] (GDT, IDT, PIC, Framebuffer, Serial, Limine) ──► [COMPLETED]
         │
         ▼
[2. Memory Management] (Higher-Half, Paging, 1MiB Heap, alloc) ───────► [COMPLETED]
         │
         ▼
[3. Input & Timers] (PS/2 buffer, Alt+Shift, PIT 100 Hz) ─────────────► [COMPLETED]
         │
         ▼
[4. Multitasking] (Kernel threads, context switching) ────────────────► [IN PROGRESS]
         │
         ▼
[5. Filesystem & VFS] (Initramfs / ramdisk loading)
         │
         ▼
[6. Transition to Ring 3] (User Space isolation, syscall / sysret)
         │
         ▼
[7. Initial Process: init / wsh] ────► Userland Interactive Shell
```

---

## 3. Detailed Task Checklist

### Phase 1: Hardware Bring-Up and Boot
- [x] GDT descriptors and TSS (Task State Segment) setup.
- [x] Dedicated 20 KB emergency stack (IST0) for Double Fault interception.
- [x] IDT interrupt table configuration (Breakpoint, Double Fault, Page Fault).
- [x] 8259 PIC controller remapping to vector range `32..47`.
- [x] UART 16550 serial port driver (COM1, port `0x3F8`) for diagnostic terminal logging.
- [x] **Limine v0.6.5** bootloader protocol integration (Higher-Half, UEFI and BIOS support).
- [x] Modular directory organization adhering to Section 5 of AGENTS.md.
- [x] Kernel localization subsystem supporting runtime RU and EN selection at boot.

### Phase 2: Memory Management Subsystem
- [x] Limine HHDM (Higher-Half Direct Map) offset resolution.
- [x] 4-level page table translator setup using `OffsetPageTable`.
- [x] `LimineFrameAllocator` physical frame allocator (dispensing 4 KiB frames from `MEMMAP_USABLE`).
- [x] `linked_list_allocator` heap integration and `extern crate alloc` activation.
- [x] Higher-Half heap mapping (`0xFFFF_9000_0000_0000`, 1 MiB initial capacity).
- [x] `MemoryStats` structure reporting physical memory distribution (`real mem`, `avail mem`).
- [x] UNIX/BSD autoconfiguration boot log formatting.

### Phase 3: Core Input Drivers and System Timers
- [x] PS/2 keyboard driver with ring event buffer (`RingBuffer<char, 128>`).
- [x] Shift, Ctrl, Alt, and CapsLock modifier tracking.
- [x] Dual keyboard layout support (EN / RU) with `Alt+Shift` toggling.
- [x] `Ctrl+C` line cancellation handling.
- [x] PIT 8254 system timer initialization (100 Hz, divisor 11932).
- [x] Global atomic tick counter and `uptime_ms` elapsed time function.

### Phase 4: Kernel Multitasking and Scheduling (Active)
- [ ] Thread Control Block (TCB) data structure.
- [ ] Individual stack allocation for kernel execution threads.
- [ ] Assembly CPU register save/restore procedure (`context_switch`).
- [ ] Cooperative thread yielding (`yield_now`).
- [ ] Preemptive Round-Robin scheduler driven by PIT timer ticks.
- [ ] Kernel synchronization primitives: spinlocks (`SpinMutex`), sleep locks, and wait queues.
