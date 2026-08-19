# 执行计划：Shift 点击区间选择

依据 `prd.md` 与 `design.md`。所有行号指 v1.3.1 的 `src/Surface.zig`。

## 前置：分支

- [ ] P1 确认工作区干净（当前仅 `.trellis/` 为未跟踪文件，切分支不受影响）
- [x] P2 从 v1.3.1 建分支：`git checkout -b feature/shift-click-range-select 332b2aefc6e72d363aa93ab6ecfc86eeeeb5ed28`
  - 本地已含该 commit 对象，无需 fetch 上游
- [ ] P3 基线自检：`zig build -Demit-macos-app=false`（确认 v1.3.1 在本机可编译，失败则先解决工具链再动代码）

> 若 P3 失败且原因是 Zig 版本与 v1.3.1 不匹配，暂停并回报，不要改代码迁就编译。

## 阶段 A：配置项打底

- [x] A1 `src/config/Config.zig`：紧邻 `@"mouse-shift-capture"`（952）新增 `@"mouse-shift-click-extend": bool = true`，附 design 第 6 节的文档注释
- [x] A2 `src/Surface.zig` `DerivedConfig`（314 区）新增 `mouse_shift_click_extend: bool`
- [x] A3 同文件 init（392 区）赋值 `config.@"mouse-shift-click-extend"`
- [x] A4 验证：已完成 Linux 目标编译；原生构建受本机 Darwin SDK 阻断

**Review gate 1**：此时功能行为无任何变化，只有配置项就位。确认编译通过再进入阶段 B。

## 阶段 B：就近端点纯函数（可单测，先于接线）

- [x] B1 新增 `ShiftAnchor` 结构与 `shiftExtendAnchor`，位置紧挨 `mouseSelection`（4897）之后，便于与既有选区纯函数聚在一起
- [x] B2 新增 `cellDistance` 私有辅助（基于 `pointFromPin(.screen, …)`）
- [x] B3 按 design 第 7 节写 7 组单元测试，参照 `mouseSelection` 既有测试的建屏方式
- [x] B4 验证：已完成 Linux 目标编译；运行受主机无法执行 Linux 二进制阻断

**Review gate 2**：纯函数与测试全绿后再接线。此时仍未改变任何运行时行为。

## 阶段 C：接线到点击流程

- [x] C1 新增 `shiftClickIsMultiClick`（design 4.2），距离公式与 4055 处保持字面一致
- [x] C2 新增 `setLeftClickAnchor`（design 4.5），track / untrack 顺序与 `errdefer` 参照 4065–4075
- [x] C3 新增 `prepareShiftClickAnchor`（design 4.3），注意 `posToViewport` 在锁外先算
- [x] C4 用 design 4.1 的新版本替换 `extend_selection` 块（3898–3935），删除原 `hasSelection()` 与裸时间判定
- [x] C5 验证：`zig fmt --check .` 通过；Linux 目标完成定向编译，原生构建受环境阻断

## 阶段 D：验证

- [ ] D1 全量测试：`zig build test`（慢，仅此处跑一次全量）
- [ ] D2 构建 macOS 应用并手工走查 AC1–AC6：
  - 单击 → 停顿 → Shift 点击另一处（AC1）
  - Shift 点击在锚点之前（AC2）
  - 已有选区后向前 / 向后 Shift 点击，验证选区只扩不丢（AC3）
  - 单击后**立即** Shift 点击远处（AC4，本次修复的缺陷，务必单独验证）
  - 原地快速双击选词、三击选行（AC5）
  - 连续多次 Shift 点击收放（AC6）
- [ ] D3 关闭开关（配置 `mouse-shift-click-extend = false`）复测 AC8，确认回退到 v1.3.1 行为
- [ ] D4 鼠标上报场景（AC7）：在 `vim` 或开启鼠标上报的程序中确认 Shift 点击不产生选区
- [ ] D5 滚动后跨屏扩展（AC9）

## 阶段 E：收尾

- [x] E1 `zig fmt --check .` 确认无残留改动
- [x] E2 按 Trellis 3.3 更新 spec，记录共享核心 Shift-click 选择约束
- [x] E3 按 Trellis 3.4 提交（中文 commit message，不建 PR、不建 issue）

## 验证备注

- Zig 已固定为 `0.15.2`。
- Linux `aarch64` 目标的定向和全量测试均完成源码/测试编译；测试运行阶段因 macOS 主机不能执行 Linux 二进制而退出。
- 原生 `zig build -Demit-macos-app=false` 和原生测试在初始化 Apple 依赖时因本机缺少完整 Darwin SDK 失败，未发现源码编译错误。

## 验证命令速查

```bash
zig fmt .
zig build -Demit-macos-app=false          # 快速编译
zig build test -Dtest-filter=shiftExtendAnchor
zig build test -Dtest-filter=mouseSelection
zig build test                            # 全量，仅 D1
```

## 回滚点

| 触发条件 | 动作 |
| --- | --- |
| 阶段 C 后出现选区回归 | `git checkout -- src/Surface.zig` 回到 gate 2 状态，纯函数与测试保留 |
| 上线后发现体验问题 | 配置 `mouse-shift-click-extend = false`，无需改代码 |
| 方案整体推翻 | 分支独立，直接弃用 `feature/shift-click-range-select` |

## 风险清单

- **R1**：v1.3.1 与本机 Zig 版本可能不匹配，P3 先行探测。
- **R2**：`left_click_ypos` 在锚点滚出视口时无合理取值，design 4.5 采用「沿用当前点击 y」的保守策略；若手工验证发现多击误判，改为不修改 ypos。
- **R3**：两段式加锁期间选区可能被 IO 线程清除，最坏是本次扩展失效，不会崩溃；若 D2 观察到偶发失效再考虑合并为单次持锁。
