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
**Stage 1 — 单项目结构理解** 🔄 NinDriver ✅ | GsDriver 待做 | VT_Driver 待做

## 已完成产出
- `notes/project-map-{gsdriver,nindriver,vtdriver}.md` — 三个项目地图 (Stage 0)
- `stages/stage-{0..5}-*.md` — 全部阶段定义文件 (Stage 0)
- `notes/nindriver-structure.md` — **NinDriver 完整结构分析** (Stage 1) ← 最新

## NinDriver 结构分析核心结论
- 双阶段初始化：DriverEntry(自毁) → MapEntry(持久，重映射地址空间)
- 内存访问：PTE 自映射三层架构（物理读写 → 页表遍历 → 跨页循环）
- CR3 获取：PFN 数据库扫描（利用页表自引用特征，无需偏移表）
- 偏移解析：从内核函数机器码提取 EPROCESS 字段偏移
- 隐蔽：文件自删除 + PE 头清零 + 独立物理页 + 匿名驱动对象

## 尚未开始
- Stage 1: GsDriver 结构分析、VT_Driver 结构分析
- Stage 2: 核心代码路径分析
- Stage 3+: 跨项目比较及后续

## 下一步
1. GsDriver 结构分析 → `notes/gsdriver-structure.md`
2. VT_Driver 结构分析 → `notes/vtdriver-structure.md`
3. 完成 Stage 1 后进入 Stage 2

## 关键规则
- 所有内容用中文
- 区分 facts / inferences / recommendations / open questions
- 产出写入文件，不仅在聊天中回答
- 不修改被研究项目代码
- 见 `learning-roadmap.md` 了解完整阶段规划
- 见 `current-status.md` 了解详细进度
