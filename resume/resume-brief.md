# Resume Brief — 快速接续文件

> 供新 Claude 会话快速了解当前学习状态。最后更新：2026-03-28

## 仓库用途
这是一个**中文学习仓库**，不是应用项目。用于系统性研究 3 个 Windows 内核驱动项目。

## 三个研究对象

| 项目 | 路径 | 定位 | 规模 |
|------|------|------|------|
| GsDriver | `../GsDriver-master/` | 双层驱动 (外壳+核心)，注册表通信 | ~7,800行 |
| NinDriver | `../NinDriver/` | 单驱动，自重映射，物理内存 IOCTL | ~880行 |
| VT_Driver | `../VT_Driver/` | VT-x 虚拟化 + 完整功能框架 | ~7,600行 |

## 当前阶段
**Stage 2 — 核心代码路径分析** 🔄 进行中（第 1/4 个主题已完成）

## 已完成产出
- `notes/project-map-{gsdriver,nindriver,vtdriver}.md` — 三个项目地图 (Stage 0)
- `stages/stage-{0..5}-*.md` — 全部阶段定义文件 (Stage 0)
- `notes/nindriver-structure.md` — NinDriver 完整结构分析 (Stage 1)
- `notes/gsdriver-structure.md` — GsDriver 完整结构分析 (Stage 1)
- `notes/vtdriver-structure.md` — VT_Driver 完整结构分析 (Stage 1)
- `comparisons/modules/memory-read-write-paths.md` — **内存读写路径跨项目对比** (Stage 2) ← 最新

## Stage 2 内存读写对比核心结论

三个项目的内存读写代表了三个不同抽象层级：

| 项目 | 方案 | 一句话特征 |
|------|------|-----------|
| NinDriver | PTE 自映射物理内存 | 物理层操作，完全绕过 API Hook，但每页 4 次物理读+PTE改写，性能最差 |
| GsDriver | MmCopyVirtualMemory + MDL | 标准 API 路径，最稳定最高效，但调用栈完全暴露 |
| VT_Driver | KeStackAttach + CR3 切换双模式 | 直接模式简单，系统线程模式隐蔽但有 NMI 蓝屏风险 |

**关键发现**：
- VT_Driver 存在 EPROCESS 引用计数泄漏
- NinDriver 读写独立 SpinLock 有理论并发竞态
- VT_Driver `_TX` 后缀暗示系统线程模式针对腾讯系反作弊

## Stage 2 剩余主题
1. ~~内存读写路径对比~~ ✅
2. **VMX 初始化完整路径** — LoadHV 到 VMLAUNCH（建议下一个做驱动加载对比）
3. **EPT Hook 机制深度分析** — 代码页/数据页分离 + MTF 乒乓
4. **驱动隐蔽加载路径对比** — 自重映射 / 手动映射+系统线程 / LeiLei加载器

**建议下一主题**: 驱动隐蔽加载路径对比（三个项目都涉及，与内存分析互补）

## 关键规则
- 所有内容用中文
- 区分 facts / inferences / recommendations / open questions
- 产出写入文件，不仅在聊天中回答
- 不修改被研究项目代码
- 见 `learning-roadmap.md` 了解完整阶段规划
- 见 `current-status.md` 了解详细进度
