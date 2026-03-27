# NinDriver 项目地图

> 生成时间：2026-03-28 | Stage 0 产出

## 项目初步判断

### Facts
- 项目名称: NinDriver，位于 `../NinDriver/`
- **单驱动架构**：一个 .sys 文件，3 个源文件 + 辅助头文件
- VS2019+ 解决方案，WDM 驱动类型，C++17
- 提供**物理内存读写**和**进程信息查询**功能
- 含详细的 CLAUDE.md 项目文档
- 有编译输出 (x64/Release/)，说明已成功构建过
- 设备名伪装为 NVIDIA GPU PCI 路径

### Inferences
- 代码量最小 (~880 行源码)，架构最简洁——适合作为入门学习对象
- 自重映射技术 (IoCreateDriver) 和文件自删除是核心隐蔽手段
- 通过 PFN 数据库扫描获取 CR3，这是一种不依赖 EPROCESS 偏移表的方法
- 运行时动态解析内核偏移，避免了硬编码版本表——比 GsDriver 更灵活

### Open Questions
- `MmAllocateIndependentPages` 是未导出函数，通过特征码扫描定位——具体 pattern 是什么？
- PFN 数据库扫描获取 CR3 的准确率和性能如何？
- 自重映射后，原驱动 .sys 卸载，新驱动对象如何管理生命周期？

---

## 语言与构建信号

| 属性 | 值 |
|------|-----|
| 语言 | C++ (C++17) |
| 构建系统 | Visual Studio 2019+ (.sln + .vcxproj) |
| 平台 | x64 Release 为主 |
| 驱动类型 | WDM (Windows Driver Model) |
| SDK | 10.0.19041.0 |
| 字符串保护 | XOR 编译时加密 (skCrypt) |
| 优化 | Release 下 Disabled (调试方便?) |

---

## 顶层目录结构

```
NinDriver/
├── CLAUDE.md                        ← 项目文档 (非常详细)
├── Nin Driver.sln                   ← 解决方案
└── Nin Driver/
    ├── Nin Driver.vcxproj           ← 项目配置
    ├── Nin Driver.vcxproj.filters
    ├── Nin Driver.vcxproj.user
    ├── NinDriver.inf                ← 驱动安装配置
    ├── Driver.cpp                   ← 入口 + IOCTL 分发 (191行)
    ├── Driver.h                     ← IOCTL 码 + DataStruct 定义
    ├── NativeStruct.h               ← 运行时偏移 + PEB/LDR 结构
    ├── XorString.h                  ← 编译时 XOR 字符串加密
    ├── 导出函数/
    │   ├── Export.cpp               ← 偏移初始化、特征码扫描、工具函数 (396行)
    │   └── Export.h
    ├── 内存管理/
    │   ├── Memory.cpp               ← 物理内存读写核心 (~260行)
    │   └── Memory.h
    └── x64/Release/                 ← 编译输出
```

---

## 入口点与初始化

### DriverEntry
- **位置**: `Nin Driver/Driver.cpp:133`
- **签名**: `EXTERN_C NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObj, PUNICODE_STRING RegString)`
- **核心流程 — 自重映射**:
  1. 计算 DispatchCreate/Close/Ioctl 相对驱动基址的偏移
  2. 获取驱动 LDR 表项
  3. **强制删除驱动文件** (消除磁盘痕迹)
  4. `InitSystemOffsets()` — 动态解析内核结构偏移
  5. `MmAllocateIndependentPages()` — 分配独立物理页
  6. 设置页面保护为 RWX
  7. 将驱动镜像完整复制到新地址
  8. `IoCreateDriver()` — 创建重映射的驱动实例
  9. **返回 STATUS_UNSUCCESSFUL** — 让系统卸载原始镜像

### MapEntry (真正的入口)
- **位置**: `Driver.cpp:93-131`
- **职责**: 在重映射地址空间中执行
  1. 创建设备对象 (伪装为 PCI NVIDIA 设备)
  2. 创建符号链接
  3. 关联 IRP 分发函数到重映射地址

### IRP 分发
- `DispatchCreate` (Driver.cpp:79) — 直接完成
- `DispatchClose` (Driver.cpp:86) — 直接完成
- `DispatchIoctl` (Driver.cpp:11) — IOCTL 分发核心

---

## IOCTL 接口

| 代码 | 宏名 | 功能 | 关键参数 |
|------|------|------|----------|
| 0x9D7 | IOCTL_GETPROCESSPID | 按名称获取 PID | Name |
| 0xE4B | IOCTL_GETPROCESSCR3 | 获取进程 CR3 | ProcessPid |
| 0xA1C | IOCTL_GETPROCESSMODULE | 获取模块基址 | ProcessPid, Name |
| 0xF62 | IOCTL_READMEMORY | 读取物理内存 | ProcessCr3, TargetAddress, Length, Buffer |
| 0xB8D | IOCTL_WRITEMEMORY | 写入物理内存 | ProcessCr3, TargetAddress, Length, Buffer |
| 0xC3E | IOCTL_GETKEYSTATE | 内核键盘状态 | VirtualKey |

**通信结构体**:
```c
typedef struct _DataStruct {
    PCHAR   Name;           // 进程/模块名
    ULONG   ProcessPid;     // 进程 ID
    ULONG64 ProcessCr3;     // 页表基址
    ULONG64 TargetAddress;  // 目标地址
    ULONG64 Length;         // 操作长度
    PVOID   Buffer;         // 数据缓冲区
    SHORT   VirtualKey;     // 虚拟键码
} DataStruct;
```

---

## 模块职责表

| 模块 | 文件 | 行数 | 职责 |
|------|------|------|------|
| 驱动框架 | Driver.cpp/h | 191+37 | 入口、自重映射、IOCTL 分发 |
| 导出函数 | Export.cpp/h | 396+26 | 偏移初始化、特征码扫描、进程查询、文件删除、键盘读取、日志 |
| 内存管理 | Memory.cpp/h | ~260+14 | 物理内存读写、地址转换、CR3 查询、模块枚举 |
| 结构定义 | NativeStruct.h | ~115 | 运行时偏移 (Offsets namespace)、PEB/LDR 结构 |
| 字符串加密 | XorString.h | 124 | skCrypter 编译时 XOR 加密模板 |

---

## 关键技术线索

### 自重映射 (Self-Remapping) — 核心隐蔽技术
1. 分配独立物理页 (`MmAllocateIndependentPages` — 未导出，特征码定位)
2. 复制驱动镜像到新地址
3. 通过 `IoCreateDriver` 创建新驱动对象指向新地址
4. 原始 DriverEntry 返回失败，系统自动卸载原镜像
5. 结果：内存中存在一个"无源"驱动，不在正常驱动列表中

### 物理内存读写
- **地址转换**: 从 CR3 开始遍历 x64 四级页表 (PML4→PDPT→PDT→PT)
- **读写方式**: 自映射 PTE 技术，在自旋锁保护下修改 PTE + `__invlpg` 刷新 TLB
- **跨页处理**: 循环调用单页函数 + `__try/__except` 异常保护

### CR3 获取 — PFN 数据库扫描
- 不依赖 EPROCESS 偏移表
- 遍历物理内存范围，扫描 PFN 数据库匹配 EPROCESS
- 比硬编码偏移更通用，但性能可能更差

### 动态偏移解析
- 通过 `MmGetSystemRoutineAddress` 获取导出函数地址
- 反汇编函数提取字段偏移 (如 `PsGetProcessId` 第 3 字节 = UniqueProcessId 偏移)
- 避免硬编码 Windows 版本偏移表

### 设备名伪装
```
\\Device\\PCI#VEN_10DE&DEV_2684#{7f8a3c91-5e2d-4b6c-9a1f-8e4b7d3c5a92}
```
伪装为 NVIDIA RTX 4090 的 PCI 设备路径。

---

## 下一步建议重点阅读

| 优先级 | 文件 | 原因 |
|--------|------|------|
| P0 | Driver.cpp | 理解完整的自重映射流程 (仅 191 行，最佳起点) |
| P0 | Memory.cpp | 理解物理内存读写和 PTE 自映射技术 |
| P1 | Export.cpp | 理解偏移动态解析和特征码扫描 |
| P2 | NativeStruct.h | 理解运行时偏移布局和 PEB/LDR 结构 |
| P2 | CLAUDE.md | 已有详细文档，可交叉验证分析结果 |
