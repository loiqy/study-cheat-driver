# GsDriver 结构分析

> 生成时间：2026-03-28 | Stage 1 产出

## 一句话总结

GsDriver 是一个**双层驱动框架**：驱动外壳负责环境初始化和手动映射核心驱动，驱动核心以系统线程运行，通过**注册表回调**实现用户态通信，提供完整的进程操作、DLL 注入、内存隐藏、硬件 ID 伪造、键鼠模拟和反反作弊功能。

---

## 1. 总体架构

### Facts

```
┌─────────────────────────────────────────────────────────────────┐
│                    驱动外壳 (Loader)                             │
│  DriverEntry → DriverStart → KernelStart → SystemStart          │
│            → GetPteTable → MmMapLoadDriver                      │
│  结果: 手动映射核心驱动到 NonPagedPool, 启动为系统线程            │
│  最后: 自删除 .sys 文件, 返回 0xE0000000                         │
└────────────────────────┬────────────────────────────────────────┘
                         │ 通过文件 GSDrv.bin 传递 DynamicData 指针
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    驱动核心 (Core)                               │
│  DriverEntry() → VariateInit → DriverStart → RegisterNotifyInit │
│  → PsTerminateSystemThread (线程退出，回调留存)                   │
│                                                                 │
│  核心运行机制: CmRegisterCallback 注册表回调                     │
│  用户态: 写注册表 → 触发 RegisterNotify → Type 字段分发命令       │
└─────────────────────────────────────────────────────────────────┘
```

- **驱动外壳**：~2,083 行，纯粹是加载器，运行完即被系统卸载
- **驱动核心**：~5,755 行，提供全部功能，以匿名内存块存在

### Inferences

- 双层分离使"外壳"可以合法签名加载，而"核心"作为匿名代码块避开模块列表检测
- 外壳返回 `0xE0000000` 而非 `STATUS_SUCCESS`，触发系统将外壳从模块列表移除，但核心已经在独立内存中运行
- 文件自删除 + 非标准返回码 = 加载完成后外壳无痕消失

---

## 2. 驱动外壳详细分析

### 2.1 DriverEntry 初始化链

**位置**: `驱动外壳/驱动外壳.cpp:192`

```
DriverEntry
  ├── DynamicData = RtlAllocateMemory(sizeof(DYNDATA))
  ├── DriverStart()           → 版本检测 + EPROCESS 偏移表计算
  ├── KernelStart()           → 定位 ntoskrnl 基址和模块列表
  ├── SystemStart()           → 读 ntdll.dll 获取 SSDT 系统调用地址
  │     └── NtCreateThreadEx, NtProtectVirtualMemory
  ├── GetPteTable()           → 计算 PTE 四级页表基地址
  └── MmMapLoadDriver()       → 手动映射核心驱动
        ├── 分配 NonPagedPoolExecute 内存
        ├── 复制 PE 头 + 各段
        ├── LdrRelocateImageWithBias() — 重定位
        ├── ResolveImageRefs()         — IAT 修复 (用 hash 匹配)
        ├── 写 DynamicData 指针到 GSDrv.bin
        └── PsCreateSystemThread() 启动核心 DriverEntry
```

### 2.2 外壳→核心桥接机制

**Facts:**
- 外壳将 `DynamicData` 指针写入 `\SystemRoot\System32\GSDrv.bin`
- 核心的 `DriverStart()` 从该文件读回指针，然后删除文件
- DynamicData 包含：WinVersion、BuildNumber、KernelBase、ModuleList、PageTables[4]、所有 EPROCESS 偏移、SSDT 函数地址、DriverBase
- 核心读取后立即清零 `DriverBase` 的首页 (PE 头)

**Inferences:**
- 文件桥接而非参数传递，因为核心以系统线程启动，`PsCreateSystemThread` 的 StartContext 参数为 NULL
- GSDrv.bin 是临时传输通道，读完即删，不留痕迹

### 2.3 版本兼容性

**Facts:**
- `DriverStart()` 计算 7 个 EPROCESS 偏移：VadRoot、PrcessId、Protection、PspCidTable、ProcessLinks、PrcessIdOffset、ParentPrcessIdOffset
- 每个偏移覆盖 Win7 / Win8 / Win8.1 / Win10 (多个 BuildNumber 分支)
- PTE 页表基址：Win7/Win8 使用硬编码地址 (0xFFFFF68000000000)，Win8.1+ 通过 CR3 → PML4 自引用项动态发现

### 2.4 IAT 修复 — Hash 匹配

**Facts:**
- `ResolveImageRefs()` 遍历核心驱动的 IAT
- 对每个导入模块名和函数名计算 hash (`GetTextHashA`)
- 通过 `GetModuleBaseForHash` 查找模块基址（内核特判 hash `0xFF2A308D`，其他遍历模块列表）
- 通过 `GetRoutineAddressForHash` 在导出表中匹配 hash

**Inferences:**
- 全部使用 hash 而非明文字符串匹配，增加静态分析难度
- hash 算法：`65599 * (Hash + char + caseFold)`，一个简单的大小写不敏感乘法哈希

---

## 3. 驱动核心详细分析

### 3.1 初始化流程

**位置**: `驱动核心/驱动核心.cpp:58`

```
DriverEntry()        ← 无参数，以系统线程身份运行
  ├── VariateInit()  ← 清零 WSK 网络和键鼠模拟全局变量
  ├── DriverStart()
  │     ├── 从 GSDrv.bin 读回 DynamicData 指针
  │     ├── 清零核心驱动 PE 头 (DriverBase 首页)
  │     └── 分配 Hook 缓冲区 (3个 HOOK_NOTIFY_BUFFER + 1个 HIDE_MEMORY_BUFFER)
  ├── RegisterNotifyInit(TRUE) ← 注册注册表回调
  └── PsTerminateSystemThread() ← 线程退出，回调持续存在
```

### 3.2 通信机制：注册表跳板

**位置**: `驱动核心/通讯回调.cpp`

**Facts:**
- 使用 `CmRegisterCallback` 注册注册表回调 `RegisterNotify`
- 拦截 `RegNtPreSetValueKey` 事件
- 以 `PreSetValueInfo->Type` 字段作为命令码分发
- 以 `PreSetValueInfo->Data` 作为输入参数缓冲区
- 以 `PreSetValueInfo->DataSize` 验证参数大小
- 返回自定义 NTSTATUS (ERROR_成功 = 0xE0000000 等) 作为结果码

**回调注册反检测：**
- 在内核代码段搜索 `\xFF\xE1` (jmp rcx) 指令
- 将该地址作为回调入口注册给 `CmRegisterCallback`
- 运行时通过 `RtlSuperCopyMemory` (MDL 写入) 将 `jmp <实际回调>` 写入该位置

**Inferences:**
- 注册表通信避开了设备对象/IRP 拦截，反检测更隐蔽
- jmp rcx trampoline 意味着回调地址指向合法系统模块的代码段，反检测工具枚举回调时看到的是系统驱动地址

### 3.3 命令分发表

| Type 编码 | 功能 | 参数结构大小 |
|-----------|------|-------------|
| `'0000'` | 通讯测试 | Data == NULL |
| `'0001'` | 用户验证 | (触发 UserVerify 标志) |
| `'0002'` | 离线注入 | sizeof(INJECT_DATA) |
| `'0003'` | 句柄提权 | sizeof(HANDLE) |
| `'0004'` | 进程基址 | hProcessId + OutBuffer |
| `'0005'` | 进程模块 | hProcessId + ModuleName + OutBuffer |
| `'0006'` | 内存读写 | hProcessId + Target + Source + Size + Type(0/1/2) |
| `'0007'` | 强删文件 | FilePath |
| `'0008'` | 保护进程 | hProcessId + Enable |
| `'0009'` | 隐藏进程 | hProcessId (已注释，返回失败) |
| `'0010'` | 强杀进程 | ProcessName |
| `'0011'` | 申请内存 | hProcessId + MemSize + MemProtect + HighAddress + OutBuffer |
| `'0012'` | 释放内存 | hProcessId + MemoryAddress |
| `'0013'` | 内存属性 | hProcessId + MemAddress + RegionSize + NewProtect |
| `'0014'` | 隐藏内存 | hProcessId + MemAddress + NumberOfBytes |
| `'0015'` | 查询内存 | hProcessId + MemAddress + OutBuffer |
| `'0016'` | 创建线程 | hProcessId + Address |
| `'0017'` | 模拟鼠标 | sizeof(MOUSE_INPUT_DATA) |
| `'0018'` | 模拟键盘 | sizeof(KEYBOARD_INPUT_DATA) |
| `'0019'` | 改机器码 | Type (ULONG32) |
| `'0020'` | 搜特征码 | hProcessId + SiginCode + Size + Protect + Address + OutBuffer |
| `'0021'` | 窗口反截 | hWnd + Flags |
| `'1000'`–`'1006'` | 注入内部接口 | (被注入 DLL 使用的内部命令) |

**Facts:**
- `'0001'` 以上命令需要 UserVerify == TRUE 才可执行（门禁）
- `'0006'` 内存读写有三种模式：Type 0 = 读(MmCopyVirtualMemory)，Type 1 = 普通写，Type 2 = MDL 强写（绕只读保护）
- `'1000'`–`'1006'` 是注入后 DLL 内部使用的辅助命令（内存分配、强写、键鼠模拟等）

---

## 4. 功能模块分析

### 4.1 DLL 注入引擎 (`注入回调.cpp` ~970行)

**Facts:**
- 通过 `PsSetLoadImageNotifyRoutine` 监听模块加载事件
- 支持 x86 和 x64 两种目标
- 三种注入模式 (InjectMode 0/1/2)：
  - **Mode 0**: 直接 `ZwCreateThreadEx` 创建远程线程执行 ShellCode
  - **Mode 1**: Hook `ZwContinue` 入口（jmp 劫持），进程恢复时触发注入
  - **Mode 2**: Steam 专用 — 监听 `GameOverlayRenderer.dll` / `GameOverlayRenderer64.dll` 加载，劫持其函数指针

**注入内存分配：**
- `AllocMemory_x86/x64`：根据 InjectHide 级别选择分配方式
  - 普通 (`PAGE_EXECUTE_READWRITE`)：`ZwAllocateVirtualMemory`
  - 隐蔽 (`PAGE_NOACCESS`)：`MmAllocatePagesForMdlEx` + MDL 映射 + PFN 清零 + PTE 用户态可见
- 分配后布局：`[映射空间] [ShellCode + DLL 数据] [PE 加载器 ShellCode]`

**隐藏策略 (InjectHide 0/1/2/3):**
- 0: 无隐藏
- 1: VAD 树摘除 (`AddMemoryItem`)
- 2: PTE 修改 (`SetPhysicalPage`，设置 NoExecute=0, Write=1)
- 3: MDL 物理页分配 + PFN 清零 + PTE Owner=1 (用户态可见)

**PTE 操作细节 (`SetPhysicalPage`, `ShareMemoryEx`):**
- 直接修改四级页表项 (PXE → PPE → PDE → PTE)
- 每页修改后调用 `__invlpg` 刷新 TLB
- `ShareMemoryEx` 额外设置 Owner=1 使内核页对用户态可见

### 4.2 进程回调 + VAD 操作 (`进程回调.cpp` ~558行)

**Facts:**
- `ProcessNotify` 通过 `PsSetCreateProcessNotifyRoutine` 注册
- 进程退出时调用 `DelMemoryItem` 恢复被隐藏的 VAD 节点
- VAD 隐藏：遍历 AVL 树找到目标节点，调用 `MiRemoveNode`/`RtlAvlRemoveNode` 摘除
- 四个版本的 `MiFindNodeOrParent`：Win7 / Win8 / Win8.1 / Win10 各有不同的 VAD 结构体

**回调注册反检测 (`GetSystemDrvJumpHook`):**
- 遍历模块列表，在10个系统驱动 (tcpipreg.sys, null.sys, beep.sys 等) 中搜索 13 字节 `\xCC` 填充
- 找到后写入 `mov rax, <callback>; nop; jmp rax` (13 字节)
- 将该系统驱动内地址注册为回调入口
- 同时设置模块 Flags |= 0x20

**Inferences:**
- 所有回调（注册表、进程、镜像加载）都借用系统驱动的代码空间作为跳板
- 检测工具查看回调列表时看到的是 null.sys / beep.sys 等系统模块地址
- 0x20 标志可能与 MmVerifierData 验证有关

### 4.3 句柄提权 (`句柄提权.cpp` ~147行)

**Facts:**
- 使用 `ExEnumHandleTable` 枚举进程句柄表
- 找到目标句柄后设置 `GrantedAccessBits = PROCESS_ALL_ACCESS`
- Win7 和 Win8+ 使用不同的回调签名
- Win8+ 版本额外处理 `HandleContentionEvent` push lock

### 4.4 反反作弊 (`反反作弊.cpp` ~86行)

**Facts:**
- 针对 **BattlEye** (BEDaisy.sys)
- 当 BEDaisy.sys 模块加载时，hook 其 IAT 中的三个函数：
  - `ExAllocatePool` → 当 PoolType=PagedPool && Size=24 时返回 NULL
  - `ExAllocatePoolWithTag` → 当 Tag='EB' 时返回 NULL
  - `MmGetSystemRoutineAddress` → 对上述两个函数返回 hook 版本
- IAT hook 使用 `RtlSuperCopyMemory` (MDL) 绕过写保护

**Inferences:**
- BattlEye 用 24 字节 PagedPool 分配做某种检测，返回 NULL 使其检测路径失败
- 三层 hook 确保即使 BE 动态获取函数地址也会拿到 hook 版本

### 4.5 键鼠模拟 (`键鼠模拟.cpp` ~300行)

**Facts:**
- 定位 mouclass/kbdclass 驱动的 `ClassServiceCallback`
- 通过遍历设备扩展内存搜索函数指针（而非 hook）
- 直接调用 `MouseClassServiceCallback` / `KeyboardClassServiceCallback` 注入输入数据
- 支持 mouhid/i8042prt 两种端口驱动

**Inferences:**
- 直接调用 class driver 回调绕过了 SendInput API 级的检测
- 扫描 DeviceExtension 找回调指针是一种经典但脆弱的方法，依赖内存布局

### 4.6 硬件 ID 伪造 (`过机器码.cpp` ~1398行)

**Facts:**
- 项目中最大的单文件
- 使用 IRP 完成例程 (CompletionRoutine) 拦截模式
- 目标：NIC (网卡 MAC)、磁盘序列号、SMBIOS 数据、GPU 信息
- `BuildCompRoutine` 构建拦截：替换 IRP 的 CompletionRoutine，在完成时修改返回数据
- 随机生成函数：`RtlRandomAnsi`, `RtlRandomUnicode`, `RtlRandomAnsiGuid`, `RtlRandomUnicodeGuid`
- 过滤进程：检查调用进程名 hash，排除系统关键进程

### 4.7 内核发包 (`内核发包.cpp` ~405行)

**Facts:**
- 基于 WSK (Winsock Kernel) 实现
- 提供 HTTP POST 功能
- WSK 初始化/清理/socket 创建/连接/发送/接收的完整封装
- 用于用户验证等网络通信

---

## 5. 内存读写方式

### Facts

GsDriver 提供三种内存读写方式 (`'0006'` 命令的 ReadWriteType):

| Type | 方式 | 用途 |
|------|------|------|
| 0 | `MmCopyVirtualMemory(Target → Current)` | 读目标进程内存 |
| 1 | `MmCopyVirtualMemory(Current → Target)` | 写目标进程内存 (普通) |
| 2 | MDL 映射 + `MmMapLockedPagesSpecifyCache` | 写只读内存页 (强写) |

Type 2 (强写) 的流程:
1. `KeStackAttachProcess` 附加到目标进程
2. `MmCreateMdl` + `MmProbeAndLockPages` 锁定目标页
3. `MmMapLockedPagesSpecifyCache` 映射到内核地址空间
4. 通过映射地址写入数据
5. 解锁、解映射、分离进程

限制: Type 2 单次最大 `PAGE_SIZE` (4096 字节)

---

## 6. 与 NinDriver 的关键差异

| 维度 | GsDriver | NinDriver |
|------|----------|-----------|
| 架构 | 双层 (外壳 + 核心) | 单层 (自重映射) |
| 通信 | 注册表回调 (`CmRegisterCallback`) | IOCTL (`IoCreateDriver` + DeviceIoControl) |
| 内存读写 | `MmCopyVirtualMemory` + MDL 强写 | PTE 自映射物理内存直接读写 |
| CR3 获取 | 不需要 (通过 EPROCESS 操作) | PFN 数据库扫描 |
| 偏移来源 | 硬编码查表 (按版本号) | 从内核函数机器码动态提取 |
| 功能范围 | 完整框架 (注入/键鼠/硬件伪造/网络/反检测) | 纯内存读写 + 进程信息查询 |
| 隐蔽手段 | 系统驱动跳板 + VAD 摘除 + PTE 修改 + PFN 清零 | 自重映射 + PE 头清零 + 匿名驱动 |
| 代码规模 | ~7,800 行 | ~880 行 |

---

## 7. 关键数据结构

### DynamicData (DYNDATA)

外壳和核心共享的全局状态，外壳创建并初始化，通过文件传递给核心：

| 字段 | 用途 |
|------|------|
| WinVersion | 版本编码 (Major<<8 \| Minor<<4 \| SP) |
| BuildNumber | Windows Build 号 |
| KernelBase | ntoskrnl.exe 基址 |
| ModuleList | 加载模块链表 (InLoadOrderLinks) |
| PageTables[4] | PTE/PDE/PPE/PXE 基地址 |
| DriverBase | 核心驱动映射基址 |
| NtCreateThreadEx | SSDT 地址 |
| NtProtectVirtualMemory | SSDT 地址 |
| VadRoot, PrcessId, Protection, ... | EPROCESS 字段偏移 |
| UserVerify | 用户验证标志 |

### HOOK_NOTIFY_BUFFER

用于回调注册的跳板管理结构：

| 字段 | 用途 |
|------|------|
| HookPoint | 系统驱动中的 CC 填充地址 |
| Cookie | CmRegisterCallback 返回的 cookie |
| Enable | 当前启用状态 |
| OldBytes | 原始字节 (13 bytes) |
| NewBytes | 跳转代码 (mov rax, addr; nop; jmp rax) |

---

## 8. Open Questions

1. **VMProtect 保护范围**：项目链接了 VMProtectDDK64.lib，但源码中未看到 `VMProtectBegin/End` 宏调用。VMProtect 在最终编译版本中保护了哪些函数？
2. **DrvData 占位符**：`驱动文件.h` 中 `static unsigned char DrvData[1000]` 全零——实际核心驱动字节码应在编译流程中填入，但源码中没有看到填充机制。核心驱动是如何嵌入外壳的？
3. **UserVerify 机制**：`'0001'` 命令总是返回 ERROR_成功 并设置 UserVerify，但 RegisterVerify 函数在头文件中声明却未在 cpp 中实现。实际验证逻辑是否被删除？
4. **隐藏进程命令 (`'0009'`)**：代码已注释为 `ERROR_失败`，ZwHideProcess 函数未实现。这是遗留代码还是未完成功能？
5. **PFN 清零的安全性**：`AllocMemory_x86/x64` 中将 MDL PFN 数组清零——这会导致物理页无法被系统回收，是否有内存泄漏风险？
6. **自定义错误码 vs NTSTATUS**：通信使用 0xE0000000 系列作为成功/失败码，这些不是标准 NTSTATUS 值。用户态如何区分驱动错误和系统错误？

---

## 9. 技术亮点与风险

### 技术亮点

1. **系统驱动跳板**：在合法系统驱动 (.text 段 CC 填充) 中写入跳转代码，使回调注册指向系统模块
2. **注册表通信**：完全避开设备对象和 IRP 路径，不创建任何驱动对象
3. **三级注入隐藏**：VAD 摘除 / PTE 属性修改 / MDL 物理页 + PFN 清零，分级对抗不同检测手段
4. **IAT hash 匹配**：手动映射的核心驱动使用 hash 而非字符串解析导入表

### 风险与局限

1. **EPROCESS 偏移硬编码**：每个新 Windows 版本都需要手动添加偏移值
2. **CC 填充搜索**：依赖系统驱动有足够的对齐填充，如果编译器优化改变可能失败
3. **VAD 摘除不可逆**：进程退出时才恢复，中间如果蓝屏则 VAD 树损坏
4. **GSDrv.bin 竞态**：文件写入和读取之间理论上存在竞态窗口，虽然外壳线程等待核心完成
