# 当前状态

> 最后更新：2026-03-28

## 当前阶段
**Stage 2 — 核心代码路径分析** 🔄 进行中

## Stage 2 进度
- [x] **内存读写路径对比** → `comparisons/modules/memory-read-write-paths.md` ← 最新
- [ ] VMX 初始化完整路径 — LoadHV 到 VMLAUNCH
- [ ] EPT Hook 机制深度分析 — 代码页/数据页分离 + MTF 乒乓
- [ ] 驱动隐蔽加载路径对比 — 自重映射 / 手动映射+系统线程 / LeiLei加载器

## Stage 2 内存读写路径关键发现

### 三种方案一句话差异
- **NinDriver**：物理地址层 PTE 窗口映射——完全绕过 API Hook，但每页需 4 次物理内存读取 + PTE 修改
- **GsDriver**：MmCopyVirtualMemory + MDL 重映射——最稳定最高效，但 API 调用栈完全暴露
- **VT_Driver**：双模式 (KeStackAttach + CR3 切换系统线程)——兼顾便利和隐蔽，但 CR3 切换有 NMI 蓝屏风险

### 关键发现
- VT_Driver `Memory::TargetProcess` 存在 EPROCESS 引用计数泄漏（PsLookupProcessByProcessId 后无 ObDereferenceObject）
- NinDriver 的 Read/Write 使用独立 SpinLock，存在理论并发竞态
- GsDriver Type 2 MDL 强写路径中 MmProbeAndLockPages 缺少 __try 保护
- VT_Driver 系统线程模式的 `_TX` 后缀暗示针对腾讯系反作弊
- NinDriver 的物理地址掩码仅支持 36 位（64GB），现代系统可能不够

## Stage 1 已完成 ✅
- [x] **NinDriver 结构分析** → `notes/nindriver-structure.md`
- [x] **GsDriver 结构分析** → `notes/gsdriver-structure.md`
- [x] **VT_Driver 结构分析** → `notes/vtdriver-structure.md`

## Stage 0 已完成 ✅
- [x] workspace 目录结构初始化
- [x] 阶段框架定义 (stage 0-5)
- [x] 核心状态文件建立
- [x] **GsDriver 项目地图** → `notes/project-map-gsdriver.md`
- [x] **NinDriver 项目地图** → `notes/project-map-nindriver.md`
- [x] **VT_Driver 项目地图** → `notes/project-map-vtdriver.md`

## 待解决问题 (Open Questions)

### 来自 Stage 2 内存读写分析
- NinDriver EPROCESS 解密公式 `(value | 0xF000000000000000) >> 13 | 0xFFFF000000000000` 的数学推导？
- VT_Driver TargetProcess 引用泄漏是有意设计还是 bug？
- GsDriver MmProbeAndLockPages 目标页被换出时是否阻塞？
- VT_Driver 系统线程模式能否被 Hypervisor 级反作弊通过 MOV CR3 拦截检测？

### 来自 Stage 1
- GsDriver: DrvData[1000] 占位符如何被实际核心驱动字节码替换？
- GsDriver: VMProtect 保护范围？
- VT_Driver: EPT 预分配 512 页是否足够？
- VT_Driver: capstone vs LDasm vs ShellCode LDE 三个反汇编器的分工？

## 下一步行动
1. **继续 Stage 2 — 建议下一主题：驱动隐蔽加载路径对比**
   - 三种加载方式都有完整代码，且与内存读写分析互补
   - 或选 VMX 初始化完整路径（VT_Driver 独有复杂度最高）

## 快速跳转
- 阶段规划 → `learning-roadmap.md`
- 新会话接续 → `resume/resume-brief.md`
- Stage 0 详情 → `stages/stage-0-global-map.md`
- Stage 1 详情 → `stages/stage-1-project-structure.md`
- **Stage 2 内存读写对比** → `comparisons/modules/memory-read-write-paths.md`
- NinDriver 结构 → `notes/nindriver-structure.md`
- GsDriver 结构 → `notes/gsdriver-structure.md`
- VT_Driver 结构 → `notes/vtdriver-structure.md`
- 进度日志 → `progress/stage-2-log.md`
