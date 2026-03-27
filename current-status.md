# 当前状态

> 最后更新：2026-03-28

## 当前阶段
**Stage 0 — 全局地图** （已完成核心产出，待确认推进 Stage 1）

## 已完成
- [x] workspace 目录结构初始化
- [x] 阶段框架定义 (stage 0-5)
- [x] 核心状态文件建立
- [x] **GsDriver 项目地图** → `notes/project-map-gsdriver.md`
- [x] **NinDriver 项目地图** → `notes/project-map-nindriver.md`
- [x] **VT_Driver 项目地图** → `notes/project-map-vtdriver.md`

## Stage 0 关键发现摘要

### 三个项目的定位和规模

| 项目 | 架构 | 源码行数 | 通信方式 | 复杂度 |
|------|------|----------|----------|--------|
| GsDriver | 双层: 外壳(loader)+核心 | ~7,800 | 注册表跳板 (非常规) | 中 |
| NinDriver | 单驱动, 自重映射 | ~880 | 标准 IOCTL | 低 |
| VT_Driver | VT-x 虚拟化 + 功能框架 | ~7,600+ | IOCTL + NtDeviceIoControlFile hook | 高 |

### 共性特征
- 三者都涉及**内核内存读写**
- 三者都有**隐蔽/反检测**策略
- 三者都使用**动态获取内核函数**的方式 (避免硬编码)
- GsDriver 和 VT_Driver 都使用 VMProtect 保护

### 差异化特征
- **GsDriver**: 注册表通信 (独特)、硬件 ID 伪造、WSK 网络
- **NinDriver**: 自重映射技术、物理内存 PTE 自映射、PFN 扫描获取 CR3
- **VT_Driver**: Intel VT-x 虚拟化、EPT 钩取、PatchGuard 绕过、SSSDT hook

## 待解决问题 (Open Questions)
- GsDriver 外壳→核心的加载/映射具体机制？
- NinDriver 自重映射后驱动对象的生命周期管理？
- VT_Driver 65 个 VM-exit handler 中哪些有实际逻辑？
- 三个项目的内存读写方式各有什么优劣？
- 三者的隐蔽策略有何可比较性？

## 下一步行动
1. **确认 Stage 0 完成** — 三个项目地图已生成
2. **进入 Stage 1** — 逐项目深入结构理解
3. 建议 Stage 1 顺序: NinDriver (最小) → GsDriver (中等) → VT_Driver (最复杂)

## 快速跳转
- 阶段规划 → `learning-roadmap.md`
- 新会话接续 → `resume/resume-brief.md`
- Stage 0 详情 → `stages/stage-0-global-map.md`
- GsDriver 地图 → `notes/project-map-gsdriver.md`
- NinDriver 地图 → `notes/project-map-nindriver.md`
- VT_Driver 地图 → `notes/project-map-vtdriver.md`
