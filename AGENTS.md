# AGENTS.md — SuperPet

> 本文件是 AI 编码助手的入口说明书。**所有自动化产出（AI 生成的代码、测试、配置、文档）都必须遵守本文件。**
> 详细设计见 [docs/开发文档.md](docs/开发文档.md) ｜ 功能范围见 [docs/功能文档.md](docs/功能文档.md) ｜ 借鉴思路见 [docs/借鉴文档.md](docs/借鉴文档.md)

---

## 1. 项目是什么

**SuperPet = 桌宠工坊 + 桌面 AI 助手。** 两条产品主线：

| 主线 | 形态 | 收费 | 核心 |
|---|---|---|---|
| A. 桌宠工坊 | 上传照片/选形象 → 选画风/动作/功能/语音 → 一键生成 | 按量（点数） | 个性化、低门槛、可分享 |
| B. 语音操控电脑 | 自然语言操作电脑，带风险分级与二次确认 | 订阅制 | 从观赏宠物升级为桌面助手 |

主线 A 是获客入口，主线 B 是 LTV 核心。

---

## 2. 两条硬约束（不可违反）

1. **可扩展优先**：能力包、技能、画风、动作、模型供应商全部是**声明式配置**，新增能力不改主干代码。
2. **不写死**：价格、能力清单、技能白名单、模型供应商一律配置化。**客户端不得包含任何价格常量。**

---

## 3. 目录结构

```
superpet/
  apps/desktop/            # Electron 主进程 + 渲染进程
    src/main/              # 主进程：状态机、窗口、托盘、权限裁决、插件沙箱
    src/renderer/          # React + PixiJS
    src/preload/           # contextBridge
  packages/
    pet-runtime/           # 桌宠渲染 + 动画状态机（禁止 import electron）
    petpack-schema/        # PetPack manifest 类型与校验（zod + JSON Schema）
    skill-protocol/        # 技能描述符 + MCP 适配 + 风险注解
    ipc-client/            # 本地 IPC 客户端（供外部集成复用）
    pricing-sdk/           # 计价契约类型（仅预估，不结算）
    workshop-ui/           # 工坊流程组件
  crates/
    executor/              # Rust 技能执行器（阶段二/三）
    execpolicy/            # 执行策略引擎（规则 DSL）
  services/
    api/                   # Go：账户/订单/订阅/计价
    ai-gateway/            # Go：LLM/ASR/TTS/图像供应商适配
  plugins/{official,community,dev}/
  policies/                # 技能策略规则（*.rules）
  docs/
  references/              # 参考项目（已 gitignore，仅本地）
```

---

## 4. 分层依赖方向（只能自上而下，禁止反向）

| 层 | 职责 | 禁止 |
|---|---|---|
| UI 层 | 渲染、交互、展示 | 直接读写文件、直接调系统 API |
| 宿主层（主进程） | 状态、窗口、权限裁决、插件沙箱 | 直接执行系统命令 |
| 执行层（Rust） | 技能执行、沙箱、审计 | 承载业务逻辑 |
| 服务层（Go） | 账户、计价、订阅、AI 代理 | 依赖客户端实现细节 |

**核心原则：宿主拥有所有副作用，插件只描述意图。**

---

## 5. 契约是唯一真相

契约全部**版本化 + JSON Schema + 代码生成**。**禁止手写契约类型，禁止让不同模块各自定义同名类型。**

| 契约 | 位置 | 消费方 |
|---|---|---|
| PetPack manifest | `packages/petpack-schema` | 工坊产出、运行时消费、服务端存储 |
| 计价（`/v1/quote`、`/v1/orders`） | `packages/pricing-sdk` + `services/api` | 工坊 UI 预估、服务端结算 |
| 本地 IPC | `packages/ipc-client` | 主进程、外部集成（CLI/编辑器插件） |
| 技能描述符 | `packages/skill-protocol` | 执行器、AI 网关、插件（阶段二起） |

**改契约的流程**：改 Schema → 重新生成 → 三方 review → conformance check 通过。不允许「顺手把接口改一下」。

---

## 6. 命令

> 骨架搭好后生效；若命令与实际不符，以仓库根 `package.json` / `Makefile` 为准，并回来修正本文件。

```bash
pnpm install            # 安装依赖（pnpm workspace）
pnpm -r typecheck       # 全量类型检查
pnpm -r test            # 全量测试
pnpm -r lint            # 全量 lint
pnpm --filter desktop dev   # 启动桌面客户端

cd services/api && go test ./...   # 服务端测试
cd crates/executor && cargo test   # 执行器测试（阶段三）
```

---

## 7. 反面清单（明确不做）

- ❌ 客户端内置 Python 运行时
- ❌ 客户端硬编码价格
- ❌ 桌宠包（PetPack）携带可执行代码
- ❌ 插件直接操作 UI 或系统（必须经宿主）
- ❌ 第一步就上 Rust + Go + MCP 全套
- ❌ 复制 GPL-3.0 项目的代码进闭源产品（`references/` 里 Ark-Pets、DyberPet、live2d-widget、ComfyUI 均为 GPL-3.0，**仅可参考思路**）
- ❌ 金额使用浮点数（必须 decimal）
- ❌ 生成失败重复扣费

---

## 8. 完成的定义（DoD）

任何改动合入前必须满足：

1. **带测试**：契约改动带 conformance check；计价改动带报价用例表/快照；技能带 fixture 与 dry-run 快照
2. **跨线互审**：桌宠本体线 / 工坊生成线 / 平台交易线，两两互审（见 [docs/团队分工.md](docs/团队分工.md)）
3. **lint / typecheck / test 全绿**
4. **不违反第 2、4、7 节任何一条**

---

## 9. 必须人工逐行审的地方（不得只信 AI）

- `services/api` 的金额计算：decimal 类型、并发扣点、幂等键、计价快照
- `packages/pet-runtime` 的动画状态机抢占优先级
- `packages/workshop-ui` 的风格 × 形象 × 动作兼容矩阵（AI 极易漏掉禁用逻辑）
- 任何涉及用户文件系统或系统设置的调用

---

## 10. 参考项目怎么用

`references/` 是浅克隆的参考仓库，**只读、不提交、不是依赖**。按 [references/参考项目索引.md](references/参考项目索引.md) 的建议顺序阅读：

1. `openpets/DESIGN.md` + `plugins/` — 插件化桌宠平台的整体架构
2. `leon/core` — 技能声明与路由协议
3. `UFO` + `OmniParser` — 桌面操控与屏幕理解
4. `open-interpreter` — 执行前的确认与安全边界
5. `PPet/src` + `pixi-live2d-display` — 桌宠渲染与透明窗口
6. `ComfyUI` — 生成流水线

**License 红线**：MIT / Apache-2.0 可参考并保留声明；CC-BY-4.0（OmniParser）商用需署名；**GPL-3.0 只能看思路，不能复制代码**。
