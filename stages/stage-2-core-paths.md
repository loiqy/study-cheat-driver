# Stage 2 — 核心路径和关键模块

## 状态：待开始

## 目标
深入各项目最核心的代码路径：驱动通信机制、内存读写实现、VT-x 进出流程等。

## 完成标准
- [ ] 每个项目有 `notes/{project}-core-paths.md`
- [ ] 能精确指出关键函数及其职责
- [ ] 理清用户态 ↔ 内核态通信全链路

## 预期产出
| 文件 | 说明 |
|------|------|
| `notes/gsdriver-core-paths.md` | GsDriver 核心代码路径 |
| `notes/nindriver-core-paths.md` | NinDriver 核心代码路径 |
| `notes/vtdriver-core-paths.md` | VT_Driver 核心代码路径 |

## 前置条件
Stage 1 完成
