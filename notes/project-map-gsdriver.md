# GsDriver 项目地图

> 生成时间：2026-03-28 | Stage 0 产出

## 项目初步判断

### Facts
- 项目名称: GsDriver，位于 `../GsDriver-master/`
- **双层驱动架构**：驱动外壳 (loader) + 驱动核心 (功能模块)
- VS2017 解决方案，支持 x64/x86/ARM/ARM64
- 使用 VMProtect 代码保护 (VMProtectDDK64.lib)
- 文件和变量名全部使用中文命名

### Inferences
- "出租驱动" 的命名暗示这是一个面向多用户/租户的驱动框架
- 外壳负责加载和初始化，核心负责全部功能——这种分层可能是为了热替换核心模块
- 使用注册表通信而非 IOCTL，可能是为了避免被常规驱动通信检测拦截

### Open Questions
- 外壳如何定位并加载核心驱动的 .sys 文件？映射方式？
- 注册表通信的完整协议是怎样的？
- VMProtect 保护的范围有多大？保护了哪些函数？

---

## 语言与构建信号

| 属性 | 值 |
|------|-----|
| 语言 | C/C++ (混合 .cpp 和 .c 文件) |
| 构建系统 | Visual Studio 2017 (.sln + .vcxproj) |
| 平台 | 主要 x64，支持 x86/ARM/ARM64 |
| 保护 | VMProtect DDK |
| 命名风格 | 中文文件名和变量名 |

---

## 顶层目录结构

```
GsDriver-master/
├── .gitattributes
├── .gitignore
├── README.md
├── 驱动外壳.sln                    ← 解决方案入口
├── 驱动外壳/                       ← 子项目 1: 加载器/外壳
│   ├── 驱动外壳.vcxproj
│   ├── 驱动外壳.vcxproj.filters
│   ├── 驱动外壳.h                  ← 主头文件
│   ├── 驱动外壳.cpp                ← DriverEntry 所在文件
│   ├── 驱动文件.h                  ← 文件路径/结构定义
│   ├── 全局变量.h / .cpp           ← 全局变量
│   ├── 导出函数.h / .cpp           ← 工具函数 (文件IO、内存、签名搜索)
│   └── 资源文件/
│       ├── NativeEnums.h           ← Windows 内核枚举
│       ├── NativeStructs.h         ← Windows 内核结构体 (EPROCESS等)
│       └── VMProtect/              ← VMProtect DDK
└── 驱动核心/                       ← 子项目 2: 功能核心
    ├── 驱动核心.vcxproj
    ├── 驱动核心.h                  ← 主头文件
    ├── 驱动核心.cpp                ← DriverEntry (线程模式)
    ├── 全局变量.h / .cpp           ← DynamicData 全局变量
    ├── 导出函数.h / .cpp           ← 内核函数动态获取/包装
    ├── 通讯回调.h / .cpp           ← 注册表通信机制 (最大模块 1188行)
    ├── 注入回调.h / .cpp           ← DLL 注入引擎 + ShellCode
    ├── 注入代码.h                  ← x86/x64 ShellCode 字节码
    ├── 进程回调.h / .cpp / .c      ← 进程保护 + VAD 操作
    ├── 键鼠模拟.h / .cpp / .c      ← 键盘/鼠标输入模拟
    ├── 句柄提权.h / .cpp           ← 句柄权限提升
    ├── 反反作弊.h / .cpp           ← 反 BattlEye 等
    ├── 自定声明.h                  ← 错误码 + PTE 宏
    ├── 过机器码.h / .cpp           ← 硬件 ID 伪造 (1398行)
    ├── 内核发包.h / .cpp           ← WSK 网络通信
    └── 资源文件/                   ← 同外壳，NativeStructs + VMProtect
```

---

## 入口点与初始化

### 驱动外壳 DriverEntry
- **位置**: `驱动外壳/驱动外壳.cpp:192`
- **签名**: `auto DriverEntry(PDRIVER_OBJECT, PUNICODE_STRING) -> NTSTATUS`
- **流程**:
  1. 初始化 PTE 页表地址 (支持 Win7-Win10+)
  2. 读取 ntdll.dll 获取系统函数地址表
  3. 定位内核基址和模块列表
  4. 计算 Windows 版本特定偏移量 (VAD Root, PID Offset 等)
  5. 加载并映射核心驱动到内存

### 驱动核心 DriverEntry
- **位置**: `驱动核心/驱动核心.cpp:58`
- **签名**: `auto DriverEntry() -> VOID` (无参数，线程模式)
- **流程**:
  1. `VariateInit()` — 变量初始化
  2. `DriverStart()` — 驱动启动
  3. `RegisterNotifyInit(TRUE)` — 注册回调
  4. `PsTerminateSystemThread(NULL)` — 以系统线程结束

---

## 模块职责表

### 驱动外壳 (~2,083 行)

| 文件 | 行数 | 职责 |
|------|------|------|
| 驱动外壳.cpp | 220 | 驱动入口、PTE 初始化、内核基址定位、版本偏移计算 |
| 导出函数.cpp | 823 | 文件 IO、内存管理、签名搜索、导出表遍历、核心驱动加载 |
| 全局变量.cpp | — | 全局变量初始化 |
| NativeStructs.h | — | EPROCESS, ETHREAD 等内核结构体定义 (多版本) |

### 驱动核心 (~5,755 行)

| 文件 | 行数 | 职责 |
|------|------|------|
| 驱动核心.cpp | 68 | 入口、初始化、回调注册 |
| 导出函数.cpp | 1194 | 动态获取并包装内核函数 |
| 通讯回调.cpp | 1188 | **注册表跳板通信**、命令分发 |
| 注入回调.cpp | 970 | DLL 注入、ShellCode 执行、PTE 权限修改 |
| 过机器码.cpp | 1398 | **最大文件**，多种硬件 ID 伪造 |
| 进程回调.cpp/c | 558 | 进程保护、VAD 树遍历/修改 |
| 内核发包.cpp | 405 | WSK 网络通信 (HTTP POST) |
| 键鼠模拟.cpp/c | 300 | kbdclass/mouclass 输入模拟 |
| 句柄提权.cpp | 147 | 句柄表枚举、权限提升 |
| 反反作弊.cpp | 86 | 反 BattlEye 对抗 |

---

## 关键技术线索

### 通信机制：注册表跳板 (非常规)
- **不使用 IOCTL**，而是用注册表回调 (`CmRegisterCallback`) 作为通信通道
- 用户态写注册表 → 触发驱动回调 → 解析 Type 字段分发命令
- Type 编码: `'0000'`=通讯测试, `'0001'`=用户验证, `'0002'`=离线注入...

### 内存操作
- `MmCopyVirtualMemory` — 进程间内存拷贝
- 直接 PTE 修改 — 改变内存页属性
- VAD 树操作 — 隐藏进程内存区域
- CR3 寄存器 + PML4 遍历 — 物理地址访问

### PTE 宏 (自定声明.h)
```c
MiGetPxeAddress(BASE, VA)  // PML4E
MiGetPpeAddress(BASE, VA)  // PDPTE
MiGetPdeAddress(BASE, VA)  // PDE
MiGetPteAddress(BASE, VA)  // PTE
```

### 隐藏策略
1. PTE 操作隐藏注入的 DLL
2. VAD 树修改隐藏进程内存区域
3. 自删除驱动文件 (`RtlForceDeleteFile`)
4. DriverEntry 返回 `0xE0000000` (非标准返回值)

---

## 下一步建议重点阅读

| 优先级 | 文件 | 原因 |
|--------|------|------|
| P0 | 驱动外壳/驱动外壳.cpp | 理解加载流程和外壳→核心的桥接 |
| P0 | 驱动核心/通讯回调.cpp | 理解注册表通信协议的完整实现 |
| P1 | 驱动核心/注入回调.cpp | 理解 DLL 注入的 PTE 级实现 |
| P1 | 驱动核心/过机器码.cpp | 最大文件，理解硬件 ID 伪造策略 |
| P2 | 驱动核心/进程回调.cpp | VAD 树操作和内存隐藏 |
| P2 | 驱动外壳/导出函数.cpp | 工具函数库，理解签名搜索等基础能力 |
