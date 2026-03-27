# Resume Brief — 快速接续文件

> 供新 Claude 会话快速了解当前学习状态。最后更新：2026-03-28

## 仓库用途
这是一个**学习仓库**，不是应用项目。用于系统性研究 3 个 Windows 内核驱动项目。

## 三个研究对象
1. **GsDriver** (`../GsDriver-master/`) — 双层架构（驱动外壳 + 驱动核心）
2. **NinDriver** (`../NinDriver/`) — 独立驱动方案
3. **VT_Driver** (`../VT_Driver/`) — Intel VT-x 虚拟化驱动，含 capstone

## 当前阶段
**Stage 0 — 全局地图**（刚开始）

## 已完成
- workspace 目录结构初始化
- 阶段框架定义 (stage 0-5)
- 核心状态文件建立

## 尚未开始
- 任何项目的实际代码阅读
- 项目地图文件
- 跨项目比较

## 下一步
1. 对每个项目生成文件树 → `notes/project-map-{name}.md`
2. 识别入口点、构建方式、模块边界
3. 推进到 Stage 1

## 关键规则
- 所有内容用中文
- 区分 facts / inferences / recommendations / open questions
- 产出写入文件，不仅在聊天中回答
- 不修改被研究项目代码
- 见 `learning-roadmap.md` 了解完整阶段规划
- 见 `current-status.md` 了解最新进度
