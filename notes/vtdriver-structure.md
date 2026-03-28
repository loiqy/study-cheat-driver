# VT_Driver 结构分析

> 生成时间：2026-03-28 | Stage 1 产出

## 一句话总结

VT_Driver 是一个**基于 Intel VT-x 的 Type-2 Hypervisor**：通过在每个 CPU 上执行 VMXON → VMLAUNCH 将操作系统"降级"为 Guest，利用 EPT 分页实现透明内核钩取（代码页/数据页分离），上层驱动框架通过钩取 NtDeviceIoControlFile 接收 IOCTL 命令，提供进程内存读写、SSDT/SSSDT hook、PatchGuard 绕过、文件隐藏、窗口隐藏、网络包拦截和反调试等完整功能。

---

## 1. 总体架构

### Facts

```
┌──────────────────────────────────────────────┐
│           用户态应用                            │
│  NtDeviceIoControlFile(被钩取) → 11 个 IOCTL  │
└─────────────────┬────────────────────────────┘
                  │ IOCTL (0x9800 ~ 0x9828)
┌─────────────────▼────────────────────────────┐
│  VT_Driver/ — 驱动框架层 (~4,000+ 行)         │
│  ├─ Main.cpp: DriverEntry (LeiLei 加载)       │
│  │   └─ DriverInit(): LoadHV + CreateCallback │
│  ├─ Memory: CR3 切换 / APC Attach 内存读写    │
│  ├─ Process: PEB 遍历模块枚举                 │
│  ├─ Utils: SSDT 定位 + Inline Hook + LDE      │
│  ├─ Bypass: ObCallback/NtRead/NtWrite 反调试  │
│  ├─ PatchGuard: Win10/WinX KPP 绕过 PoC      │
│  ├─ Filter: MiniFilter 文件隐藏              │
│  ├─ Intercept: AFD 协议网络包拦截            │
│  ├─ ProtectWindow: SSSDT 窗口 API Hook       │
│  └─ Exception: 动态 SEH 注册 + 偏移解析      │
└─────────────────┬────────────────────────────┘
                  │ VMCALL / EPT Hook
┌─────────────────▼────────────────────────────┐
│  Intel-vt/ — VT-x 虚拟化核心层 (~3,600 行)    │
│  ├─ vt.cpp: LoadHV() 全局初始化               │
│  ├─ vmx.cpp: VMCS 配置 + VMX 生命周期         │
│  ├─ VmxExitHandlers.cpp: 65 个 VM-exit 分发   │
│  ├─ ept.cpp: EPT 恒等映射 + 钩取分页          │
│  ├─ public.cpp: EPT Hook 管理 + 物理内存查询  │
│  ├─ hvm.cpp: DPC 多核初始化                   │
│  ├─ LDasm.cpp: 线性反汇编器                   │
│  └─ kernel_stl.cpp: 内核 C++ 适配             │
└─────────────────┬────────────────────────────┘
                  │
           ┌──────▼──────┐
           │  Intel CPU   │
           │  VT-x 硬件   │
           └─────────────┘
```

- **Intel-vt/ 层**：~3,600 行，纯 Hypervisor 实现，管理 VMX 生命周期和 EPT 分页
- **VT_Driver/ 层**：~4,000+ 行，驱动业务逻辑，通过 VMCALL 与 VT 层交互

### Inferences

- 两层分离使 VT-x 代码可独立复用；上层功能模块不需要了解 VMCS 细节
- EPT 层是整个架构的核心优势——传统 Inline Hook 可被检测，EPT 代码页/数据页分离可实现"读取看到原始代码、执行走修改后的代码"
- 与 GsDriver/NinDriver 不同，VT_Driver 建立了一个额外的硬件抽象层（Hypervisor），这是复杂度量级跳跃的根源

---

## 2. 初始化流程

### 2.1 DriverEntry 与 LeiLei 加载链

**位置**: `VT_Driver/Main.cpp`

```
DriverEntry()                              ← 系统加载入口
  ├── 解析注册表获取驱动路径/名称
  ├── 创建 MiniFilter 注册表项 (Instances/Altitude)
  ├── LeiLeiMMapDriver()                   ← 多阶段驱动加载器
  │     ├── 分配内存，映射新镜像
  │     ├── 修复重定位和导入
  │     └── 调用 DriverInit() 作为真正入口
  └── 自身卸载 (stub 角色结束)
```

**Facts:**
- DriverEntry 是一个**加载桩**，真正的初始化在 `DriverInit()` 中完成
- LeiLei 加载器隐藏了原始驱动对象——加载完成后驱动对象的字段被清空
- 加载器在 `LoadDriver/` 子目录实现，支持驱动加载/卸载和驱动对象获取

### 2.2 DriverInit() — 真正的初始化

**位置**: `VT_Driver/Main.cpp`

```
DriverInit()
  ├── LoadHV()                             ← 启动 Hypervisor
  ├── Utils::LDE_init()                    ← 初始化反汇编引擎 (加载 ShellCode)
  ├── MiniFilter 初始化                     ← 文件系统保护
  ├── Intercept 初始化                      ← 网络包拦截
  ├── ProtectWindow 初始化                  ← 窗口保护
  ├── CreateCallback()                     ← 钩取 NtDeviceIoControlFile
  │     ├── MainVtMode=TRUE → PHHook()     (EPT 页级钩取)
  │     └── MainVtMode=FALSE → HookKernelApi() (Inline Hook)
  └── 清空 DriverObject 字段               ← 隐藏驱动痕迹
```

**Facts:**
- `CreateCallback()` 根据 `MainVtMode` 全局变量选择钩取方式
- 如果 LoadHV 成功，使用 EPT 透明钩取（更隐蔽）
- 如果 LoadHV 失败，退回到传统 Inline Hook

### Inferences

- 双路径设计提供了降级能力——在不支持 VT-x 的 CPU 上仍可工作
- EPT 钩取优先于 Inline Hook，说明隐蔽性是主要设计目标

---

## 3. VT-x Hypervisor 核心

### 3.1 LoadHV() 完整流程

**位置**: `Intel-vt/vt.cpp:67`

```
LoadHV()
  ├── 初始化自旋锁 (R3pageList / R3PageLock)
  ├── IsHVSupported()
  │     └── VmxHardSupported()              ← CPUID + MSR_IA32_FEATURE_CONTROL 检查
  ├── GetSSDTEntry()                        ← 获取 SSDT 基址 (后续 hook 需要)
  ├── AllocGlobalData()
  │     ├── 分配 GLOBAL_DATA + cpu_count × sizeof(VCPU)
  │     ├── 每个 VCPU 预分配 EPT_PREALLOC_PAGES (512页)
  │     └── 分配 MSR Bitmap (PAGE_SIZE)
  ├── UtilQueryPhysicalMemory()             ← 获取物理内存区间表
  ├── HvmCheckFeatures()                    ← 检测 EPT/VPID/Execute-Only 支持
  ├── StartHV()
  │     └── KeGenericCallDpc → HvmpHVCallbackDPC → IntelSubvertCPU
  │           └── 每个 CPU 上执行 VmxInitializeCPU()
  └── MainVtMode = TRUE
```

**Facts:**
- `AllocGlobalData()` 动态分配：`sizeof(GLOBAL_DATA) + cpu_count * sizeof(VCPU)`
- 每个 VCPU 预分配 512 页用于 EPT，避免高 IRQL 下的内存分配困难
- 使用 `KeGenericCallDpc` 在所有 CPU 上并行执行虚拟化初始化
- 物理内存查询结果包括 MMIO 区间 (0xF0000000 - 0xFFFFFFF) 和 APIC 页

### 3.2 Per-CPU VMX 初始化

**位置**: `Intel-vt/vmx.cpp` (VmxSubvertCPU) + `Intel-vt/hvm.cpp` (VmxInitializeCPU)

```
VmxInitializeCPU()
  ├── KeSaveStateForHibernate()
  ├── RtlCaptureContext()                  ← 保存当前 RIP/RSP
  ├── if VmxState == TRANSITION:           ← VMLAUNCH 成功后回到此处
  │     ├── VmxState = ON
  │     └── RtlRestoreContext() → 返回调用者 (DPC callback)
  └── if VmxState == OFF:                  ← 首次进入
        ├── 保存 SystemDirectoryTableBase (Host CR3)
        └── VmxSubvertCPU()

VmxSubvertCPU()
  ├── 读取全部 VMX 相关 MSR
  ├── 分配 VMXON Region (物理连续)
  ├── 分配 VMCS Region (物理连续)
  ├── 分配 VMMStack (KERNEL_STACK_SIZE, 物理连续)
  ├── VmxEnterRoot()
  │     ├── 检查: VMCS ≤ 1 PAGE, MemoryType = WB, TrueMSR 支持
  │     ├── 修正 CR0/CR4 的 must-be-zero/must-be-one 位
  │     ├── __vmx_on()                     ← 进入 VMX Root 模式
  │     ├── __vmx_vmclear()                ← 初始化 VMCS
  │     └── __vmx_vmptrld()                ← 加载活动 VMCS
  ├── VmxSetupVMCS()                       ← 配置全部 VMCS 字段
  ├── EptBuildIdentityMap()                ← 构建 EPT 恒等映射
  ├── EptEnable()                          ← 在 VMCS 中启用 EPT + VPID
  └── __vmx_vmlaunch()                     ← 启动 Guest
        ├── 成功: RIP 回到 RtlCaptureContext 下一条指令
        └── 失败: 清理所有分配，__vmx_off()
```

**Facts:**
- `RtlCaptureContext` 是 Guest 恢复点——VMLAUNCH 成功后 Guest 继续执行 `RtlCaptureContext` 之后的代码
- Host RIP 指向汇编入口 `VmxVMEntry`，Host RSP 指向 VMMStack + 24KB - sizeof(CONTEXT)
- Guest 和 Host 共享相同的 CR3/GDT/IDT/段寄存器值（subvert 模型）

### 3.3 VMCS 配置要点

**位置**: `Intel-vt/vmx.cpp` (VmxSetupVMCS)

**控制字段配置:**
| 类别 | 关键位 | 用途 |
|------|--------|------|
| Pin-Based | External Interrupt Exiting, NMI Exiting | 拦截中断和 NMI |
| CPU-Based | Secondary Controls, TSC Offsetting, RDTSC Exiting | 支持二级控制和时间戳拦截 |
| Secondary | RDTSCP, XSAVES/XRSTORS, EPT, VPID | 核心功能 |
| Entry | IA32e Mode Guest | 64 位 Guest |
| Exit | Acknowledge Interrupt, Host Address Space Size | 中断确认和 64 位 Host |

**MSR Bitmap 拦截:**
- VMX 相关 MSR (VMX_BASIC ~ VMX_TRUE_ENTRY_CTLS): 读拦截
- 这防止 Guest 发现 VMX 已被使用

**异常位图:** 全部清零（不拦截任何异常，全部直通 Guest）

### Inferences

- 不拦截异常意味着 Guest 的异常处理完全透明——Hypervisor 不干预正常的异常/中断流程
- MSR 拦截 VMX 相关 MSR 是反检测关键——Guest 中的代码无法通过读取 MSR 发现 VMX 处于活动状态
- RDTSC 拦截用于时间戳防检测——可以修改返回的 TSC 值隐藏 VM-exit 造成的时间差

---

## 4. EPT (扩展页表) 实现

### 4.1 EPT 结构

**位置**: `Intel-vt/ept.h` + `Intel-vt/ept.cpp`

```
4 级分页: PML4 → PDPTE → PDE → PTE (每级 512 项，9 bit 索引)

EPT_TABLE_POINTER (EPTP):
  ├── MemoryType (3 bit): 通常 = 6 (WB)
  ├── PageWalkLength (3 bit): 通常 = 3 (4 级)
  └── PhysAddr (40 bit): PML4 物理地址

EPT_PTE_ENTRY (最终页表项):
  ├── Read / Write / Execute (各 1 bit): 独立权限控制
  ├── MemoryType (3 bit): UC(0) / WB(6)
  └── PhysAddr (40 bit): 物理页地址
```

### 4.2 恒等映射构建

**Facts:**
- `EptBuildIdentityMap()` 为所有物理内存区间建立 Guest 物理地址 = Host 物理地址 的 1:1 映射
- 物理内存范围来自 `MmGetPhysicalMemoryRanges()` + 手动添加的 MMIO 区间
- 缺失的页在 EPT_VIOLATION 时按需构建

### 4.3 EPT Hook 机制 — 代码页/数据页分离

**这是 VT_Driver 最核心的技术特征。**

```
R3EPT_HOOK 结构:
  ├── CodePage: 包含修改后代码的物理页
  ├── DataPage: 包含原始代码的物理页
  ├── TargetCR3: 目标进程的页表基址
  └── 链表节点: 挂在全局 R3pageList 上

EPT Hook 原理:
  ┌───── 读取/数据访问 ─────┐     ┌───── 执行访问 ─────┐
  │  EPT PTE → DataPage     │     │  EPT PTE → CodePage │
  │  (原始代码，校验通过)     │     │  (修改后代码，实际执行) │
  └─────────────────────────┘     └─────────────────────┘
                    ↑ EPT_VIOLATION 触发切换 ↑
```

**Facts:**
- VMCALL `HYPERCALL_HOOK_PAGE` (0x0003) 更新 EPT：将目标页标记为 Execute-Only（指向 CodePage）
- 当 Guest 尝试读取该页时，触发 EPT_VIOLATION → 切换到 DataPage (Read/Write) + 启用 MTF
- 下一条指令执行后，MTF 触发 VM-exit → 切换回 CodePage (Execute-Only)
- 这种乒乓切换实现了"读到原始代码、执行修改代码"的透明钩取

**Hook 分为两类:**
- **R0 Hook**: 内核态页（如 NtDeviceIoControlFile）
- **R3 Hook**: 用户态页（通过 R3pageList 管理，带进程 CR3 匹配）

### Inferences

- EPT Hook 是目前最隐蔽的内核钩取方式——内存扫描检测看到的是原始代码
- MTF (Monitor Trap Flag) 单步执行是避免长期处于"读取模式"的关键——只在一条指令窗口内切换
- R3 Hook 需要维护进程列表（PmainList），因为不同进程的 CR3 不同

---

## 5. VM-exit 处理

### 5.1 分发机制

**位置**: `Intel-vt/VmxExitHandlers.cpp`

```
VmxpExitHandler() [DECLSPEC_NORETURN, 汇编入口 VmxVMEntry 跳入]
  ├── 提升 IRQL 到 HIGH_LEVEL
  ├── 从 VMCS 读取 GUEST_STATE: RIP, RSP, RFLAGS, 退出原因, 资格信息...
  ├── 注入 #DB (如果 TF 标志置位)
  ├── 清除 RF 标志
  ├── g_ExitHandler[exitReason]()           ← 65 个函数指针表分发
  ├── if ExitPending:                       ← VMCALL UNLOAD 触发
  │     ├── 重载 GDT/IDT/CR3
  │     ├── __vmx_off()                     ← 退出 VMX Root
  │     └── 恢复上下文 → 返回正常执行
  └── else:
        └── VmxpResume()                    ← VMRESUME 回到 Guest
```

### 5.2 有实际逻辑的 VM-exit Handler

| 退出原因 | 函数 | 逻辑说明 |
|----------|------|----------|
| 0 EXCEPTION_NMI | VmExitEvent | 根据类型重注入异常/中断（NMI/硬件/软件） |
| 2 TRIPLE_FAULT | VmExitTripleFault | BugCheck（不可恢复） |
| 10 CPUID | VmExitCPUID | 执行 CPUID，**屏蔽 EAX=1 时 ECX 的 VMX 位** |
| 13 INVD | VmExitINVD | 执行 __wbinvd() |
| 16 RDTSC | VmExitRdtsc | 执行 __rdtsc()，返回 EDX:EAX |
| **18 VMCALL** | VmExitVmCall | **6 个超调用接口** (见下) |
| 19-27 VM指令 | VmExitVMOP | 注入 #UD（禁止 Guest 执行 VMX 指令） |
| 28 CR_ACCESS | VmExitCR | 处理 MOV to/from CR0/3/4，CR3 写入时 INVVPID |
| 31 MSR_READ | VmExitMSRRead | 虚拟化 LSTAR/GS_BASE/FS_BASE/VMX MSR |
| 32 MSR_WRITE | VmExitMSRWrite | 允许写 GS_BASE/FS_BASE/LSTAR (未 hook 时) |
| 37 MTF | VmExitMTF | EPT Hook 乒乓切换的核心——单步后恢复 Execute-Only |
| **48 EPT_VIOLATION** | VmExitEptViolation | **EPT Hook 触发点**: 按需映射 + 钩取分发 |
| 49 EPT_MISCONFIG | VmExitEptMisconfig | BugCheck（配置错误） |
| 51 RDTSCP | VmExitRdtscp | 执行 __rdtscp()，返回 EDX:EAX + ECX |
| 55 XSETBV | VmExitXSETBV | 执行 _xsetbv() 设置 XCR0 |

**其余约 40 个 Handler 为空桩 (`VmExitUnknown`)，直接返回。**

### 5.3 VMCALL 超调用接口

| 调用码 | 名称 | 用途 |
|--------|------|------|
| 0x0000 | HYPERCALL_UNLOAD | 关闭 Hypervisor，设置 ExitPending |
| 0x0001 | HYPERCALL_HOOK_LSTAR | Hook LSTAR MSR (syscall 入口) |
| 0x0002 | HYPERCALL_UNHOOK_LSTAR | 恢复 LSTAR MSR |
| 0x0003 | HYPERCALL_HOOK_PAGE | EPT Hook：将页设为 Execute-Only → CodePage |
| 0x0004 | HYPERCALL_UNHOOK_PAGE | 恢复 EPT 页为全权限 |

### Inferences

- CPUID 屏蔽 VMX 位 + MSR 虚拟化 VMX MSR = 双重反检测，Guest 无法通过标准方法发现 Hypervisor
- LSTAR Hook 意味着可以拦截所有 syscall 入口——这是最底层的系统调用拦截点
- 禁止 Guest 执行 VMX 指令 (#UD) 防止嵌套虚拟化冲突

---

## 6. 通信机制：NtDeviceIoControlFile 钩取

### Facts

**位置**: `VT_Driver/Main.cpp`

```
Fake_NtDeviceIoControlFile()
  ├── 检查 IoControlCode 范围: 0x9800 ~ 0x9828
  ├── 仅处理 OwnPid 的请求 (PROTO_TEST 除外)
  ├── 同时拦截 AFD_SEND/SENDTO/BIND/CONNECT (网络包拦截)
  └── 不匹配 → 调用原始函数
```

**IOCTL 命令表:**

| 码 | 宏 | 功能 |
|----|-----|------|
| 0x9800 | PROTO_TEST | 回显测试 |
| 0x9804 | PROTO_READWRITE | 内存读写 |
| 0x9808 | PROTO_READWRITE_TX | 系统线程内存读写 |
| 0x980C | PROTO_SET_PROCESS | 设置目标进程 |
| 0x9810 | PROTO_GET_MODULE | 获取模块基址 |
| 0x9814 | PROTO_FILE_PROTECTION | 添加/清除文件保护规则 |
| 0x9818 | PROTO_DISABLE_PG | 禁用 PatchGuard |
| 0x981C | PROTO_DELETE_FILES | 删除文件 |
| 0x9820 | PROTO_PACKET_FEATURE | 网络包特征替换 |
| 0x9824 | PROTO_PROTECT_WINDOW | 窗口保护 |
| 0x9828 | PROTO_DEBUG_GINGTOOL | 调试工具绕过 |

### Inferences

- 钩取 `NtDeviceIoControlFile` 而非使用标准 IRP 分发，使得即使设备对象被发现，通信路径也不走常规 IOCTL 栈
- 仅响应 `OwnPid` 的请求是安全措施——防止其他进程触发驱动功能
- AFD 拦截复用同一个 hook 点，减少了额外的 hook 数量

---

## 7. 功能模块分析

### 7.1 内存读写 (Memory.cpp)

**两种模式:**

| 模式 | 函数 | 机制 |
|------|------|------|
| 直接 | ReadMemory / WriteMemory | KeStackAttachProcess → 直接读写 → KeUnstackDetachProcess |
| 系统线程 | SystemReadMemory / SystemWriteMemory | 创建系统线程 → 线程内 CR3 切换到目标进程 → 读写 → 事件信号通知 |

**Facts:**
- `COPY_MEMORY` 结构: `{address, value, size, type(1=读/2=写)}`
- 系统线程模式更隐蔽——不使用 KeStackAttachProcess，直接修改 CR3
- `MmIsAddressValid()` 作为安全检查

### 7.2 进程管理 (Process.cpp)

- `GetProcessModules()`: PEB → LDR_DATA → 遍历 InLoadOrderModuleList
- `ApplyMemory()`: 通过 ZwAllocateVirtualMemory 在目标进程分配/释放内存
- 直接通过 PsGetProcessPeb() 获取 PEB，无需 EPROCESS 偏移表

### 7.3 Hook 体系 (Utils.cpp)

**四层 Hook 机制:**

| 层级 | 类型 | 实现 |
|------|------|------|
| L0 | EPT Hook | VT-x 层代码页/数据页分离，最隐蔽 |
| L1 | Inline Hook | 14 字节 JMP trampoline (`FF 25 00 00 00 00 [addr]`)，LDE 计算覆盖长度 |
| L2 | SSDT Hook | 修改系统服务描述符表条目 |
| L3 | SSSDT Hook | 修改 Win32k 影子服务表 (窗口相关 API) |

**SSDT 定位:**
- 通过模式扫描 `KiSystemServiceRepeat` 特征码: `\x4c\x8d\x15\xcc\xcc\xcc\xcc\x4c\x8d\x1d\xcc\xcc\xcc\xcc\xf7`
- `GetSSDTEntry()` 从 SSDT 索引计算实际函数地址

**LDE (Length Disassembly Engine):**
- 嵌入在 ShellCode.h 中的 ~12,800 字节编译好的引擎
- 用于 `GetPatchSize()`: 计算 hook 覆盖的最小完整指令边界

### 7.4 反调试 (Bypass/Bypass.cpp)

**钩取的函数:**
| 目标 | 效果 |
|------|------|
| ObpCallPreOperationCallbacks | 绕过对象回调通知（隐藏进程句柄操作） |
| RtlpCopyLegacyContextX86 | 绕过 WOW64 上下文检测 |
| NtReadVirtualMemory | 隐藏对受保护进程的内存读取 |
| NtWriteVirtualMemory | 隐藏对受保护进程的内存写入 |
| NtQueryInformationThread | 隐藏调试线程信息 |

**Facts:**
- 使用硬编码偏移（Win7），版本兼容性有限
- 维护调试进程列表 (`RULE_BYPASS_OWN`) 和线程 DR 状态表 (`THREAD_dr_List`)

### 7.5 PatchGuard 绕过 (PatchGuard/)

- 两个 PoC: `Win10PatchguardPoc` 和 `WinXPatchguardPoc`
- 必须先绕过 PatchGuard 才能安全执行 SSDT Hook 等内核修改
- 通过 IOCTL `PROTO_DISABLE_PG` (0x9818) 触发

### 7.6 文件隐藏 (Filter/MiniFilter.cpp)

- 使用 FLT 框架注册文件系统 MiniFilter
- 拦截 `IRP_MJ_DIRECTORY_CONTROL` 的 Post 操作
- 从目录枚举结果中移除受保护文件名 (`CleanFileXxxDirectoryInformation`)
- 规则通过 `Add_Rule()` / `Clear_Rule()` 动态管理

### 7.7 窗口保护 (ProtectWindow.cpp)

**钩取的 Win32k API (SSSDT):**
| 函数 | 索引 (Win7/Win8+) | 效果 |
|------|-------------------|------|
| NtUserFindWindowEx | 0x106E / 0x106F | 阻止窗口枚举 |
| NtUserQueryWindow | 0x1010 / 0x1013 | 阻止窗口查询 |
| NtUserGetForegroundWindow | 0x103C / 0x103F | 阻止前台窗口获取 |
| NtUserBuildHwndList | — | 阻止窗口列表构建 |
| NtUserWindowFromPoint | — | 阻止坐标→窗口转换 |
| NtUserGetClassName | — | 阻止类名查询 |
| NtGdiGetPixel | — | 阻止像素读取 |
| NtUserChildWindowFromPointEx | — | 阻止子窗口查找 |
| NtUserWindowFromPhysicalPoint | — | 阻止物理坐标→窗口转换 |

### 7.8 网络包拦截 (Intercept.cpp)

- 钩取 NtDeviceIoControlFile 的 AFD 操作 (AFD_SEND/SENDTO/BIND/CONNECT)
- `LookupSendPacket()` 检测 HTTP 流量 (GET/POST)
- 支持特征匹配和内容替换 (`RULE_PACKET_FEATURE` 规则)

### 7.9 异常处理 (Exception.cpp)

- `HandlingExceptions()`: 根据 NtBuildNumber 版本选择不同的 SEH 注册路径
- 核心: 调用 `RtlInsertInvertedFunctionTable` 将驱动注册到异常处理框架
- `MiProcessLoaderEntry` 通过模式扫描定位——用于让手动映射的驱动也能使用 SEH
- 支持 Win7 和 Win10+ 两个代码路径

---

## 8. 关键数据结构

### GLOBAL_DATA

```
GLOBAL_DATA
  ├── vcpus: ULONG                         — 已启动的 VCPU 数量
  ├── SystemDirectoryTableBase: ULONG_PTR  — Host CR3
  ├── Memory: PHYSICAL_MEMORY_DESCRIPTOR*  — 物理内存区间表
  ├── MsrBitmap: PUCHAR                    — MSR 拦截位图
  ├── Vcpus[cpu_count]: VCPU               — Per-CPU 虚拟化状态
  └── ...
```

### VCPU (Per-CPU)

```
VCPU
  ├── VmxState: enum (OFF/TRANSITION/ON)
  ├── VmxonRegion: PHYSICAL_ADDRESS
  ├── VmcsRegion: PHYSICAL_ADDRESS
  ├── VMMStack: 虚拟地址
  ├── EPT: EPT 分页结构 (PML4 + 预分配页)
  ├── MSR 值缓存
  └── ...
```

### GUEST_STATE (VM-exit 时填充)

```
GUEST_STATE
  ├── GuestRIP, GuestRSP, GuestRFLAGS
  ├── ExitReason
  ├── ExitQualification
  ├── LinearAddress, PhysicalAddress
  ├── ExitPending: BOOLEAN                 — 设置后下次 exit 关闭 VMX
  └── ...
```

### R3EPT_HOOK

```
R3EPT_HOOK
  ├── CodePage: 物理页 (含修改后代码)
  ├── DataPage: 物理页 (含原始代码)
  ├── TargetCR3: 目标进程 CR3
  └── ListEntry: 全局链表节点
```

---

## 9. 与 GsDriver / NinDriver 的初步对比

| 维度 | NinDriver | GsDriver | VT_Driver |
|------|-----------|----------|-----------|
| **通信机制** | IOCTL (标准设备) | 注册表回调 | NtDeviceIoControlFile 钩取 |
| **内存访问** | PTE 自映射物理读写 | MmCopyVirtualMemory + MDL | KeStackAttach / CR3 切换 |
| **隐蔽层级** | 自重映射 + PE 头清零 | 手动映射 + 匿名线程 + 跳板 | VT-x Hypervisor + EPT 分页 |
| **Hook 能力** | 无 | 无 (纯功能驱动) | 四层: EPT/Inline/SSDT/SSSDT |
| **规模** | ~880 行 | ~7,800 行 | ~7,600 行 + capstone |
| **复杂度核心** | PTE 页表操作 | 注册表回调 + 注入 | VMX 生命周期 + EPT |

---

## 10. Open Questions

1. **EPT 预分配 512 页是否足够？** 高 IRQL 下如果预分配用尽，会触发 panic——实际场景下什么时候会触发？
2. **Win10/WinX PatchGuard PoC 的实际可靠性？** 代码中存在两个版本，是否都经过验证？
3. **Bypass 模块的硬编码偏移** (ObpCallPreOperationCallbacks 等) 仅适用于 Win7——是否有动态版本？
4. **LSTAR Hook 的实际使用场景？** 代码提供了 HOOK_LSTAR/UNHOOK_LSTAR VMCALL，但上层何时调用？
5. **capstone vs LDasm vs ShellCode LDE**——三个反汇编器各自的使用场景？capstone 是否只在初始化时使用？
6. **MMIO 硬编码区间 (0xF0000000 - 0xFFFFFFF)** 在不同硬件上是否可靠？
7. **DriverObject 字段清空后**，是否会影响系统稳定性（如 PnP 或电源管理）？
8. **EPT Hook MTF 乒乓窗口** 中，如果恰好发生中断，是否会产生竞态？

---

## 11. Recommendations

1. **Stage 2 核心路径分析建议优先选 VT_Driver 的 VMX 初始化路径**——这是三个项目中最独特且最复杂的路径
2. **跨项目比较重点**: 三种内存访问方式（PTE 自映射 vs MmCopyVirtualMemory vs CR3 切换）的安全性和检测面对比
3. **EPT Hook 机制值得单独出一份深度笔记**——这是区别于传统方案的核心技术
