# NinDriver 结构分析

> Stage 1 产出 | 生成时间：2026-03-28

## 一句话总结

NinDriver 是一个 ~880 行的单驱动内核内存读写工具：通过**自重映射**隐藏自身，通过 **PTE 自映射**直接操作物理内存，通过 **PFN 数据库扫描**获取目标进程 CR3，对外暴露 6 个 IOCTL 命令供用户态客户端调用。

---

## 全局调用图

```
[系统加载]
    │
    ▼
DriverEntry (Driver.cpp:133)           ← 一次性入口，自毁式
    ├── ForceDeleteFile()              ← 删除磁盘上的 .sys 文件
    ├── InitSystemOffsets()            ← 动态解析所有运行时偏移
    ├── MmAllocateIndependentPages()   ← 分配独立物理页 (特征码定位)
    ├── MmSetPageProtection(RWX)       ← 设置可执行权限
    ├── memcpy(新地址, 原镜像)          ← 复制驱动镜像
    ├── memset(新地址, 0, PAGE_SIZE)    ← 清除 PE 头
    ├── IoCreateDriver → MapEntry      ← 在新地址创建驱动
    └── return STATUS_UNSUCCESSFUL     ← 触发系统卸载原镜像
            │
            ▼
MapEntry (Driver.cpp:93)               ← 真正的入口，运行在重映射地址
    ├── IoCreateDevice()               ← 伪装为 NVIDIA PCI 设备
    ├── IoCreateSymbolicLink()
    └── 注册 IRP 分发函数 ──┐
                             │
                             ▼
DispatchIoctl (Driver.cpp:11)          ← IOCTL 分发核心
    ├── GETPROCESSPID     → GetProcessIdByName()
    ├── GETPROCESSCR3     → GetProcessDirectoryBase()
    ├── GETPROCESSMODULE  → GetProcessModuleByPhyical()
    ├── READMEMORY        → ReadPhysicalMemory()
    ├── WRITEMEMORY        → WritePhysicalMemory()
    └── GETKEYSTATE       → GetKeyStateFromKernel()
```

---

## 双阶段初始化（核心设计）

### Facts

NinDriver 的初始化分为两个阶段，这是整个驱动最关键的设计：

**阶段 1：DriverEntry（一次性，自毁）**

| 步骤 | 代码位置 | 操作 | 目的 |
|------|----------|------|------|
| 1 | Driver.cpp:141-145 | 计算 3 个 dispatch 函数相对于 `DriverStart` 的偏移 | 记录函数位置，供重映射后使用 |
| 2 | Driver.cpp:149-152 | 通过 `DriverSection`(LDR 表项) 获取驱动文件完整路径 | 为文件删除做准备 |
| 3 | Driver.cpp:153 | `ForceDeleteFile()` 删除 .sys 文件 | 消除磁盘痕迹 |
| 4 | Driver.cpp:160 | `InitSystemOffsets()` | 动态解析 EPROCESS 偏移和内存管理地址 |
| 5 | Driver.cpp:167 | `MmAllocateIndependentPages()` | 分配不被 MiGetPteAddress 跟踪的独立物理页 |
| 6 | Driver.cpp:174 | `MmSetPageProtection(PAGE_EXECUTE_READWRITE)` | 使新分配的内存可执行 |
| 7 | Driver.cpp:181 | `memcpy` 复制整个驱动镜像 | 在新地址建立完整副本 |
| 8 | Driver.cpp:182 | `memset` 清零第一页 | 抹除 PE 头，阻止内存扫描识别 |
| 9 | Driver.cpp:183 | `IoCreateDriver(NULL, MapEntry偏移)` | 在新地址空间创建匿名驱动对象 |
| 10 | Driver.cpp:191 | `return STATUS_UNSUCCESSFUL` | 让系统自动卸载原始驱动镜像 |

**阶段 2：MapEntry（持久，运行在重映射地址）**

| 步骤 | 代码位置 | 操作 |
|------|----------|------|
| 1 | Driver.cpp:101-103 | 创建设备名和符号链接（伪装为 `\\Device\\PCI#VEN_10DE&DEV_2684#...`） |
| 2 | Driver.cpp:105 | `IoCreateDevice()` 创建设备对象 |
| 3 | Driver.cpp:113 | `IoCreateSymbolicLink()` 暴露给用户态 |
| 4 | Driver.cpp:122-126 | 将 IRP 分发函数指向 `MapImage + 偏移` (重映射地址空间) |

### Inferences

- **偏移计算的精妙之处**：因为 `memcpy` 是整体复制，函数在新旧地址中的相对偏移不变。所以 `DriverEntry` 里提前算好偏移，`MapEntry` 里用 `MapImage + offset` 就能正确指向新地址中的函数。
- **IoCreateDriver(NULL, ...)** 传入 NULL 表示不注册驱动名——这意味着新驱动不出现在 `\Driver\` 对象目录中。
- **清零 PE 头** 是双保险：即使有工具扫描到了这块内存，也无法通过 `MZ`/`PE` 签名识别它是一个驱动。

### Open Questions

1. `memcpy` 复制后，原始镜像中的**全局变量**（如 `MapImage`、`pCreateDevice`）在新地址中的值是 DriverEntry 执行时设置的值——但这些是在复制**之后**还是**之前**设置的？
   - **事实**：`MapImage = MmAllocateIndependentPages(...)` 在 Driver.cpp:167，`memcpy` 在 :181。所以 `MapImage` 的值已经在全局变量中，会被一起复制到新地址。MapEntry 中的 `MapImage` 全局变量已包含正确的地址。✅
2. 原始驱动卸载后，DriverEntry 中的局部变量和栈帧被释放——但 MapEntry 此时是否已经在独立栈上运行？
   - **推断**：`IoCreateDriver` 是同步调用，MapEntry 在 DriverEntry 返回前已经完成执行。所以不存在栈帧竞争问题。

---

## IOCTL 分发层 (Driver.cpp:11-77)

### Facts

统一使用 `METHOD_BUFFERED`，通过 `SystemBuffer` 传递 `DataStruct` 结构体。

| IOCTL 码 | 宏名 | 调用目标 | 返回方式 |
|-----------|------|----------|----------|
| 0x9D7 | GETPROCESSPID | `GetProcessIdByName(Name)` | 覆写 OutputData 前 4 字节 |
| 0xE4B | GETPROCESSCR3 | `GetProcessDirectoryBase(Pid)` | 覆写 OutputData 前 8 字节 |
| 0xA1C | GETPROCESSMODULE | `GetProcessModuleByPhyical(Pid, Name, Buffer)` | 通过函数内部写入 Buffer |
| 0xF62 | READMEMORY | `ReadPhysicalMemory(Cr3, Addr, Buf, Len)` | 数据写入用户态 Buffer 指针 |
| 0xB8D | WRITEMEMORY | `WritePhysicalMemory(Cr3, Addr, Buf, Len)` | 从用户态 Buffer 读取写入 |
| 0xC3E | GETKEYSTATE | `GetKeyStateFromKernel(VirtualKey)` | 覆写 OutputData 前 2 字节 |

### Inferences

- READMEMORY/WRITEMEMORY 的 `pData->Buffer` 是**用户态指针**，直接在内核中使用。这能工作是因为当 IOCTL 在调用进程上下文中执行时，用户态地址空间是有效的。
- 没有对 `pData->Buffer` 做 `ProbeForRead`/`ProbeForWrite` 检查——这是一个**有意的简化**（或安全隐患），依赖调用者传入有效地址。

### Open Questions

- 如果用户态传入非法 `Buffer` 地址，`ReadPhysicalMemory` 内部的 `__try/__except` 能否捕获？答案取决于异常是发生在 `RtlCopyMemory(Buffer, ...)` 还是在物理地址映射阶段。

---

## 内存管理模块 (Memory.cpp) — 三层架构

### Facts

```
ReadPhysicalMemory / WritePhysicalMemory     ← 用户接口层 (跨页循环)
        │
        ▼
GetPhyicalAddress                             ← 地址转换层 (四级页表遍历)
        │
        ▼
ReadPhysicalAddress / WritePhysicalAddress    ← 物理内存访问层 (PTE 自映射)
```

#### 第 1 层：物理内存访问（PTE 自映射）

**核心原理**（Memory.cpp:103-135 / 137-169）：

```
┌─────────────────────────────────────────────────┐
│  Offsets::MapAddress — 一个预分配的"窗口"虚拟页    │
│  ┌─────────┐          ┌─────────┐                │
│  │ PTE     │ ──修改──→│物理页 X  │                │
│  │         │          │(目标)    │                │
│  └─────────┘          └─────────┘                │
│       ↓ invlpg                                   │
│  MapAddress 现在映射到物理页 X                     │
│  通过 MapAddress + 页内偏移 读写目标物理内存         │
│  ↓ 恢复原 PTE                                    │
└─────────────────────────────────────────────────┘
```

具体步骤：
1. 计算 `MapAddress` 对应的 PTE 地址：`PteOffsets + (MapAddress >> 12) * 8`
2. 保存原 PTE 值
3. 将 PTE 的物理页帧号替换为目标物理地址的页帧号（保留控制位）
4. `__invlpg` 刷新 TLB
5. 通过 `MapAddress + 页内偏移` 进行 `RtlCopyMemory`
6. 恢复原 PTE
7. 全程在 `KSPIN_LOCK` + `DISPATCH_LEVEL` 保护下

**关键约束**：单次操作不能跨页（检查 `PhysicalAddress >> 12 == (PhysicalAddress + Size - 1) >> 12`）。

#### 第 2 层：地址转换（四级页表遍历）

**GetPhyicalAddress** (Memory.cpp:54-101)：

```
CR3 → PML4E → PDPTE → PDE → PTE → 物理地址
       ↓         ↓       ↓      ↓
   [+8*idx]  [+8*idx] [+8*idx] [+8*idx]
```

- 每一级都通过 `ReadPhysicalAddress` 读取页表项（用物理内存读物理内存）
- 检查 Present 位（bit 0）
- **大页处理**：
  - PDE 的 bit 7 = 1 → 2MB 大页，直接计算物理地址
  - PTE level 的 bit 7 = 1 → 此处实际对应 PDPTE 的大页(1GB)，但代码中的命名有误（变量名是 `NewPteAddress`，实际对应 PDE 级别的下一级）
- 物理地址掩码：`(~0xfull << 8) & 0xfffffffffull` 提取 PFN（bits 12-35，支持 36 位物理地址）

#### 第 3 层：用户接口（跨页循环）

**ReadPhysicalMemory / WritePhysicalMemory** (Memory.cpp:171-255)：

- 参数校验：CR3、BaseAddress、Size 范围检查
- 循环处理：每次最多一页，`min(PAGE_SIZE - 页内偏移, 剩余大小)`
- 异常保护：`__try/__except(EXCEPTION_EXECUTE_HANDLER)`
- 对每个页片段：`GetPhyicalAddress` 转换 → `ReadPhysicalAddress/WritePhysicalAddress` 读写

### Inferences

- **Read 和 Write 使用独立的 `static KSPIN_LOCK`**（各自函数内部声明）——这意味着一个读操作和一个写操作可以并发修改同一个 `MapAddress` 的 PTE。这是一个**潜在的竞态条件**，但在实际使用中可能不会触发，因为通常是单线程调用。
- PTE 自映射的优势：避免使用 `MmMapIoSpace` 等被监控的 API，操作在 DISPATCH_LEVEL 执行也更隐蔽。
- 物理地址掩码只支持到 36 位（64GB），现代系统可能有更高物理地址。

---

## CR3 获取 — PFN 数据库扫描 (Memory.cpp:3-52)

### Facts

**GetProcessDirectoryBase** 的算法：

1. **计算 PXE（PML4 self-reference）基址**：
   ```
   g_PdeBase   = PteOffsets + ((PteOffsets >> 9) & 0x7FFFFFFFF8)
   g_PpeBase   = PteOffsets + ((g_PdeBase >> 9) & 0x7FFFFFFFF8)
   g_PxeBase   = PteOffsets + ((g_PpeBase >> 9) & 0x7FFFFFFFF8)
   Index       = (PteOffsets >> 39) - 0x1FFFE00
   Cr3PteBase  = Index * 8 + g_PxeBase
   ```
   这利用了 Windows 页表自引用结构来计算最顶层 PML4 entry 对应的 PTE 地址。

2. **遍历物理内存范围**（`MmGetPhysicalMemoryRanges`）
3. **扫描每个 PFN 条目**（每个 0x30 字节）：
   - `MMPFN[0]` (第一个 QWORD) != 0 且 != 1 → 有效条目
   - `MMPFN[+8]` (第二个 QWORD) == Cr3PteBase → 此 PFN 是某个进程的 CR3 页
   - 从 `MMPFN[0]` 解密 EPROCESS：`((value | 0xF000000000000000) >> 13) | 0xFFFF000000000000`
   - 验证地址有效 + `UniqueProcessId` 匹配

### Inferences

- 这个方法的核心思想：进程的 CR3 页在 PFN 数据库中有特殊标记——其 PTE 地址指向页表自引用的 PXE entry。通过反向查找这个特征，可以找到所有进程的 CR3。
- **不 break on first match**（循环继续到末尾）——最终返回最后一个匹配的 PFN。这可能是有意的：如果有多个匹配，取最后一个（可能是最新创建的 EPROCESS）。
- 性能代价：每次调用都要扫描整个物理内存的 PFN 数据库，对于大内存系统可能很慢。
- EPROCESS 解密公式是 Windows 10+ 特有的 PFN 加密方案。

### Open Questions

- PFN 数据库中 MMPFN 结构的第一个字段在不同 Windows 版本中含义是否一致？代码依赖 `0x30` 字节大小和前两个 QWORD 的布局。
- EPROCESS 解密公式 `(value | 0xF000000000000000) >> 13 | 0xFFFF000000000000` 的数学来源是什么？是否所有 Windows 10 版本通用？

---

## 导出函数模块 (Export.cpp) — 运行时基础设施

### Facts

#### InitSystemOffsets (Export.cpp:246-287) — 偏移解析引擎

| 偏移变量 | 来源函数 | 提取方式 | 用途 |
|----------|----------|----------|------|
| `UniqueProcessId` | `PsGetProcessId` | `*(ULONG*)(func + 3)` | EPROCESS.UniqueProcessId 偏移 |
| `ActiveProcessLinks` | 推导 | `UniqueProcessId + 8` | EPROCESS.ActiveProcessLinks 偏移 |
| `ImageFileName` | `PsGetProcessImageFileName` | `*(ULONG*)(func + 3)` | EPROCESS.ImageFileName 偏移 |
| `Wow64Process` | `PsGetProcessWow64Process` | `*(ULONG*)(func + 3)` | EPROCESS.Wow64Process 偏移 |
| `ProcessPeb` | `PsGetProcessPeb` | `*(ULONG*)(func + 3)` | EPROCESS.Peb 偏移 |
| `ExitStatus` | `PsGetProcessExitStatus` | `*(ULONG*)(func + 2)` | EPROCESS.ExitStatus 偏移 |
| `PteOffsets` | `MmGetVirtualForPhysical` | `*(ULONG64*)(func + 0x22)` | PTE 自引用基址 |
| `PfnOffsets` | `MmGetVirtualForPhysical` | `*(ULONG64*)(func + 0x10) & ~0xF` | PFN 数据库基址 |
| `MapAddress` | 运行时分配 | `MmAllocateIndependentPages(PAGE_SIZE)` | PTE 自映射窗口页 |

**原理**：这些 `PsGetProcessXxx` 函数的前几条指令通常是 `mov rax, [rcx + offset]`，其中 `offset` 就是 EPROCESS 中对应字段的偏移。直接读取指令中的立即数即可获得偏移值。

#### 特征码扫描定位的未导出函数

| 函数 | 特征码 (Pattern) | 扫描目标 | 解析方式 |
|------|------------------|----------|----------|
| `MmAllocateIndependentPages` | `\xE8...\x48\x8B\xF0\x48\x85\xC0\x0F\x84...` | ntoskrnl.exe .text | call 指令相对偏移 |
| `MmFreeIndependentPages` | `\xBA\x00\x60\x00\x00\x48\x8B\xCB\xE8...` | ntoskrnl.exe .text | call 指令相对偏移 |
| `MmSetPageProtection` | `\x41\xB8...\x48...\x8B\x00\xE8...` | ntoskrnl.exe .text | call 指令相对偏移 |

三个函数都是通过找到调用点（`call` 指令），然后计算 `call目标 = 指令地址 + *(INT*)(指令地址+1) + 5`。

#### 进程查询

| 函数 | 算法 | 关键点 |
|------|------|--------|
| `GetProcessIdByName` | 从 `PsInitialSystemProcess` 走 `ActiveProcessLinks` 链表 | 额外检查 `ExitStatus == 0x103`(STATUS_PENDING)，过滤已退出进程 |
| `GetProcessByProcessId` | 同上 | 仅匹配 PID，不检查退出状态 |

#### 文件强制删除 (ForceDeleteFile, Export.cpp:289-328)

1. `IoCreateFileEx` — 带 `IO_NO_PARAMETER_CHECKING` 打开文件
2. `ObReferenceObjectByHandleWithTag` — 获取 FILE_OBJECT
3. 清零 `SectionObjectPointer->ImageSectionObject` — **绕过文件正在使用的保护**
4. `MmFlushImageSection(MmFlushForDelete)` — 刷新镜像节
5. `ZwDeleteFile` — 删除文件

#### 键盘状态读取 (GetKeyStateFromKernel, Export.cpp:331-396)

1. 通过 `GetKernelModuleBase` 获取 `win32kbase.sys` 基址（回退 `win32k.sys`）
2. 特征码 `48 8D 0D ?? ?? ?? ?? E8` 扫描 `gafAsyncKeyState` 地址
3. RIP-relative 解码：`found + 7 + *(INT*)(found + 3)`
4. 直接读取 `keyStateArray[VirtualKey]`，检查 bit 7

### Inferences

- 偏移解析依赖内核函数的**机器码布局**——如果编译器优化改变了函数前几条指令的格式（比如 Windows 大版本更新），这些硬编码偏移（+3, +2, +0x22, +0x10）就会失效。
- `ActiveProcessLinks = UniqueProcessId + 8` 这个推导依赖 EPROCESS 结构中这两个字段**始终相邻**——在所有已知 Windows 10/11 版本中确实如此，但不是正式保证。

---

## 数据结构总览 (NativeStruct.h)

### Facts

```
namespace Offsets {                       ← 运行时偏移 (全部动态解析)
    UniqueProcessId    : ULONG            ← EPROCESS 偏移
    ActiveProcessLinks : ULONG            ← EPROCESS 偏移 (= UniqueProcessId + 8)
    ImageFileName      : ULONG            ← EPROCESS 偏移
    Wow64Process       : ULONG            ← EPROCESS 偏移
    ProcessPeb         : ULONG            ← EPROCESS 偏移
    ExitStatus         : ULONG            ← EPROCESS 偏移
    PteOffsets         : ULONG64          ← 页表自引用基址
    PfnOffsets         : ULONG64          ← PFN 数据库基址
    MapAddress         : PVOID            ← PTE 自映射窗口页
}

DataStruct {                              ← IOCTL 通信结构 (Driver.h)
    Name           : PCHAR
    ProcessPid     : ULONG
    ProcessCr3     : ULONG64
    TargetAddress  : ULONG64
    Length         : ULONG64
    Buffer         : PVOID
    VirtualKey     : SHORT
}

PEB32/64, PEB_LDR_DATA32/64,             ← 进程模块枚举所需
LDR_DATA_TABLE_ENTRY32/64                 ← 同时支持 WOW64 和原生 64 位
```

---

## 生命周期

```
时间线：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[加载]  sc create / 手动加载
    │
    ▼
[DriverEntry]  删除文件 → 初始化偏移 → 分配新内存 → 复制镜像 → 创建新驱动
    │
    ├── 新驱动 (MapEntry) 创建成功，设备对象就绪
    │
    └── return STATUS_UNSUCCESSFUL → 系统卸载原始镜像
                                      │
[运行]  用户态打开设备 → IOCTL → 读写内存          ← 持续服务
                                      │
[终止]  仅在系统重启时结束                         ← 无卸载路径
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Inferences

- **没有 DriverUnload 回调**——一旦加载无法正常卸载。`MmAllocateIndependentPages` 分配的内存和通过 `IoCreateDriver` 创建的设备对象将持续存在直到重启。
- `MapEntry` 失败时（IoCreateDevice 或 IoCreateSymbolicLink 失败）会调用 `MmFreeIndependentPages` 清理——这是唯一的清理路径。

---

## 模块依赖关系

```
Driver.cpp ──────┬──→ Export.cpp (InitSystemOffsets, ForceDeleteFile,
    │            │                MmAllocateIndependentPages, MmSetPageProtection,
    │            │                MmFreeIndependentPages)
    │            │
    │            └──→ Memory.cpp (ReadPhysicalMemory, WritePhysicalMemory,
    │                             GetProcessDirectoryBase, GetProcessModuleByPhyical)
    │
    ▼
Export.cpp ──────────→ Memory.cpp: 无直接依赖
                       (但 Memory.cpp 使用 Offsets namespace，由 Export.cpp 初始化)

Memory.cpp ──────────→ Export.cpp: GetProcessByProcessId (仅 GetProcessModuleByPhyical 使用)
```

**隐式依赖**：Memory.cpp 的所有功能都依赖 `Offsets` namespace 已被 `InitSystemOffsets()` 正确初始化。

---

## 技术要点与设计判断

### 1. 自重映射 vs 常规驱动注册

| 方面 | 自重映射 (NinDriver) | 常规方式 |
|------|---------------------|----------|
| 驱动列表可见性 | 不在 PsLoadedModuleList | 可见 |
| 磁盘痕迹 | 文件被删除 | 文件存在 |
| PE 头 | 被清零 | 存在 |
| 内存分配跟踪 | IndependentPages 不在常规 VAD | 在常规跟踪中 |
| 代价 | 无法卸载，无崩溃转储支持 | 正常生命周期管理 |

### 2. PTE 自映射 vs MmMapIoSpace

| 方面 | PTE 自映射 (NinDriver) | MmMapIoSpace |
|------|----------------------|--------------|
| 可监控性 | 低——直接修改 PTE | 高——API 调用可被 hook |
| 开销 | 极低——改 8 字节 + invlpg | 较高——涉及 MDL/映射 |
| 约束 | 单页操作，需自旋锁 | 可映射任意大小 |
| 风险 | TLB 不一致、竞态条件 | 相对安全 |

### 3. PFN 扫描 vs EPROCESS 偏移表

| 方面 | PFN 扫描 (NinDriver) | 偏移表 |
|------|---------------------|--------|
| 版本兼容性 | 较好——不依赖硬编码偏移 | 每个版本需维护 |
| 性能 | 差——扫描全部物理内存 | 好——直接读取 |
| 可靠性 | 依赖 PFN 结构布局 (0x30 大小) | 依赖 EPROCESS 偏移 |
| 适用场景 | CR3 获取（无导出 API） | 其他 EPROCESS 字段 |

---

## 遗留问题 (Stage 2 输入)

### 需深入分析

1. **PTE 自映射的并发安全**：Read 和 Write 各有独立 spin lock，是否存在竞态？
2. **EPROCESS 解密公式**的数学推导和版本适用范围
3. **PFN 0x30 字节大小**在不同 Windows 版本中是否一致？
4. **GetKeyStateFromKernel** 的 session 空间限制——win32kbase.sys 在 session space，驱动在 system context 下访问是否可靠？
5. 特征码扫描 pattern 在不同 ntoskrnl 版本中的稳定性

### 已自答的问题

- ~~全局变量在重映射后的值~~：memcpy 在赋值之后，全局变量被正确复制。
- ~~DriverEntry 返回后 MapEntry 的栈安全~~：IoCreateDriver 同步调用，MapEntry 在 DriverEntry 返回前已完成。
