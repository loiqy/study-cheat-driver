# Stage 1 进度日志

## 2026-03-28 — NinDriver 结构分析完成
- 产出: `notes/nindriver-structure.md`
- 关键发现: 双阶段初始化、PTE 自映射三层架构、PFN 数据库扫描、从机器码提取偏移
- 耗时: 单次会话

## 2026-03-28 — GsDriver 结构分析完成
- 产出: `notes/gsdriver-structure.md`
- 关键发现:
  - 双层驱动架构：外壳负责手动映射核心，核心以系统线程运行
  - 桥接机制：DynamicData 指针通过 GSDrv.bin 文件传递
  - 注册表回调通信：CmRegisterCallback + 22+ 命令码分发
  - 系统驱动跳板：在 null.sys/beep.sys 等 CC 填充中写入跳转代码
  - 三种注入模式 × 三级内存隐藏
  - 反 BattlEye IAT hook、硬件 ID 伪造、WSK 网络、键鼠模拟
- 与 NinDriver 对比:
  - 通信: 注册表回调 vs IOCTL
  - 内存读写: MmCopyVirtualMemory + MDL vs PTE 自映射物理内存
  - 偏移: 硬编码查表 vs 机器码提取
  - 功能: 完整框架 vs 纯内存读写
- 耗时: 单次会话
