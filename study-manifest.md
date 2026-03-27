# 学习仓库清单 (Study Manifest)

## 主题
系统性研究 3 个 Windows 内核驱动项目的架构、实现和设计取舍。

## 研究对象
1. **GsDriver** — `../GsDriver-master/` — 双层驱动架构（驱动外壳 + 驱动核心）
2. **NinDriver** — `../NinDriver/` — 独立驱动方案
3. **VT_Driver** — `../VT_Driver/` — Intel VT-x 虚拟化驱动，含 capstone 反汇编库

## 学习目标
- 独立理解每个项目的架构和实现
- 对比设计选择和技术取舍
- 积累可复用的笔记和分析
- 支持新 Claude 会话快速接续

## 核心文件索引
| 文件 | 用途 |
|------|------|
| `study-manifest.md` | 本文件，仓库总纲 |
| `current-status.md` | 当前进度快照 |
| `learning-roadmap.md` | 阶段规划和完成标准 |
| `resume/resume-brief.md` | 新会话快速接续文件 |

## 目录结构
| 目录 | 用途 |
|------|------|
| `notes/` | 单项目学习笔记 |
| `comparisons/` | 跨项目比较分析 |
| `stages/` | 阶段定义和完成标准 |
| `progress/` | 进度日志 |
| `plans/` | 分析计划 |
| `summaries/` | 总结和教学产出 |
| `resume/` | 会话接续文件 |

## 横切规则
1. 所有内容用**中文**撰写
2. 区分 **事实 (facts)** / **推断 (inferences)** / **建议 (recommendations)** / **待解决问题 (open questions)**
3. 重要发现写入文件，不仅在聊天中回答
4. 不修改被研究项目代码，除非明确要求
