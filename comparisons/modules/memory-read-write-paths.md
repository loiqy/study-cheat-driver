# 内存读写路径对比 — Stage 2 专题分析

> Stage 2 产出 | 生成时间：2026-03-28
> 分析对象：NinDriver / GsDriver / VT_Driver

---

## 0. 一句话总结

三个项目各自选择了完全不同的内存读写策略：**NinDriver 在物理地址层面操作 PTE 窗口映射**，**GsDriver 在虚拟地址层面通过 MmCopyVirtualMemory + MDL 强写**，**VT_Driver 在进程上下文层面通过 KeStackAttachProcess / CR3 切换直接拷贝**。这三种方案分别代表了从底层到高层的三个抽象层级，各自有截然不同的检测面、稳定性和复杂度。

---

## 1. 调用链全景图

### 1.1 NinDriver — PTE 自映射物理内存读写

```
用户态 DeviceIoControl(IOCTL_READMEMORY / IOCTL_WRITEMEMORY)
    │
    ▼
DispatchIoctl (Driver.cpp:11)
    ├── pData->ProcessCr3     ← 用户态预先通过 IOCTL_GETPROCESSCR3 获取
    ├── pData->TargetAddress  ← 目标虚拟地址
    ├── pData->Buffer         ← 用户态缓冲区指针（直接使用，无 Probe）
    └── pData->Length
         │
         ▼
ReadPhysicalMemory / WritePhysicalMemory (Memory.cpp:171/214)     ← 第 3 层：跨页循环
    ├── 参数校验：CR3、BaseAddress、Size 范围检查
    ├── __try/__except 异常保护
    └── while (TotalSize) 循环：
         │
         ├── GetPhyicalAddress(ProcessCr3, VA)  (Memory.cpp:54)   ← 第 2 层：四级页表遍历
         │    ├── CR3 → 读 PML4E → 读 PDPTE → 读 PDE → 读 PTE
         │    ├── 每级通过 ReadPhysicalAddress 读物理内存
         │    ├── 检查 Present 位 (bit 0)
         │    ├── 大页处理：PDE bit7=1 → 2MB，PDPTE bit7=1 → 1GB
         │    └── 返回物理地址
         │
         └── ReadPhysicalAddress / WritePhysicalAddress            ← 第 1 层：PTE 自映射
              (Memory.cpp:103/137)
              ├── 跨页检查：PhysAddr>>12 == (PhysAddr+Size-1)>>12
              ├── KeAcquireSpinLockRaiseToDpc(&MapLock)  → DISPATCH_LEVEL
              ├── 计算 MapAddress 的 PTE 地址
              ├── 保存原 PTE 值
              ├── 修改 PTE：替换物理页帧号，保留控制位
              ├── __invlpg(MapAddress)  ← 刷新 TLB
              ├── RtlCopyMemory 通过 MapAddress + 页内偏移 读/写
              ├── 恢复原 PTE
              └── KeReleaseSpinLock
```

**CR3 获取调用链（前置步骤）：**

```
用户态 DeviceIoControl(IOCTL_GETPROCESSCR3)
    │
    ▼
GetProcessDirectoryBase(ProcessPid) (Memory.cpp:3)
    ├── 计算 PXE 自引用基址：PteOffsets → PdeBase → PpeBase → PxeBase → Cr3PteBase
    ├── MmGetPhysicalMemoryRanges() 获取物理内存范围
    └── 遍历所有 PFN 条目（每个 0x30 字节）：
         ├── MMPFN[0] != 0 且 != 1  → 有效
         ├── MMPFN[+8] == Cr3PteBase → 此 PFN 是 CR3 页
         ├── 解密 EPROCESS：(value | 0xF000000000000000) >> 13 | 0xFFFF000000000000
         └── 验证 UniqueProcessId 匹配 → 返回 PFN << 12
```

### 1.2 GsDriver — MmCopyVirtualMemory + MDL 强写

```
用户态 RegSetValueEx(Type='0006')
    │
    ▼
RegisterNotify (通讯回调.cpp:24)
    ├── 拦截 RegNtPreSetValueKey
    ├── PreSetValueInfo->Type == '0006'
    └── 解析 READ_WRITE_MEMORY_BUFFER：
         ├── hProcessId, TargetAddress, SourceAddress, NumberOfBytes
         └── ReadWriteType: 0=读, 1=写, 2=MDL强写
              │
              ├─────────────────────────────────────────────────────
              │ Type 0 (读) / Type 1 (写)
              │
              ▼
         ZwCopyVirtualMemory (导出函数.cpp:189)
              │  ← 实际调用 MmCopyVirtualMemory
              ├── 动态解析：RtlGetSystemFun(L"MmCopyVirtualMemory")
              ├── Type 0: MmCopyVirtualMemory(TargetProcess, TargetAddr,
              │                                CurrentProcess, SourceAddr,
              │                                Size, UserMode, &Copied)
              └── Type 1: MmCopyVirtualMemory(CurrentProcess, SourceAddr,
                                               TargetProcess, TargetAddr,
                                               Size, UserMode, &Copied)
              │
              ├─────────────────────────────────────────────────────
              │ Type 2 (MDL 强写 — 绕只读保护)
              │ 前提：NumberOfBytes <= PAGE_SIZE
              │
              ▼
         KeStackAttachProcess(pProcess, &ApcState)         ← 附加到目标进程
              │
              ├── MmCreateMdl(NULL, TargetAddress, Size)   ← 创建 MDL
              ├── MmProbeAndLockPages(MDL, KernelMode, IoReadAccess)
              │    ← 锁定物理页（IoReadAccess 允许读锁定只读页）
              ├── MmMapLockedPagesSpecifyCache(MDL, KernelMode, MmCached, ...)
              │    ← 重新映射到内核地址空间（绕过原虚拟地址的只读保护）
              ├── RtlCopyMemoryEx(MappedAddress, WriteData, Size)
              ├── MmUnmapLockedPages
              ├── MmUnlockPages
              ├── IoFreeMdl
              └── KeUnstackDetachProcess(&ApcState)
```

**关键辅助函数 — RtlSuperCopyMemory（用于内核代码页写入，如回调跳板）：**

```
RtlSuperCopyMemory(pDst, pSrc, Length) (导出函数.cpp:753)
    ├── IoAllocateMdl(pDst, Length, FALSE, FALSE, NULL)
    ├── MmBuildMdlForNonPagedPool(pMdl)     ← NonPaged 目标特化
    ├── pMdl->MdlFlags |= MDL_MAPPED_TO_SYSTEM_VA
    ├── MmMapLockedPagesSpecifyCache(KernelMode, MmNonCached, ...)
    ├── KeRaiseIrqlToDpcLevel()
    ├── RtlCopyMemory(pMapped, pSrc, Length)
    ├── KeLowerIrql
    ├── MmUnmapLockedPages
    └── IoFreeMdl
```

### 1.3 VT_Driver — KeStackAttachProcess / CR3 切换

```
用户态 NtDeviceIoControlFile(PROTO_READWRITE=0x9804 / PROTO_READWRITE_TX=0x9808)
    │
    ▼
Fake_NtDeviceIoControlFile (Main.cpp:207)
    ├── 检查 IoControlCode 范围 (0x9800~0x9828)
    ├── 验证 PsGetCurrentProcessId() == OwnPid
    └── ControlCenter()
         │
         ├─────────────────────────────────────
         │ PROTO_READWRITE (0x9804) — 直接模式
         │
         ▼
    Memory::ReadMemory / WriteMemory (Memory.cpp:128/169)
         ├── 校验 TargetProcess 有效
         ├── 分配 NonPagedPool 中转缓冲区
         ├── [读] KeStackAttachProcess(TargetProcess, &apc_state)
         │        → RtlCopyMemory(DriverBuffer, TargetAddr, Size)
         │        → KeUnstackDetachProcess
         │        → RtlCopyMemory(UserBuffer, DriverBuffer, Size)
         ├── [写] RtlCopyMemory(DriverBuffer, UserBuffer, Size)
         │        → KeStackAttachProcess(TargetProcess, &apc_state)
         │        → RtlCopyMemory(TargetAddr, DriverBuffer, Size)
         │        → KeUnstackDetachProcess
         └── ExFreePool(DriverBuffer)
         │
         ├─────────────────────────────────────
         │ PROTO_READWRITE_TX (0x9808) — 系统线程模式
         │
         ▼
    Memory::SystemReadWrite (Memory.cpp:10)
         ├── KeInitializeEvent(&kEvent, SynchronizationEvent, FALSE)
         ├── PsCreateSystemThread → SystemReadMemory / SystemWriteMemory
         ├── ZwClose(hThread)
         └── KeWaitForSingleObject(&kEvent, ...) ← 同步等待线程完成
              │
              ▼
         SystemReadMemory (Memory.cpp:34) / SystemWriteMemory (:81)
              ├── 分配 NonPagedPool 中转缓冲区
              ├── __readcr3()  → 保存当前 CR3
              ├── TargetProcess->DirectoryTableBase[0] → 获取目标 CR3
              ├── Utils::WriteProtectOff()
              │    ├── KeRaiseIrqlToDpcLevel()
              │    ├── __readcr0() → cr0 &= ~0x10000  ← 清除 WP 位
              │    ├── __writecr0(cr0)
              │    └── _disable()  ← 关中断
              ├── __writecr3(ulPDT)  ← 切换到目标进程地址空间
              ├── MmIsAddressValid 检查
              ├── RtlCopyMemory 实际读写
              ├── __writecr3(ulOldCr3)  ← 恢复原 CR3
              ├── Utils::WriteProtectOn(irql) → 恢复 WP 位 + 开中断 + 降 IRQL
              ├── ExFreePool
              ├── KeSetEvent(&kEvent, 0, TRUE)  ← 通知主线程
              └── PsTerminateSystemThread(status)
```

---

## 2. 关键数据结构与依赖

### 2.1 NinDriver

| 结构/对象 | 位置 | 用途 | 生命周期 |
|-----------|------|------|----------|
| `Offsets::MapAddress` | Export.cpp:283 动态分配 | PTE 窗口页——所有物理内存访问都通过此页 | 驱动加载时分配，永不释放 |
| `Offsets::PteOffsets` | Export.cpp:276 从 MmGetVirtualForPhysical 提取 | PTE 自引用基址 | 系统常量，读取后不变 |
| `Offsets::PfnOffsets` | Export.cpp:279-280 | PFN 数据库基址 | 系统常量 |
| `KSPIN_LOCK MapLock` (×2) | Memory.cpp:111, :145 | 保护 PTE 修改的自旋锁（Read/Write 各一个） | 函数 static，永久存在 |
| `DataStruct` | Driver.h | IOCTL 通信结构（包含 CR3/地址/缓冲区指针） | 每次 IOCTL 调用 |
| 页表项 (PTE/PDE/PDPTE/PML4E) | 物理内存 | 四级地址转换的中间产物 | 由 OS 管理 |

**Fact**: CR3 作为参数由用户态传入（用户态需先调用 IOCTL_GETPROCESSCR3 获取），驱动不维护进程引用计数。

### 2.2 GsDriver

| 结构/对象 | 位置 | 用途 | 生命周期 |
|-----------|------|------|----------|
| `PEPROCESS pProcess` | 通讯回调.cpp:265 | 目标进程对象——通过 `PsLookupProcessByProcessId` 获取 | 每次命令调用，调用后 ObDereferenceObject |
| `HANDLE hProcess` | 通讯回调.cpp:267 | 目标进程句柄（Type 2 需要）——通过 `ObOpenObjectByPointer` 获取 | 每次命令调用，调用后 ObCloseHandle |
| `PMDL lpMemoryDescriptorList` | 通讯回调.cpp:356 | MDL 描述符——描述目标虚拟地址的物理页映射 | 单次强写操作内创建销毁 |
| `KAPC_STATE ApcState` | 通讯回调.cpp:353 | APC 状态——KeStackAttachProcess 的上下文 | 栈变量，Attach/Detach 配对 |
| `DynamicData (DYNDATA)` | 全局 | 外壳传入的运行时偏移和版本信息 | 驱动加载后永久存在 |
| `READ_WRITE_MEMORY_BUFFER` | 通讯回调.cpp:253 | 通信参数结构 | 注册表回调中临时 |

**Fact**: GsDriver 不需要用户态提前获取 CR3——MmCopyVirtualMemory 内部通过 EPROCESS 自动处理地址空间切换。

### 2.3 VT_Driver

| 结构/对象 | 位置 | 用途 | 生命周期 |
|-----------|------|------|----------|
| `Memory::TargetProcess` | Memory.h / Memory.cpp:7 | 全局静态 PEPROCESS，通过 PROTO_SET_PROCESS 设置 | 设置后持续存在直到下次设置 |
| `Memory::kEvent` | Memory.cpp:8 | 同步事件——系统线程完成时通知主线程 | 每次 SystemReadWrite 调用重新初始化 |
| `COPY_MEMORY` | Memory.h:4 | 通信结构 {address, value, size, type} | 每次调用从 InputBuffer 拷贝 |
| `KAPC_STATE apc_state` | Memory.cpp:138 | 直接模式的进程附加状态 | 栈变量 |
| `PVOID DriverBuffer` | Memory.cpp:136 | NonPagedPool 中转缓冲区 | 每次调用分配释放 |

**Fact**: VT_Driver 使用全局 `TargetProcess` 静态变量，两步操作：先 `SetTargetProcess`（PROTO_SET_PROCESS），再读写。`TargetProcess` 通过 `PsLookupProcessByProcessId` 获取但**未 ObDereferenceObject**——这是一个引用计数泄漏。

---

## 3. 入口到读写的完整路径对比

### 3.1 入口方式

| 维度 | NinDriver | GsDriver | VT_Driver |
|------|-----------|----------|-----------|
| **通信通道** | 标准 IOCTL (DeviceIoControl) | 注册表回调 (RegSetValueEx) | 钩取 NtDeviceIoControlFile |
| **入口函数** | DispatchIoctl | RegisterNotify | Fake_NtDeviceIoControlFile → ControlCenter |
| **命令码** | 0xF62(读) / 0xB8D(写) | Type='0006' + ReadWriteType 0/1/2 | 0x9804(直接) / 0x9808(系统线程) |
| **调用上下文** | 调用进程线程 (PASSIVE_LEVEL) | 注册表回调上下文 (PASSIVE_LEVEL) | 调用进程线程 (系统调用返回路径) |
| **参数传递** | METHOD_BUFFERED SystemBuffer | PreSetValueInfo->Data | InputBuffer 指针 |
| **身份验证** | 无 (任何人可调用) | UserVerify 标志门禁 | OwnPid 进程白名单 |

### 3.2 地址空间切换

| 维度 | NinDriver | GsDriver | VT_Driver (直接) | VT_Driver (系统线程) |
|------|-----------|----------|------------------|---------------------|
| **切换方式** | 不切换——在物理地址层操作 | MmCopyVirtualMemory 内部处理 / KeStackAttachProcess | KeStackAttachProcess | __writecr3(目标 CR3) |
| **切换级别** | N/A (物理层绕过) | 内核 API 级 | 内核 API 级 | 硬件寄存器级 |
| **IRQL 要求** | DISPATCH_LEVEL (SpinLock) | PASSIVE_LEVEL | PASSIVE_LEVEL | DPC_LEVEL (手动提升) |
| **中断状态** | 中断可用(SpinLock 自动) | 中断可用 | 中断可用 | 中断关闭 (_disable) |
| **WP 位修改** | 否 | 否 | 否 | 是 (清除 CR0.WP) |

### 3.3 地址转换

| 维度 | NinDriver | GsDriver | VT_Driver |
|------|-----------|----------|-----------|
| **VA→PA 转换** | 自行实现四级页表遍历 | MmCopyVirtualMemory 内部 | 硬件 MMU 自动（通过 CR3 切换） |
| **转换代码量** | ~50 行手写页表遍历 | 0（内核 API 封装） | 0（CPU 硬件完成） |
| **大页支持** | 显式处理 2MB/1GB 大页 | 内核自动处理 | 内核/CPU 自动处理 |
| **转换开销** | 每页 4 次物理内存读取 | 内核优化路径 | 无额外开销 |

### 3.4 实际读写操作

| 维度 | NinDriver | GsDriver (Type 0/1) | GsDriver (Type 2) | VT_Driver |
|------|-----------|---------------------|--------------------|-----------|
| **读写原语** | PTE 修改 + invlpg + RtlCopyMemory | MmCopyVirtualMemory | MDL 映射 + RtlCopyMemory | RtlCopyMemory (直接在目标地址空间) |
| **操作粒度** | 单页 (不跨页) | 任意大小 | ≤ PAGE_SIZE | 任意大小 |
| **绕过只读** | 物理层操作，天然绕过 | 不能 (UserMode 检查) | 能 (MDL 重映射) | 天然绕过 (WP 位关闭) |
| **中转缓冲区** | 无——直接写到用户态 Buffer | 无——MmCopyVirtualMemory 直接传输 | 有 (WriteData 临时缓冲区) | 有 (DriverBuffer NonPagedPool) |

---

## 4. 资源 Ownership 与生命周期关键点

### 4.1 NinDriver

```
生命周期图：

[驱动加载]
    │
    ├── MmAllocateIndependentPages(MapAddress)  ← 分配窗口页，永不释放
    │
    ├── 初始化 Offsets::PteOffsets / PfnOffsets  ← 一次性读取，永久有效
    │
    │   ┌── [每次 IOCTL_GETPROCESSCR3]
    │   │     └── PFN 扫描 → 返回 CR3 值（不持有任何引用）
    │   │
    │   ├── [每次 IOCTL_READMEMORY / WRITEMEMORY]
    │   │     ├── SpinLock acquire (DISPATCH_LEVEL)
    │   │     ├── PTE 修改 → invlpg → RtlCopyMemory → PTE 恢复
    │   │     └── SpinLock release
    │   └── (循环)
    │
[系统重启]  ← 唯一释放时机
```

**关键 Ownership 点**：
- **MapAddress 永不释放**：`MmAllocateIndependentPages` 分配的页不在标准 VAD 跟踪中，没有 DriverUnload，只能重启回收
- **CR3 无引用计数**：GetProcessDirectoryBase 返回物理地址值，不持有 EPROCESS 引用。如果目标进程退出，CR3 可能被回收，后续使用会导致读取垃圾数据或蓝屏
- **SpinLock 是 static**：生命周期等同驱动，不存在分配/释放问题

### 4.2 GsDriver

```
生命周期图：

[每次 '0006' 命令]
    │
    ├── OpenProcessEx(hProcessId, &pProcess, &hProcess)
    │     ├── PsLookupProcessByProcessId → EPROCESS (引用 +1)
    │     ├── ObOpenObjectByPointer → HANDLE
    │     └── ObDereferenceObject → 引用 -1（但 HANDLE 仍然有效）
    │
    ├── [Type 0/1] ZwCopyVirtualMemory
    │     └── MmCopyVirtualMemory 内部管理所有资源
    │
    ├── [Type 2] MDL 强写
    │     ├── SetPreviousMode(KernelMode)     ← 修改 KTHREAD.PreviousMode
    │     ├── KeStackAttachProcess             ← APC 队列切换
    │     ├── MmCreateMdl                      ← 分配 MDL
    │     ├── MmProbeAndLockPages              ← 锁定物理页
    │     ├── MmMapLockedPagesSpecifyCache     ← 内核地址映射
    │     ├── RtlCopyMemoryEx                  ← 写入
    │     ├── MmUnmapLockedPages               ← 解除映射
    │     ├── MmUnlockPages                    ← 解锁物理页
    │     ├── IoFreeMdl                        ← 释放 MDL
    │     ├── KeUnstackDetachProcess            ← 恢复 APC 队列
    │     └── SetPreviousMode(OldMode)          ← 恢复 PreviousMode
    │
    └── ObCloseHandle(hProcess)                 ← 关闭句柄
```

**关键 Ownership 点**：
- **EPROCESS 引用立即释放**：`ObDereferenceObject` 在 `OpenProcessEx` 内部调用，但后续 MmCopyVirtualMemory 仍使用 `pProcess` 指针——这在实际中是安全的，因为调用在同一上下文中完成，进程不会在此期间退出（Fact: 注册表回调在 PASSIVE_LEVEL 同步执行）
- **MDL 资源严格配对**：Create → Lock → Map → Unmap → Unlock → Free，六步完整配对
- **PreviousMode 临时修改**：SetPreviousMode(KernelMode) 绕过地址检查，完成后恢复——如果中间发生异常未恢复，会留下安全隐患
- **hProcess 句柄**：仅 Type 2 打开，使用后关闭

### 4.3 VT_Driver

```
生命周期图：

[PROTO_SET_PROCESS]
    │
    ├── PsLookupProcessByProcessId → Memory::TargetProcess (全局静态)
    │   ⚠ 未调用 ObDereferenceObject — 引用计数泄漏
    │
    └── 后续所有 READWRITE / READWRITE_TX 复用此指针

[PROTO_READWRITE — 直接模式]
    │
    ├── ExAllocatePool(NonPagedPool, size)     ← 中转缓冲区
    ├── KeStackAttachProcess / KeUnstackDetachProcess  ← 标准配对
    └── ExFreePool                              ← 释放中转缓冲区

[PROTO_READWRITE_TX — 系统线程模式]
    │
    ├── KeInitializeEvent                       ← 每次重新初始化
    ├── PsCreateSystemThread → 新线程
    │     ├── ExAllocatePool                    ← 中转缓冲区
    │     ├── __readcr3() → __writecr3(target)  ← CR3 切换
    │     ├── RtlCopyMemory
    │     ├── __writecr3(old)                   ← CR3 恢复
    │     ├── ExFreePool
    │     ├── KeSetEvent                        ← 通知完成
    │     └── PsTerminateSystemThread           ← 线程退出
    │
    └── KeWaitForSingleObject                   ← 等待线程完成
```

**关键 Ownership 点**：
- **TargetProcess 引用泄漏** (Fact)：`SetTargetProcess` 调用 `PsLookupProcessByProcessId` 增加引用计数，但从不调用 `ObDereferenceObject`。多次调用 `SetTargetProcess` 会持续泄漏。每次泄漏意味着目标进程的 EPROCESS 结构即使进程退出也不会被回收
- **系统线程生命周期**：线程创建后立即关闭句柄 (`ZwClose(hThread)`)，但通过 `kEvent` 等待线程完成——如果线程意外崩溃，`KeWaitForSingleObject` 会永久等待
- **CR3 切换窗口**：在 `__writecr3(target)` 和 `__writecr3(old)` 之间，当前线程运行在目标进程的地址空间中，中断被关闭。如果此窗口内发生 NMI（不可屏蔽中断），CR3 处于错误状态
- **DriverBuffer 每次分配释放**：无泄漏风险，但 `ExAllocatePool` 不检查 NULL（如果分配失败会蓝屏）

---

## 5. 脆弱点与故障模式分析

### 5.1 NinDriver 的脆弱点

| 脆弱点 | 严重程度 | 触发条件 | 后果 |
|--------|----------|----------|------|
| **Read/Write 独立 SpinLock** | 中 | 并发调用 Read 和 Write | 两个操作同时修改 MapAddress 的 PTE，数据损坏或蓝屏 |
| **CR3 过期** | 高 | 用户态获取 CR3 后目标进程退出 | 页表遍历读取已回收的物理页，数据垃圾或蓝屏 |
| **36 位物理地址限制** | 低 | 系统物理内存 > 64GB | 物理地址掩码 `0xfffffffffull` 截断高位，转换错误 |
| **PFN 0x30 大小假设** | 中 | Windows 版本更新改变 MMPFN 结构 | GetProcessDirectoryBase 扫描错位，返回错误 CR3 |
| **EPROCESS 解密公式** | 中 | 不同 Windows 10 子版本 | 解密结果不正确，PID 匹配失败 |
| **特征码偏移硬编码** | 中 | ntoskrnl 编译变化 | InitSystemOffsets 中 `+3`, `+0x22` 等偏移失效 |
| **用户态 Buffer 无 Probe** | 中 | 恶意或错误的用户态指针 | ReadPhysicalMemory 的 __try 可能无法捕获物理层异常 |

### 5.2 GsDriver 的脆弱点

| 脆弱点 | 严重程度 | 触发条件 | 后果 |
|--------|----------|----------|------|
| **MmCopyVirtualMemory 被 Hook** | 低 | 反作弊内核监控 | 读写被拦截或返回错误数据 |
| **MmProbeAndLockPages 异常** | 中 | MDL 锁定无效页 | 异常未被 __try 包裹——蓝屏 |
| **PreviousMode 修改竞态** | 低 | 多线程同时触发 Type 2 | SetPreviousMode 是 per-thread 的，理论上安全 |
| **注册表回调时序** | 低 | 进程退出与读写竞态 | ObDereferenceObject 后 pProcess 仍在使用（概率极低） |
| **Type 2 大小限制** | 低 | NumberOfBytes > PAGE_SIZE | 返回 ERROR_超出读写字节，非崩溃 |
| **EPROCESS 偏移硬编码** | 中 | 新 Windows 版本 | 偏移查表需更新 |

### 5.3 VT_Driver 的脆弱点

| 脆弱点 | 严重程度 | 触发条件 | 后果 |
|--------|----------|----------|------|
| **TargetProcess 引用泄漏** | 中 | 每次 SetTargetProcess 调用 | EPROCESS 结构永不释放，内存泄漏 |
| **DriverBuffer 分配失败** | 高 | 内存不足 | ExAllocatePool 返回 NULL，后续 RtlCopyMemory 蓝屏 |
| **CR3 切换 + NMI** | 高 | 系统线程模式下 NMI 中断 | NMI 在错误地址空间执行，蓝屏 |
| **CR3 切换 + 进程退出** | 高 | 目标进程在 CR3 切换期间退出 | 地址空间无效，MmIsAddressValid 可能不可靠 |
| **kEvent 无超时** | 中 | 系统线程崩溃 | KeWaitForSingleObject 永久阻塞 |
| **MmIsAddressValid 不可靠** | 中 | 页面被换出或竞态条件 | 检查通过但实际访问时缺页 |
| **WP 位清除窗口** | 低 | 多核竞态 | 其他 CPU 仍然受 WP 保护，不影响 |

---

## 6. 跨方案对比矩阵

### 6.1 复杂度

| 维度 | NinDriver | GsDriver | VT_Driver |
|------|-----------|----------|-----------|
| **代码行数（内存相关）** | ~255 行 (Memory.cpp) | ~140 行 (通讯回调.cpp 中) | ~238 行 (Memory.cpp) |
| **概念复杂度** | 高（PTE 操作 + 四级页表 + PFN 扫描） | 低（标准 API 调用 + MDL 模板） | 中（KeStackAttach 标准 + CR3 切换非常规） |
| **前置依赖** | PteOffsets, PfnOffsets, MapAddress 初始化 | DynamicData 偏移表 + MmCopyVirtualMemory 动态解析 | TargetProcess 预设 |
| **版本适配工作** | 中（特征码偏移可能变化） | 高（EPROCESS 偏移查表，每版本需更新） | 低（使用标准 API，但系统线程模式依赖 EPROCESS 偏移） |

### 6.2 隐蔽性（检测面）

| 检测手段 | NinDriver | GsDriver | VT_Driver (直接) | VT_Driver (系统线程) |
|----------|-----------|----------|------------------|---------------------|
| **API Hook 拦截** | ✅ 完全绕过——不调用任何 Mm/Ke API | ❌ MmCopyVirtualMemory 可被 Hook | ❌ KeStackAttachProcess 可被检测 | ✅ 绕过——直接 __writecr3 |
| **内核调用栈分析** | ✅ 无系统 API 调用栈 | ❌ 调用栈包含 MmCopyVirtualMemory | ❌ 调用栈包含 KeStackAttach | ⚠ 系统线程调用栈较简洁 |
| **PTE 修改检测** | ⚠ 临时修改可被 hypervisor 检测 | ✅ 不修改 PTE | ✅ 不修改 PTE | ✅ 不修改 PTE |
| **CR3 切换检测** | ✅ 不切换 CR3 | ✅ 不切换 CR3 | ✅ 不切换 CR3 | ⚠ MOV CR3 可被 hypervisor 拦截 |
| **ETW/系统日志** | ✅ 无日志 | ⚠ 注册表回调可被审计 | ⚠ NtDeviceIoControlFile 调用可被审计 | 同左 |
| **物理内存扫描** | ⚠ PFN 遍历特征明显 | ✅ 不涉及物理内存 | ✅ 不涉及物理内存 | ✅ 不涉及物理内存 |
| **IRQL 异常检测** | ⚠ DISPATCH_LEVEL 频繁 | ✅ PASSIVE_LEVEL | ✅ PASSIVE_LEVEL | ⚠ DPC_LEVEL + 关中断 |

### 6.3 稳定性

| 维度 | NinDriver | GsDriver | VT_Driver (直接) | VT_Driver (系统线程) |
|------|-----------|----------|------------------|---------------------|
| **异常保护** | ✅ __try/__except 全覆盖 | ⚠ MDL 路径部分缺失 | ✅ __try/__except | ✅ __try/__except |
| **资源泄漏风险** | 低 | 低（配对严格） | 中（TargetProcess 泄漏） | 中（TargetProcess + 线程风险） |
| **蓝屏风险** | 中（PTE 操作直接影响 TLB） | 低（标准 API 有内部保护） | 低（KeStackAttach 成熟稳定） | 高（CR3 切换 + 关中断） |
| **并发安全** | ⚠ 读写独立锁有竞态 | ✅ 每次调用独立 | ✅ 每次调用独立 | ⚠ 全局 TargetProcess + kEvent |
| **进程退出容错** | ❌ CR3 过期致命 | ✅ MmCopyVirtualMemory 内部处理 | ⚠ MmIsAddressValid 不完全可靠 | ❌ CR3 切换到已退出进程致命 |

### 6.4 性能

| 维度 | NinDriver | GsDriver | VT_Driver (直接) | VT_Driver (系统线程) |
|------|-----------|----------|------------------|---------------------|
| **每页开销** | SpinLock + PTE 改写 + invlpg + RtlCopy | MmCopyVirtualMemory 内部（优化路径） | Attach/Detach + RtlCopy | 线程创建 + CR3 切换 + RtlCopy + 事件同步 |
| **跨页代价** | 每页重走完整链（4 次物理读 + PTE 操作） | 内核一次性处理 | 一次 Attach 处理所有数据 | 一次 CR3 切换处理所有数据 |
| **大块读写** | 差——每页独立操作 | 好——MmCopyVirtualMemory 优化 | 好——单次 Attach | 差——线程创建/销毁开销大 |
| **CR3 获取开销** | 极差——全 PFN 数据库扫描 | 无 (API 级) | 无 (EPROCESS 直接读取) | 无 (EPROCESS 直接读取) |
| **IRQL 影响** | 阻塞调度（DISPATCH_LEVEL） | 不影响调度 | 不影响调度 | 阻塞调度（DPC + 关中断） |

---

## 7. 设计决策的深层逻辑

### 7.1 NinDriver：为什么选择 PTE 自映射？

**Fact**: NinDriver 是一个 ~880 行的小型项目，功能单一（内存读写 + 进程查询）。

**Inference**: PTE 自映射的选择可能基于以下考量：
1. **完全自治**：不依赖任何内核 API（MmCopyVirtualMemory、KeStackAttachProcess），因此不受任何 API Hook 影响
2. **最小依赖面**：只需要 PteOffsets 和一个 MapAddress 页，不需要 EPROCESS 偏移表（CR3 通过 PFN 扫描获取）
3. **教学意义**：代码清晰展示了"虚拟地址 → PTE → 物理页"的映射关系

**代价**：这是三个方案中**实现复杂度最高、性能最差**的方案。每读一个字节都需要经过完整的四级页表遍历（每级一次物理内存读取），然后再做一次 PTE 修改 + TLB 刷新。

### 7.2 GsDriver：为什么选择 MmCopyVirtualMemory + MDL？

**Fact**: GsDriver 是一个完整框架（~7800 行），内存读写只是 22+ 功能之一。

**Inference**: 这个选择可能基于以下考量：
1. **开发效率**：MmCopyVirtualMemory 是标准做法，一行调用完成全部工作
2. **稳定性优先**：MmCopyVirtualMemory 内部有完善的错误处理和页面管理
3. **MDL 补充**：对于需要绕过只读保护的场景（如 hook 安装），MDL 重映射是成熟的解决方案
4. **检测面可接受**：GsDriver 的反检测策略集中在通信层（注册表回调 + 系统驱动跳板），而不是内存读写层

**代价**：MmCopyVirtualMemory 是**最容易被检测**的内存读写方式——反作弊软件可以 Hook 此函数或监控其调用栈。

### 7.3 VT_Driver：为什么提供两种模式？

**Fact**: VT_Driver 同时实现了 KeStackAttachProcess（直接模式）和 CR3 切换（系统线程模式），通过不同 IOCTL 调用。

**Inference**: 双模式设计可能基于以下考量：
1. **直接模式** (PROTO_READWRITE)：简单、快速、适合常规读写
2. **系统线程模式** (PROTO_READWRITE_TX)：绕过 KeStackAttachProcess 的调用栈检测，适合对抗更严格的反作弊（如 TP/TX 系列）——IOCTL 名称后缀 `_TX` 暗示针对腾讯反作弊
3. **CR3 切换在系统线程中执行**：系统线程不属于任何用户进程，调用栈更干净

**代价**：系统线程模式的**线程创建/销毁开销**使其不适合高频读写，而**关中断 + CR3 切换**的窗口如果遇到 NMI 会蓝屏。

---

## 8. 事实与推断清单

### Facts（代码直接证据）

1. NinDriver 的 ReadPhysicalAddress 和 WritePhysicalAddress 各自使用独立的 `static KSPIN_LOCK` (Memory.cpp:111, :145)
2. NinDriver 的 ReadPhysicalMemory 直接使用用户态传入的 `Buffer` 指针，无 ProbeForRead/ProbeForWrite (Driver.cpp:47-48)
3. GsDriver 的 Type 2 MDL 强写限制为 `<= PAGE_SIZE` (通讯回调.cpp:297-302)
4. GsDriver 的 ZwCopyVirtualMemory 内部调用 `MmCopyVirtualMemory`，通过 `RtlGetSystemFun(L"MmCopyVirtualMemory")` 动态解析 (导出函数.cpp:197-199)
5. VT_Driver 的 `SetTargetProcess` 调用 `PsLookupProcessByProcessId` 但无对应的 `ObDereferenceObject` (Memory.cpp:215)
6. VT_Driver 的 `SystemReadMemory` 关闭中断 (`_disable()`) 并清除 CR0.WP 位 (Utils.cpp:49-53)
7. VT_Driver 的 `SystemReadWrite` 使用 `KeWaitForSingleObject` 无超时参数等待系统线程 (Memory.cpp:29)
8. NinDriver 的 `GetPhyicalAddress` 物理地址掩码为 `(~0xfull << 8) & 0xfffffffffull`，支持最大 36 位物理地址 (Memory.cpp:78,86,90,96)
9. GsDriver 通讯回调中 Type 2 在 KeStackAttachProcess 之前调用 `SetPreviousMode(KernelMode)` (通讯回调.cpp:350)

### Inferences（推断，需进一步验证）

1. NinDriver 读写独立 SpinLock 在实际使用中可能不会触发竞态，因为用户态通常是单线程串行调用——但代码结构上确实存在并发风险
2. VT_Driver 的 `_TX` 后缀暗示系统线程模式针对腾讯系反作弊，但无直接文档证据
3. GsDriver 的 `ObDereferenceObject` 在 `OpenProcessEx` 内部调用后仍使用 `pProcess` 指针，在同步回调上下文中应当安全，但严格来说违反了引用计数语义
4. NinDriver 的 PFN 0x30 字节大小在 Windows 10 1507-22H2 范围内应当一致，但 Windows 11 新版本可能变化
5. VT_Driver 的 CR3 切换模式可能是整个方案中最难被用户态反作弊检测到的——系统线程不属于任何被监控进程，且 CR3 切换不经过任何可 Hook 的 API

### Open Questions

1. NinDriver 的 EPROCESS 解密公式 `(value | 0xF000000000000000) >> 13 | 0xFFFF000000000000` 的数学推导来源？是否所有 Windows 10 版本通用？
2. VT_Driver 的 `Memory::TargetProcess` 引用泄漏是有意设计（防止 EPROCESS 被回收）还是 bug？
3. GsDriver 的 `MmProbeAndLockPages` 在 Type 2 路径中如果目标页已被换出，是否会阻塞？（Fact: MmProbeAndLockPages 在 PASSIVE_LEVEL 下可以阻塞等待页面换入）
4. VT_Driver 系统线程模式下 `KeSetEvent` 的 `Wait = TRUE` 参数意味着什么？（Fact: 这意味着 KeSetEvent 和后续 PsTerminateSystemThread 之间不允许被抢占——但 PsTerminateSystemThread 是紧接的下一条调用，所以这个 Wait=TRUE 实际上没有意义）
5. 如果反作弊使用 hypervisor 监控 MOV CR3 指令，VT_Driver 的系统线程模式是否完全暴露？（Inference: 是的，EPT VIOLATION 或 CR3 写入拦截可以检测到）

---

## 9. 方案选择建议

> 以下是基于代码分析的**有依据的建议**，不是泛泛的"谁更好"。

### 何时选择 NinDriver 方案（PTE 自映射）

- 目标环境有 API 级 Hook 全覆盖（MmCopyVirtualMemory、KeStackAttachProcess 等均被监控）
- 不需要高频大量读写（PFN 扫描 + 每页 4 次物理读取的开销可以接受）
- 不需要写入只读页（物理层操作天然绕过页面保护）
- 可以接受较高的版本适配风险

### 何时选择 GsDriver 方案（MmCopyVirtualMemory + MDL）

- 开发效率和稳定性优先
- 内存读写不是核心对抗点（反检测策略在其他层面）
- 需要可靠的大块数据传输
- 需要兼容多个 Windows 版本（MmCopyVirtualMemory 接口稳定）

### 何时选择 VT_Driver 方案（KeStackAttach + CR3 切换）

- 需要兼顾简单场景（直接模式）和高对抗场景（系统线程模式）
- 目标是对抗用户态反作弊而非内核 hypervisor
- 可以接受系统线程模式的性能开销
- 对 CR3 切换 + NMI 蓝屏风险有足够的容忍度

### 综合评价

| 维度 | 最优方案 | 依据 |
|------|----------|------|
| **隐蔽性** | NinDriver (物理层绕过) | 不调用任何可 Hook 的 API |
| **稳定性** | GsDriver (标准 API) | MmCopyVirtualMemory 内部有完善错误处理 |
| **性能** | GsDriver (Type 0/1) | 内核优化路径，无额外开销 |
| **只读绕过** | NinDriver / VT_Driver(系统线程) | 物理层操作或 WP 位关闭天然绕过 |
| **开发成本** | GsDriver | 一行 API 调用 vs 255 行手写页表操作 |
| **抗 Hypervisor** | GsDriver | 不涉及 PTE 修改或 CR3 切换 |
| **版本兼容** | GsDriver (Type 0/1) | MmCopyVirtualMemory 接口自 Windows XP 起稳定 |

---

## 10. 补充：RtlSuperCopyMemory — GsDriver 的内核代码页写入

GsDriver 的内存读写体系不止 `'0006'` 命令。`RtlSuperCopyMemory` (导出函数.cpp:753) 用于向**内核非分页代码区**写入数据（如安装回调跳板），其调用链：

```
RtlSuperCopyMemory(pDst, pSrc, Length)
    ├── IoAllocateMdl(pDst, Length)            ← 为内核地址创建 MDL
    ├── MmBuildMdlForNonPagedPool(pMdl)        ← 非分页池专用（不需要 ProbeAndLock）
    ├── pMdl->MdlFlags |= MDL_MAPPED_TO_SYSTEM_VA  ← 标记已映射
    ├── MmMapLockedPagesSpecifyCache(KernelMode, MmNonCached, ...)
    │    ← MmNonCached 确保不走 CPU 缓存，避免缓存一致性问题
    ├── KeRaiseIrqlToDpcLevel()                ← 提升 IRQL 防止被抢占
    ├── RtlCopyMemory(pMapped, pSrc, Length)   ← 通过新映射写入
    ├── KeLowerIrql
    ├── MmUnmapLockedPages
    └── IoFreeMdl
```

**与 Type 2 MDL 强写的区别**：
- Type 2 针对**用户态只读页**（需要 Attach 进程 + ProbeAndLockPages）
- RtlSuperCopyMemory 针对**内核代码页**（使用 MmBuildMdlForNonPagedPool，不需要 Attach）
- 两者都利用 MDL 重映射绕过写保护，但目标地址空间不同

---

## 11. 完整调用链代码位置索引

### NinDriver

| 函数 | 文件 | 行号 |
|------|------|------|
| DispatchIoctl | Driver.cpp | 11 |
| ReadPhysicalMemory | 内存管理/Memory.cpp | 171 |
| WritePhysicalMemory | 内存管理/Memory.cpp | 214 |
| GetPhyicalAddress | 内存管理/Memory.cpp | 54 |
| ReadPhysicalAddress | 内存管理/Memory.cpp | 103 |
| WritePhysicalAddress | 内存管理/Memory.cpp | 137 |
| GetProcessDirectoryBase | 内存管理/Memory.cpp | 3 |
| InitSystemOffsets | 导出函数/Export.cpp | 246 |

### GsDriver

| 函数 | 文件 | 行号 |
|------|------|------|
| RegisterNotify | 驱动核心/通讯回调.cpp | 24 |
| '0006' 命令处理 | 驱动核心/通讯回调.cpp | 251-396 |
| ZwCopyVirtualMemory | 驱动核心/导出函数.cpp | 189 |
| RtlSuperCopyMemory | 驱动核心/导出函数.cpp | 753 |
| OpenProcessEx | 驱动核心/通讯回调.cpp | 5 |
| SetPreviousMode | 驱动核心/导出函数.cpp | 891 |

### VT_Driver

| 函数 | 文件 | 行号 |
|------|------|------|
| Fake_NtDeviceIoControlFile | VT_Driver/Main.cpp | 207 |
| ControlCenter | VT_Driver/Main.cpp | 38 |
| ReadMemory | VT_Driver/Memory.cpp | 128 |
| WriteMemory | VT_Driver/Memory.cpp | 169 |
| SystemReadWrite | VT_Driver/Memory.cpp | 10 |
| SystemReadMemory | VT_Driver/Memory.cpp | 34 |
| SystemWriteMemory | VT_Driver/Memory.cpp | 81 |
| SetTargetProcess | VT_Driver/Memory.cpp | 210 |
| WriteProtectOff | VT_Driver/Utils.cpp | 45 |
| WriteProtectOn | VT_Driver/Utils.cpp | 60 |
