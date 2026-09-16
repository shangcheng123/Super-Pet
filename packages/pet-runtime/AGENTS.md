# AGENTS.md — packages/pet-runtime

> 桌宠渲染与动画状态机。**本包不依赖 Electron，不依赖任何业务逻辑。**
> 上层规则见仓库根 [AGENTS.md](../../AGENTS.md)。

---

## 1. 职责边界

**做**：接收 canvas + 资源，渲染桌宠，驱动动画状态机。

**不做**：窗口管理、托盘、权限裁决、业务逻辑、网络请求、文件读写。

## 2. 硬性约束

1. **禁止 `import electron`**，禁止引用 `apps/desktop` 任何代码。本包必须能在纯浏览器环境（headless）里跑起来，这是「将来换外壳 / 做网页预览」的前提。
2. **禁止读文件**。资源由调用方以已解码的 `ImageBitmap` / `Texture` / 序列帧数据传入，本包不关心资源从哪来。
3. **禁止携带可执行代码的资源**。PetPack 只描述资源，不允许 eval / new Function / 动态 import 远程代码。
4. 所有对外类型来自 `packages/petpack-schema`，**不要在本包内重新定义 PetPack 结构**。

## 3. 动画状态机规则（AI 最容易写错的地方，必须人工复核）

- 优先级：`交互 > 情绪 > 移动 > 待机`。高优先级状态可抢占低优先级状态。
- 被抢占的状态要**保留恢复点**：如「被点击」播完后回到「走路」而不是「待机」。
- `loop: false` 的动画播完必须自动回落到 `reactions` 里声明的默认状态；缺失时回退到 `idle`。
- `reactions` 是集成方与桌宠之间的**共同语言**：调用方只说「thinking / success / error」，由 PetPack 决定播哪个动画；**PetPack 未声明该 reaction 时必须静默回退，不得抛错**。

## 4. 性能要求

- 空闲降帧：无交互且窗口不可见时降到 1–5 fps。
- `prefers-reduced-motion` 为真时保持首帧静态（无障碍要求）。
- 常驻内存与 CPU 占用可控，禁止在 render loop 里分配新对象。

## 5. 测试

- **headless 渲染测试**：不启动 Electron，用假 PetPack fixture 驱动状态机，断言状态迁移序列。
- 每个状态机规则（第 3 节）都要有对应测试用例。
- fixture 放 `fixtures/`，与 `petpack-schema` 共用同一份假资源包。

## 6. 参考项目

| 仓库 | 看什么 | License |
|---|---|---|
| `PPet/src` | Electron 透明窗口、跨平台打包、配置外置 | MIT |
| `pixi-live2d-display` | PixiJS + Live2D 渲染层、动作资产标准化 | MIT |
| `Ark-Pets` | 动画行为树 / 状态切换的优先级抢占 | **GPL-3.0，仅看思路，禁止复制代码** |
