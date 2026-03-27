# 当前状态

> 最后更新：2026-03-28

## 当前阶段
**Stage 1 — 单项目结构理解** （进行中：NinDriver ✅ → GsDriver 待做 → VT_Driver 待做）

## Stage 1 进度
- [x] **NinDriver 结构分析** → `notes/nindriver-structure.md`
- [ ] GsDriver 结构分析 → `notes/gsdriver-structure.md`
- [ ] VT_Driver 结构分析 → `notes/vtdriver-structure.md`

## Stage 0 已完成 ✅
- [x] workspace 目录结构初始化
- [x] 阶段框架定义 (stage 0-5)
- [x] 核心状态文件建立
- [x] **GsDriver 项目地图** → `notes/project-map-gsdriver.md`
- [x] **NinDriver 项目地图** → `notes/project-map-nindriver.md`
- [x] **VT_Driver 项目地图** → `notes/project-map-vtdriver.md`

## NinDriver 结构分析关键发现 (Stage 1)

### 架构核心：双阶段初始化 + 自重映射
- DriverEntry 是一次性入口：删文件 → 解析偏移 → 分配独立页 → 复制镜像 → IoCreateDriver → 返回失败
- MapEntry 是持久入口：在重映射地址空间运行，创建伪装设备，注册 IOCTL dispatch
- PE 头被清零，原始镜像被系统卸载——最终结果是一个"无源"匿名驱动

### 内存访问：PTE 自映射三层架构
- 底层：修改 MapAddress 的 PTE 指向目标物理页 + invlpg（单页，自旋锁保护）
- 中层：四级页表遍历做虚拟→物理地址转换
- 顶层：跨页循环 + __try/__except 异常保护

### CR3 获取：PFN 数据库扫描
- 利用页表自引用特征定位进程 CR3 页
- 无需 EPROCESS 偏移表，但扫描全部物理内存，性能开销大

### 运行时偏移：从内核函数机器码提取
- 6 个 EPROCESS 字段偏移从 PsGetProcessXxx 函数前几条指令中提取
- PTE 基址和 PFN 基址从 MmGetVirtualForPhysical 提取
- 3 个未导出函数通过 ntoskrnl.exe 特征码扫描定位

## 待解决问题 (Open Questions)
- ~~NinDriver 自重映射后驱动对象的生命周期管理~~？→ 已解答：无卸载路径，持续到重启
- GsDriver 外壳→核心的加载/映射具体机制？
- VT_Driver 65 个 VM-exit handler 中哪些有实际逻辑？
- PTE 自映射的 Read/Write 独立自旋锁是否有并发竞态风险？
- PFN 结构 0x30 字节大小和 EPROCESS 解密公式的版本适用范围？
- 三个项目的内存读写方式各有什么优劣？

## 下一步行动
1. **GsDriver 结构分析** — Stage 1 Step 2
2. **VT_Driver 结构分析** — Stage 1 Step 3
3. 完成 Stage 1 后更新阶段状态，进入 Stage 2 核心路径分析

## 快速跳转
- 阶段规划 → `learning-roadmap.md`
- 新会话接续 → `resume/resume-brief.md`
- Stage 0 详情 → `stages/stage-0-global-map.md`
- Stage 1 详情 → `stages/stage-1-project-structure.md`
- NinDriver 结构 → `notes/nindriver-structure.md`
- GsDriver 地图 → `notes/project-map-gsdriver.md`
- NinDriver 地图 → `notes/project-map-nindriver.md`
- VT_Driver 地图 → `notes/project-map-vtdriver.md`
