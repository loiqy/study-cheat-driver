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
**Stage 0 — 全局地图** ✅ 核心产出已完成，待推进 Stage 1

## 已完成产出
- `notes/project-map-gsdriver.md` — GsDriver 完整项目地图
- `notes/project-map-nindriver.md` — NinDriver 完整项目地图
- `notes/project-map-vtdriver.md` — VT_Driver 完整项目地图
- `stages/stage-{0..5}-*.md` — 全部阶段定义文件

## Stage 0 关键发现
- 三项目共性: 内核内存读写、隐蔽策略、动态获取内核函数
- 通信方式各异: 注册表跳板 / 标准 IOCTL / IOCTL+hook
- 复杂度梯度: NinDriver < GsDriver < VT_Driver
- NinDriver 最适合作为 Stage 1 的起点

## 尚未开始
- Stage 1: 单项目深入结构理解
- Stage 2: 核心代码路径分析
- Stage 3+: 跨项目比较及后续

## 下一步
1. 确认 Stage 0 完成
2. 进入 Stage 1，建议顺序: NinDriver → GsDriver → VT_Driver
3. 为每个项目生成 `notes/{project}-structure.md`

## 关键规则
- 所有内容用中文
- 区分 facts / inferences / recommendations / open questions
- 产出写入文件，不仅在聊天中回答
- 不修改被研究项目代码
- 见 `learning-roadmap.md` 了解完整阶段规划
- 见 `current-status.md` 了解详细进度
