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
**Stage 1 — 单项目结构理解** ✅ 全部完成 → 准备进入 Stage 2

## 已完成产出
- `notes/project-map-{gsdriver,nindriver,vtdriver}.md` — 三个项目地图 (Stage 0)
- `stages/stage-{0..5}-*.md` — 全部阶段定义文件 (Stage 0)
- `notes/nindriver-structure.md` — NinDriver 完整结构分析 (Stage 1)
- `notes/gsdriver-structure.md` — GsDriver 完整结构分析 (Stage 1)
- `notes/vtdriver-structure.md` — **VT_Driver 完整结构分析** (Stage 1) ← 最新

## 三个项目核心结论

### VT_Driver (最新)
- Type-2 Hypervisor：每 CPU VMXON → VMLAUNCH，OS 降级为 Guest
- EPT 透明钩取：代码页/数据页分离，MTF 单步乒乓，读到原始代码/执行修改代码
- 四层 Hook: EPT / Inline / SSDT / SSSDT
- 通信：NtDeviceIoControlFile 钩取，11 个 IOCTL (0x9800~0x9828)
- VM-exit: 65 个 handler，~20 个有逻辑，6 个 VMCALL 超调用
- 功能: 内存读写/进程管理/PatchGuard绕过/文件隐藏/窗口隐藏/网络拦截/反调试

### GsDriver
- 双层架构：外壳手动映射核心驱动到 NonPagedPoolExecute，启动为系统线程，外壳自删文件后退出
- 桥接：DynamicData 指针通过 GSDrv.bin 文件传递，核心读完即删
- 通信：CmRegisterCallback 注册表回调，22+ 命令码，无设备对象
- 回调反检测：在系统驱动 (null.sys/beep.sys 等) CC 填充中写入跳板代码
- 注入：三种模式 (线程/Hook ZwContinue/Steam) × 三级隐藏 (VAD/PTE/MDL+PFN)
- 其他：句柄提权、进程保护、键鼠模拟、反 BattlEye IAT hook、硬件 ID 伪造、WSK 网络

### NinDriver
- 双阶段初始化：DriverEntry(自毁) → MapEntry(持久，重映射地址空间)
- 内存访问：PTE 自映射三层架构（物理读写 → 页表遍历 → 跨页循环）
- CR3 获取：PFN 数据库扫描（利用页表自引用特征，无需偏移表）
- 偏移解析：从内核函数机器码提取 EPROCESS 字段偏移
- 隐蔽：文件自删除 + PE 头清零 + 独立物理页 + 匿名驱动对象

## 下一步: Stage 2 — 核心代码路径分析

**建议主题划分:**
1. 内存读写路径对比 — PTE自映射 / MmCopyVirtualMemory+MDL / CR3切换
2. VMX 初始化完整路径 — LoadHV 到 VMLAUNCH
3. EPT Hook 机制深度分析 — 代码页/数据页分离 + MTF 乒乓
4. 驱动隐蔽加载路径对比 — 自重映射 / 手动映射+系统线程 / LeiLei加载器

**建议先做:** 内存读写路径对比（三个项目都涉及，产出最有对比价值）

## 关键规则
- 所有内容用中文
- 区分 facts / inferences / recommendations / open questions
- 产出写入文件，不仅在聊天中回答
- 不修改被研究项目代码
- 见 `learning-roadmap.md` 了解完整阶段规划
- 见 `current-status.md` 了解详细进度
