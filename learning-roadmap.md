# 学习路线图

## 研究对象

| 项目 | 路径 | 初步定位 |
|------|------|----------|
| GsDriver | `../GsDriver-master/` | 双层驱动架构（外壳 + 核心），内核读写类 |
| NinDriver | `../NinDriver/` | 独立驱动方案，VS 解决方案 |
| VT_Driver | `../VT_Driver/` | Intel VT-x 虚拟化驱动，含 capstone 反汇编引擎 |

## 阶段规划

### Stage 0 — 全局地图
- **目标**：识别每个项目的文件布局、构建方式、入口点
- **产出**：`notes/project-map-{name}.md`
- **完成标准**：能画出每个项目的顶层文件树和模块边界

### Stage 1 — 单项目结构理解
- **目标**：逐项目读入口、理清调用链、标记关键数据结构
- **产出**：`notes/{project}-structure.md`
- **完成标准**：能用一段话说清每个项目"做了什么、怎么做的"

### Stage 2 — 核心路径和关键模块
- **目标**：深入各项目最核心的代码路径（如驱动通信、内存读写、VT-x 进出）
- **产出**：`notes/{project}-core-paths.md`
- **完成标准**：能精确指出关键函数及其职责

### Stage 3 — 跨项目比较
- **目标**：对比三个项目在相同关注点上的设计差异
- **产出**：`comparisons/cross-project-*.md`
- **完成标准**：至少完成 3 个维度的对比表

### Stage 4 — 抽象、复用和设计判断
- **目标**：提炼可复用模式、评估各方案优劣
- **产出**：`summaries/reuse-patterns.md`, `summaries/design-tradeoffs.md`
- **完成标准**：形成有依据的推荐意见

### Stage 5 — 教学讲义和复盘
- **目标**：写出能教别人的讲义、回顾学习过程
- **产出**：`summaries/teaching-notes.md`, `summaries/retrospective.md`
- **完成标准**：讲义可独立阅读

## 横切原则
- 每次产出区分 **事实 (facts)**、**推断 (inferences)**、**建议 (recommendations)**、**待解决问题 (open questions)**
- 重要发现立即写文件，不仅在聊天中回答
- 不修改被研究项目代码，除非明确要求
