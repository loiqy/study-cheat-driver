# VT_Driver 项目地图

> 生成时间：2026-03-28 | Stage 0 产出

## 项目初步判断

### Facts
- 项目名称: VT_Driver，位于 `../VT_Driver/`
- **VT-x 虚拟化驱动** + 完整功能框架
- VS2017 解决方案，2 个项目: VT_Driver (驱动) + capstone_static_winkernel (静态库)
- 三个项目中**规模最大、复杂度最高**
- 包含完整的 VMX 生命周期: VMXON → VMLAUNCH → VM-exit 处理 → VMRESUME
- 包含 EPT (扩展页表) 实现
- 包含 PatchGuard 绕过 PoC
- 使用 capstone 反汇编引擎 (仅 x86 架构)
- 使用 VMProtect 保护

### Inferences
- 这是一个 **Type-2 Hypervisor** (运行在 OS 之上)，不是独立虚拟机监控器
- VT-x 层既提供内存隐藏能力，又提供 API 拦截能力 (EPT 钩取)
- PatchGuard 绕过模块的存在说明驱动需要做内核修改 (SSDT hook 等) 而这些修改需要避开 KPP 检测
- capstone 用于 LDE (Length Disassembler Engine)，支撑 inline hook 的跳板构建
- 架构比 GsDriver 和 NinDriver 复杂一个量级——VT-x 层是额外的抽象层

### Open Questions
- EPT 钩取的具体场景是什么？钩了哪些内核函数？
- 65 个 VM-exit 处理程序中，哪些是真正实现了逻辑的，哪些只是占位？
- PatchGuard 绕过的实际可靠性如何？Win10 和 WinX 两个 PoC 的区别？
- capstone 是否只在初始化时使用，还是运行时持续使用？
- VMCALL 超调用的完整接口是什么？

---

## 语言与构建信号

| 属性 | 值 |
|------|-----|
| 语言 | C/C++ (含内联汇编接口) |
| 构建系统 | Visual Studio 2017 (.sln + .vcxproj) |
| 平台 | x64 为主 |
| 依赖 | capstone 反汇编引擎 (静态链接) |
| 保护 | VMProtect DDK |
| 特殊 | 需要 CPU 支持 VT-x (Intel) |

---

## 顶层目录结构

```
VT_Driver/
├── VT_Driver.sln                     ← 解决方案入口
│
├── Intel-vt/                         ← VT-x 虚拟化核心层
│   ├── asm.h                         ← VMX 指令 + 汇编例程接口
│   ├── common_vt.h                   ← VMCS 字段编码 + VM-exit 原因 (604行)
│   ├── vmx.h / vmx.cpp              ← VMX 初始化 + VMCS 读写 (377+545行)
│   ├── ept.h / ept.cpp              ← EPT 分页结构 + 恒等映射 (76+350行)
│   ├── VmxExitHandlers.cpp           ← 65 个 VM-exit 处理程序 (627行)
│   ├── vt.h / vt.cpp                ← 全局数据 + HV 加载入口 (33+324行)
│   ├── hvm.h / hvm.cpp              ← 硬件功能检查 (21+124行)
│   ├── public.h / public.cpp        ← EPT_HOOK 结构 + 工具函数 (93+294行)
│   ├── LDasm.h / LDasm.cpp          ← 线性反汇编器 (41+807行)
│   └── kernel_stl.cpp               ← 内核 STL 实现 (151行)
│
├── VT_Driver/                        ← 驱动框架和功能模块层
│   ├── Main.cpp / Main.h            ← DriverEntry + IOCTL 分发 (449+51行)
│   ├── Common_Head.h                ← IOCTL 码 + 公共定义 (333行)
│   ├── Api.h                        ← 内核 API 声明
│   ├── Utils.cpp / Utils.h          ← 工具类: hook/SSDT/模式扫描 (821+104行)
│   ├── Memory.cpp / Memory.h        ← 内存读写 (237+42行)
│   ├── Process.cpp / Process.h      ← 进程管理 (187+74行)
│   ├── Exception.cpp / Exception.h  ← 异常处理 (491+8行)
│   ├── Intercept.cpp / Intercept.h  ← 网络包拦截 AFD (129+78行)
│   ├── ProtectWindow.cpp / .h       ← 窗口保护 SSSDT hook (226+61行)
│   ├── Registry.cpp / Registry.h    ← 注册表操作 (115+6行)
│   ├── ShellCode.h                  ← Shellcode 模板 (804行)
│   ├── VMProtectDDK.h               ← VMProtect 宏
│   ├── Bypass/                      ← 反调试/反分析
│   │   ├── Bypass.cpp / Bypass.h    ← ob 回调 hook + 内存访问隐藏
│   ├── LoadDriver/                  ← 驱动加载器
│   │   ├── LeiLeiLoad.cpp / .h      ← 驱动加载/卸载
│   │   └── GetDrvObject.cpp / .h    ← 驱动对象获取
│   ├── PatchGuard/                  ← PatchGuard/KPP 绕过
│   │   ├── PGhead.h
│   │   ├── Win10PatchguardPoc.cpp/.h
│   │   └── WinXPatchguardPoc.cpp/.h
│   └── Filter/                      ← 文件系统过滤
│       ├── MiniFilter.cpp / .h      ← FLT 框架文件隐藏
│
└── capstone/                         ← 反汇编引擎 (第三方)
    ├── arch/X86/                    ← x86 指令集支持
    ├── include/capstone.h           ← 公共 API
    ├── msvc/capstone_static_winkernel.vcxproj  ← 内核静态库配置
    └── contrib/windows_kernel/      ← 内核环境适配层
```

---

## 入口点与初始化

### DriverEntry
- **位置**: `VT_Driver/Main.cpp` — `Driver::DriverInit()` (Main.h:40)
- **调用约定**: `__fastcall`
- **流程**:
  1. 创建设备对象和符号链接
  2. 注册 IRP 分发函数
  3. 初始化 VT-x 子系统 (`LoadHV()`)
  4. 注册进程创建回调
  5. 注册 NtDeviceIoControlFile 钩取

### LoadHV() — 虚拟化加载
- **位置**: `Intel-vt/vt.cpp:67`
- **流程**:
  1. `AllocGlobalData()` — 分配全局 VT 数据结构
  2. 预分配 512 页 EPT 页表
  3. `HvmCheckFeatures()` — 检查 CPU VT-x 支持
  4. 构建 EPT 恒等映射
  5. 在每个 CPU 上启动 VMX: VMXON → 设置 VMCS → VMLAUNCH

### 关键回调
- `ControlCenter()` — IOCTL 分发中心 (Main.h:47)
- `Fake_NtDeviceIoControlFile()` — 钩取的 IOCTL 代理 (Main.h:46)
- `CreateCallback()` — 进程创建回调 (Main.h:44)
- `DriverUnload()` — 卸载回调 (Main.h:45)

---

## IOCTL 接口

| 代码 | 宏名 | 功能 |
|------|------|------|
| 0x9800 | PROTO_TEST | 测试 |
| 0x9804 | PROTO_READWRITE | 内存读写 |
| 0x9808 | PROTO_READWRITE_TX | 内存读写 (传输模式) |
| 0x980C | PROTO_SET_PROCESS | 设置目标进程 |
| 0x9810 | PROTO_GET_MODULE | 获取模块 |
| 0x9814 | PROTO_FILE_PROTECTION | 文件保护 |
| 0x9818 | PROTO_DISABLE_PG | 禁用 PatchGuard |
| 0x981C | PROTO_DELETE_FILES | 删除文件 |
| 0x9820 | PROTO_PACKET_FEATURE | 网络包特征替换 |
| 0x9824 | PROTO_PROTECT_WINDOW | 窗口保护 |
| 0x9828 | PROTO_DEBUG_GINGTOOL | 调试工具 |

---

## 模块职责表

### Intel-vt/ 层 — VT-x 虚拟化核心 (~3,601 行)

| 文件 | 行数 | 职责 |
|------|------|------|
| common_vt.h | 604 | VMCS 字段编码 (800+)、VM-exit 原因 (65种)、MSR 定义 |
| vmx.cpp | 545 | VMCS 读写、硬件支持检查、中断注入、MTF 切换 |
| vmx.h | 377 | VMX 初始化接口、特性枚举 |
| VmxExitHandlers.cpp | 627 | **65 个 VM-exit 处理程序**: CPUID/RDTSC/MSR/EPT 冲突/VMCALL... |
| ept.cpp | 350 | EPT 恒等映射构建、表更新、冲突处理 |
| vt.cpp | 324 | 全局数据分配、EPT 预分配 (512页)、LoadHV 完整流程 |
| public.cpp | 294 | 工具函数、虚拟化特性检查辅助 |
| LDasm.cpp | 807 | 线性反汇编器 (指令解析、长度计算) |
| kernel_stl.cpp | 151 | 内核环境 STL 兼容层 (malloc/free) |
| hvm.cpp | 124 | 硬件功能检查实现 |

### VT_Driver/ 层 — 驱动框架 (~4,000+ 行)

| 文件 | 行数 | 职责 |
|------|------|------|
| Utils.cpp | 821 | **最大文件**: LDE 初始化、SSDT 访问、API hook、模式扫描 |
| ShellCode.h | 804 | Shellcode 模板和注入代码 |
| Exception.cpp | 491 | 异常向量、中断模式、异常恢复 |
| Main.cpp | 449 | DriverEntry、设备管理、IOCTL 分发 |
| Common_Head.h | 333 | IOCTL 码、系统信息枚举、公共宏 |
| Memory.cpp | 237 | 跨进程内存读写 (直接方式 + 系统线程方式) |
| ProtectWindow.cpp | 226 | SSSDT hook 窗口隐藏 |
| Process.cpp | 187 | 模块枚举、PEB 遍历、内存申请 |
| Intercept.cpp | 129 | AFD 协议网络包拦截 |
| Registry.cpp | 115 | 注册表操作 |

### 子模块

| 目录 | 职责 |
|------|------|
| Bypass/ | 反调试: ob 回调 hook、内存访问 hook、WOW64 上下文 hook |
| PatchGuard/ | KPP 绕过 PoC (Win10 + WinX 两个版本) |
| Filter/ | FLT 框架文件隐藏 (MiniFilter) |
| LoadDriver/ | 驱动加载/卸载、驱动对象获取 |

---

## VT-x 技术细节

### VMX 指令使用
| 指令 | 位置 | 用途 |
|------|------|------|
| VMXON | VmxVMEntry (asm.h) | 进入 VMX 操作模式 |
| VMLAUNCH | VmxVMEntry (asm.h) | 首次启动虚拟机 |
| VMRESUME | VmxpResume (asm.h) | 恢复虚拟机执行 |
| VMREAD | vmx.cpp:44 | 读取 VMCS 字段 |
| VMWRITE | vmx.cpp | 写入 VMCS 字段 |
| VMCALL | asm.h | 虚拟机→主机超调用 |
| INVEPT | asm.h | EPT 转换失效 |
| INVVPID | asm.h | VPID 转换失效 |

### EPT 架构
- **4 级分页**: PML4 → PDPTE → PDE → PTE
- **权限独立控制**: Read / Write / Execute
- **钩取结构** (R3EPT_HOOK):
  - 代码页和数据页分离
  - 执行时映射到修改后的代码页，读取时映射到原始数据页
  - 实现透明钩取 (对读取检查不可见)

### VM-exit 关键处理程序
| 原因码 | 名称 | 用途 |
|--------|------|------|
| 10 | CPUID | 虚拟化处理器信息 |
| 16 | RDTSC | 时间戳防检测 |
| 18 | VMCALL | 超级管理程序调用接口 |
| 28 | CR_ACCESS | 控制寄存器访问拦截 |
| 31/32 | MSR_READ/WRITE | MSR 访问拦截 |
| 48 | EPT_VIOLATION | EPT 权限冲突→钩取分发 |
| 49 | EPT_MISCONFIG | EPT 配置错误处理 |

---

## 关键技术线索

### 通信机制：IOCTL + NtDeviceIoControlFile 钩取
- 不是直接 DeviceIoControl → 驱动
- 而是钩取 `NtDeviceIoControlFile`，拦截特定 IOCTL 码
- 这样即使设备对象被发现，通信路径也不走常规分发

### 内存操作
- **直接方式**: `ReadMemory`/`WriteMemory`
- **系统线程方式**: `SystemReadMemory`/`SystemWriteMemory` — 切换 CR3 到目标进程
- **COPY_MEMORY 结构**: `{address, value, size, type(1=读/2=写)}`

### 内核 hook 体系
- **SSDT hook**: `GetSSDTBase()`/`GetSSDTEntry()` 访问系统服务表
- **SSSDT hook**: Win32 系统调用表 (窗口保护)
- **Inline hook**: LDasm + capstone 计算跳板
- **EPT hook**: VT-x 层透明钩取

### PatchGuard 绕过
- 两个 PoC: Win10 和 WinX
- 必须绕过 KPP 才能安全做 SSDT/内核修改

### 隐藏策略
- MiniFilter 文件系统过滤隐藏文件
- 窗口 API hook 隐藏进程窗口
- Bypass 模块隐藏内存访问和进程查询
- EPT 分页隐藏内存修改

---

## 架构分层图

```
┌─────────────────────────────────────┐
│         用户态应用                    │
│    NtDeviceIoControlFile (被钩取)    │
└──────────────┬──────────────────────┘
               │ IOCTL
┌──────────────▼──────────────────────┐
│  VT_Driver/ — 驱动框架层             │
│  ├─ Main: IOCTL 分发                │
│  ├─ Memory: 进程内存读写             │
│  ├─ Process: 进程/模块管理           │
│  ├─ Utils: SSDT + inline hook       │
│  ├─ Bypass: 反调试钩取              │
│  ├─ PatchGuard: KPP 绕过            │
│  ├─ Filter: 文件隐藏                │
│  ├─ Intercept: 网络包拦截           │
│  └─ ProtectWindow: 窗口隐藏         │
└──────────────┬──────────────────────┘
               │ VMCALL / EPT hook
┌──────────────▼──────────────────────┐
│  Intel-vt/ — VT-x 虚拟化层          │
│  ├─ vmx: VMCS 管理 + VMX 生命周期   │
│  ├─ ept: 扩展页表 + 内存权限控制     │
│  ├─ VmxExitHandlers: 65 个退出处理  │
│  ├─ LDasm: 反汇编 (hook 支撑)       │
│  └─ hvm: 硬件功能检查               │
└──────────────┬──────────────────────┘
               │
        ┌──────▼──────┐
        │  Intel CPU   │
        │  VT-x 硬件   │
        └─────────────┘
```

---

## 下一步建议重点阅读

| 优先级 | 文件 | 原因 |
|--------|------|------|
| P0 | Main.cpp | 理解入口和 IOCTL 分发 + VT 初始化触发 |
| P0 | vt.cpp | 理解 LoadHV 完整流程 (VMX 启动链) |
| P0 | VmxExitHandlers.cpp | VM-exit 处理是 VT-x 驱动的核心 |
| P1 | ept.cpp | EPT 恒等映射和钩取机制 |
| P1 | vmx.cpp | VMCS 字段设置细节 |
| P1 | Utils.cpp | SSDT 访问和 hook 体系 |
| P2 | Bypass/Bypass.cpp | 反调试策略 |
| P2 | PatchGuard/*.cpp | KPP 绕过实现 |
| P2 | public.h | EPT_HOOK 和 GUEST_STATE 结构 |
