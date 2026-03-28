# Stage 1 — 单项目结构理解

## 状态：进行中 (2/3)

## 目标
逐项目读入口、理清调用链、标记关键数据结构。

## 完成标准
- [x] 每个项目有 `notes/{project}-structure.md` — NinDriver ✅ GsDriver ✅
- [x] 能用一段话说清每个项目"做了什么、怎么做的" — NinDriver ✅ GsDriver ✅
- [x] 标记出 DriverEntry、IOCTL handler、主要回调 — NinDriver ✅ GsDriver ✅

## 预期产出
| 文件 | 说明 | 状态 |
|------|------|------|
| `notes/nindriver-structure.md` | NinDriver 结构分析 | ✅ |
| `notes/gsdriver-structure.md` | GsDriver 结构分析 | ✅ |
| `notes/vtdriver-structure.md` | VT_Driver 结构分析 | 待做 |

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

## 前置条件
Stage 0 完成
