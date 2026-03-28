# 当前状态

> 最后更新：2026-03-28

## 当前阶段
**Stage 1 — 单项目结构理解** ✅ 已完成 → 准备进入 Stage 2

## Stage 1 进度
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

## VT_Driver 结构分析关键发现 (Stage 1)

### 架构核心：Type-2 Hypervisor + EPT 透明钩取
- 初始化：LeiLei 多阶段加载 → LoadHV() → 每个 CPU VMXON/VMLAUNCH
- 通信：NtDeviceIoControlFile 钩取，11 个 IOCTL (0x9800~0x9828)
- EPT Hook：代码页/数据页分离，MTF 单步乒乓切换，读取看原始/执行走修改
- VM-exit：65 个 handler，~20 个有逻辑，核心为 CPUID/VMCALL/EPT_VIOLATION/MTF

### 功能矩阵
- 四层 Hook: EPT / Inline / SSDT / SSSDT
- 内存读写: KeStackAttach 直接模式 + CR3 切换系统线程模式
- 反调试: ObCallback/NtRead/NtWrite/NtQueryThread hook
- 隐藏: MiniFilter 文件隐藏 + SSSDT 窗口隐藏 + EPT 内存隐藏
- PatchGuard 绕过: Win10/WinX 两个 PoC
- 网络: AFD 拦截 HTTP 包特征替换

## GsDriver 结构分析关键发现 (Stage 1)

### 架构核心：双层驱动 + 注册表通信
- 驱动外壳：DriverEntry 初始化环境 → 手动映射核心驱动 → 自删除 + 返回 0xE0000000
- 驱动核心：以系统线程启动，通过 GSDrv.bin 文件接收 DynamicData 指针
- 通信机制：CmRegisterCallback 注册表回调，22+ 命令码分发

### 功能矩阵
- 内存读写：MmCopyVirtualMemory (读/写) + MDL 强写 (绕只读)
- DLL 注入：三种模式 (线程/Hook ZwContinue/Steam 劫持) × 三级隐藏 (VAD/PTE/MDL+PFN)
- 进程操作：保护/强杀/句柄提权/内存隐藏
- 外设：键鼠模拟 (直接调用 class driver 回调)
- 对抗：反 BattlEye IAT hook + 硬件 ID 伪造 (NIC/磁盘/SMBIOS/GPU)
- 网络：WSK 内核 HTTP POST

### 反检测手段
- 系统驱动跳板：在 null.sys/beep.sys 等 .text 段 CC 填充中写入跳转代码，回调注册地址指向系统模块
- 无设备对象：不创建 DriverObject，不使用 IOCTL
- IAT hash 匹配：核心驱动导入表用 hash 解析

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
- GsDriver: DrvData[1000] 占位符如何被实际核心驱动字节码替换？
- GsDriver: VMProtect 保护范围？
- GsDriver: UserVerify 的 RegisterVerify 函数未实现？
- VT_Driver: EPT 预分配 512 页是否足够？高 IRQL 下用尽后触发 panic 的实际场景？
- VT_Driver: Bypass 模块硬编码偏移仅适用 Win7，是否有动态版本？
- VT_Driver: LSTAR Hook 的实际调用场景？
- VT_Driver: capstone vs LDasm vs ShellCode LDE 三个反汇编器的分工？
- PTE 自映射的 Read/Write 独立自旋锁是否有并发竞态风险？
- PFN 结构 0x30 字节大小和 EPROCESS 解密公式的版本适用范围？
- 三个项目的内存读写方式各有什么优劣？

## 下一步行动
1. **进入 Stage 2 — 核心代码路径分析**
2. 建议主题划分见下方

## Stage 2 建议主题划分
1. **内存读写路径对比** — 三种方案 (PTE自映射 / MmCopyVirtualMemory+MDL / CR3切换) 的完整调用链和检测面
2. **VMX 初始化完整路径** — VT_Driver 独有，从 LoadHV 到 VMLAUNCH 的每一步
3. **EPT Hook 机制深度分析** — 代码页/数据页分离 + MTF 乒乓的完整流程
4. **驱动隐蔽加载路径对比** — 三种加载方式 (自重映射 / 手动映射+系统线程 / LeiLei加载器)

## 快速跳转
- 阶段规划 → `learning-roadmap.md`
- 新会话接续 → `resume/resume-brief.md`
- Stage 0 详情 → `stages/stage-0-global-map.md`
- Stage 1 详情 → `stages/stage-1-project-structure.md`
- NinDriver 结构 → `notes/nindriver-structure.md`
- GsDriver 结构 → `notes/gsdriver-structure.md`
- VT_Driver 结构 → `notes/vtdriver-structure.md`
- GsDriver 地图 → `notes/project-map-gsdriver.md`
- NinDriver 地图 → `notes/project-map-nindriver.md`
- VT_Driver 地图 → `notes/project-map-vtdriver.md`
