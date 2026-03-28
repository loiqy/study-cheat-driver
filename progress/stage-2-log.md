# Stage 2 进度日志

> 核心代码路径分析

## 2026-03-28 — 内存读写路径对比 ✅

**产出**: `comparisons/modules/memory-read-write-paths.md`

**分析内容**:
- 三个项目完整调用链（从入口到实际读写）
- 关键数据结构与依赖关系
- 资源 Ownership 与生命周期分析
- 脆弱点与故障模式（含代码位置）
- 六维对比矩阵（复杂度/隐蔽性/稳定性/性能/只读绕过/版本兼容）
- 设计决策深层逻辑分析
- 事实 vs 推断清单
- 5 个新 Open Questions

**关键发现**:
1. VT_Driver `Memory::TargetProcess` 引用计数泄漏 — `PsLookupProcessByProcessId` 后无 `ObDereferenceObject`
2. NinDriver 的 Read/Write 各自独立 SpinLock，理论上存在并发修改 MapAddress PTE 的竞态
3. GsDriver Type 2 MDL 路径中 `MmProbeAndLockPages` 无 `__try` 保护
4. VT_Driver 系统线程模式关闭中断 + 清除 WP 位 + CR3 切换，NMI 窗口可致蓝屏
5. NinDriver 物理地址掩码仅支持 36 位（64GB），现代系统可能截断

**代码覆盖**:
- NinDriver: Memory.cpp (全部 255 行), Driver.cpp (全部), Export.cpp (全部)
- GsDriver: 通讯回调.cpp ('0006' 命令 ~140 行), 导出函数.cpp (ZwCopyVirtualMemory, RtlSuperCopyMemory)
- VT_Driver: Memory.cpp (全部 238 行), Main.cpp (ControlCenter), Utils.cpp (WriteProtectOff/On)

---

## 待完成主题

- [ ] VMX 初始化完整路径
- [ ] EPT Hook 机制深度分析
- [ ] 驱动隐蔽加载路径对比
