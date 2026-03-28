# Stage 1 — 单项目结构理解

## 状态：✅ 已完成 (3/3)

## 目标
逐项目读入口、理清调用链、标记关键数据结构。

## 完成标准
- [x] 每个项目有 `notes/{project}-structure.md` — NinDriver ✅ GsDriver ✅ VT_Driver ✅
- [x] 能用一段话说清每个项目"做了什么、怎么做的" — NinDriver ✅ GsDriver ✅ VT_Driver ✅
- [x] 标记出 DriverEntry、IOCTL handler、主要回调 — NinDriver ✅ GsDriver ✅ VT_Driver ✅

## 预期产出
| 文件 | 说明 | 状态 |
|------|------|------|
| `notes/nindriver-structure.md` | NinDriver 结构分析 | ✅ |
| `notes/gsdriver-structure.md` | GsDriver 结构分析 | ✅ |
| `notes/vtdriver-structure.md` | VT_Driver 结构分析 | ✅ |

## 已完成项目的核心发现

### NinDriver
- 双阶段初始化 (DriverEntry → MapEntry 自重映射)
- PTE 自映射物理内存读写
- PFN 数据库扫描获取 CR3

### GsDriver
- 双层架构 (外壳手动映射核心)
- 注册表回调通信 (CmRegisterCallback, 22+ 命令码)
- 系统驱动跳板反检测
- 完整功能框架 (注入/键鼠/硬件伪造/反BE/网络)

### VT_Driver
- Type-2 Hypervisor (每 CPU VMXON → VMLAUNCH，OS 降级为 Guest)
- EPT 代码页/数据页分离钩取 (MTF 乒乓切换)
- 四层 Hook 体系 (EPT/Inline/SSDT/SSSDT)
- NtDeviceIoControlFile 钩取通信，11 个 IOCTL 命令
- 完整功能框架 (内存/进程/PatchGuard/文件隐藏/窗口隐藏/网络拦截/反调试)

## 前置条件
Stage 0 完成
