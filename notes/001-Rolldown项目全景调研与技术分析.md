# 001 - Rolldown 项目全景调研与技术分析

> **编写日期：** 2026-03-20
> **文档版本：** v1.0
> **调研范围：** 项目定位、发展历程、技术架构、性能数据、生态关系、实际 bug 案例分析

---

## 一、项目概述

### 1.1 一句话定义

Rolldown 是一个用 **Rust** 编写的高性能 JavaScript/TypeScript 打包器（bundler），目标是成为 Vite 的统一底层打包引擎，提供与 Rollup 兼容的插件 API，同时拥有接近甚至超越 esbuild 的速度。

### 1.2 核心定位

| 维度 | 描述 |
|------|------|
| **语言** | Rust（核心）+ NAPI-RS（Node.js binding）|
| **兼容性** | Rollup 插件 API 兼容，已通过 900+ Rollup 测试和 670+ esbuild 测试 |
| **目标用户** | Vite 生态的所有用户（Vue、React、Svelte、Nuxt、Astro 等框架的使用者）|
| **替代目标** | 在 Vite 中同时替代 esbuild（开发时）和 Rollup（生产构建）|
| **商业支撑** | VoidZero Inc. 持有版权并控制项目方向 |

### 1.3 解决的核心问题

Vite 长期存在的架构痛点——**开发与生产的双打包器不一致**：

```
Vite 旧架构：
  开发时 → esbuild (Go, 快但插件不兼容 Rollup)
  生产时 → Rollup  (JS, 插件生态好但慢)
  问题：行为不一致、两套配置、两套 bug

Vite 新架构（Vite 8+）：
  开发 + 生产 → Rolldown (Rust, 既快又兼容)
  解决：统一行为、单一配置、单一打包器
```

---

## 二、发展历程

### 2.1 完整时间线

| 时间 | 里程碑 | 详情 |
|------|--------|------|
| **2023-09** | ViteConf 2023 宣布 | Evan You 首次公开 Rolldown 计划：用 Rust 重写 Rollup，兼容其 API，用于 Vite |
| **2023-11** | 项目开源 | GitHub 仓库 [rolldown/rolldown](https://github.com/rolldown/rolldown) 公开 |
| **2024 全年** | 密集开发期 | 实现 code splitting、tree shaking、插件系统、Rollup 兼容层等核心功能 |
| **2024-09** | ViteConf 2024 | 展示重大进展，宣布将集成进 Vite；同期发布 Rolldown beta |
| **2024-10-01** | VoidZero 成立 | Evan You 创立 VoidZero Inc.（Palo Alto），获 Accel 领投 **$4.6M** 种子轮融资 |
| **2025 上半年** | 生产验证期 | 发布 rolldown-vite 技术预览，Excalidraw、GitLab 等早期采用者开始测试 |
| **2025-06** | 早期采用者报告 | 多个项目报告 3–16× 构建加速，InfoQ 等媒体广泛报道 |
| **2025-10** | Series A 融资 | VoidZero 获 Accel、Peak XV Partners 领投 **$12.5M** A 轮融资 |
| **2025-11** | 特性完善期 | 持久缓存、粒度化 chunking 控制、WASM 构建优化 |
| **2025-12** | Vite 8 Beta 发布 | Rolldown 正式集成进 Vite 8 Beta，替代 esbuild + Rollup |
| **2026-01-21** | **Rolldown 1.0 RC** | 宣布 API 稳定，无破坏性变更计划；累计 3,400+ commits |
| **2026-03** | Vite 8 正式发布 | Rolldown 作为默认打包器随 Vite 8.0 正式发布 |

### 2.2 RC 阶段统计（自 beta.1 起）

| 指标 | 数量 |
|------|------|
| 总 commit 数 | 3,400+ |
| 新增特性 | 749 |
| Bug 修复 | 682 |
| 性能优化 | 109 |
| 文档更新 | 166 |

---

## 三、技术架构

### 3.1 项目结构

```
rolldown/
├── crates/                              # Rust 核心代码
│   ├── rolldown/                        # 打包器主逻辑
│   │   ├── src/
│   │   │   ├── stages/
│   │   │   │   ├── link_stage/          # 模块链接、符号解析、tree shaking
│   │   │   │   │   ├── tree_shaking/
│   │   │   │   │   └── patch_module_dependencies.rs
│   │   │   │   └── generate_stage/      # chunk 生成与优化
│   │   │   │       ├── code_splitting.rs        # 代码分割核心算法
│   │   │   │       ├── chunk_optimizer.rs       # chunk 合并与 facade 消除
│   │   │   │       ├── compute_cross_chunk_links.rs  # 跨 chunk 导入/导出
│   │   │   │       └── manual_code_splitting.rs # 手动分包（advancedChunks）
│   │   │   ├── module_finalizers/       # AST → 最终输出代码
│   │   │   ├── chunk_graph.rs           # chunk 依赖图数据结构
│   │   │   └── hmr/                     # Hot Module Replacement
│   │   └── tests/                       # 集成测试（1700+ fixture 测试）
│   ├── rolldown_common/                 # 共享类型（RuntimeHelper、Module 等）
│   ├── rolldown_plugin/                 # 插件系统（Rollup API 兼容层）
│   ├── rolldown_testing/                # 测试基础设施
│   ├── rolldown_plugin_*               # 内置插件（HMR、DTS、asset 等）
│   └── rolldown_error/                 # 错误处理
├── packages/                            # Node.js 包
│   ├── rolldown/                        # npm 发布包（NAPI binding）
│   └── rollup-compat/                   # Rollup 兼容垫片
└── web/                                 # 文档站点
```

### 3.2 核心处理流水线

```
源码文件
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│  Scan Stage（扫描阶段）                                    │
│  · oxc_parser 解析 AST                                    │
│  · oxc_resolver 解析模块路径                                │
│  · 收集依赖关系、import/export 信息                          │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  Link Stage（链接阶段）                                    │
│  · 符号解析与合并（SymbolDB）                                │
│  · Tree Shaking（基于语句级依赖分析）                         │
│  · CJS/ESM 互操作处理（wrap kind 决策）                      │
│  · 确定 runtime helper 依赖                                │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  Generate Stage（生成阶段）                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Code Splitting（代码分割）                          │    │
│  │ · Bit-based 模块分配算法（每个入口一个 bit）          │    │
│  │ · 共享模块 → Common Chunk                          │    │
│  │ · 动态导入 → Dynamic Entry Chunk                   │    │
│  └──────────────────────┬──────────────────────────┘    │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Chunk Optimization（chunk 优化）                    │    │
│  │ · Facade 消除：空入口 chunk 合并到 common chunk     │    │
│  │ · 循环依赖检测：BFS 检查 chunk 依赖环              │    │
│  │ · Runtime 模块放置：决定 __exportAll 等 helper 位置 │    │
│  └──────────────────────┬──────────────────────────┘    │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Cross-Chunk Links（跨 chunk 链接）                  │    │
│  │ · 计算每个 chunk 需要导入/导出的符号                  │    │
│  │ · 解析 runtime helper 到具体 chunk                  │    │
│  └──────────────────────┬──────────────────────────┘    │
│                         ▼                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Module Finalization & Rendering（最终渲染）         │    │
│  │ · AST 转换为最终代码（ESM/CJS/IIFE/UMD）           │    │
│  │ · 注入 runtime helper 调用                         │    │
│  │ · Source map 生成                                  │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
                   输出文件（dist/）
```

### 3.3 与 OXC 的关系

Rolldown 深度依赖 [OXC（Oxc）](https://oxc.rs/) 项目的基础设施，两者同属 VoidZero 旗下：

| OXC 组件 | 在 Rolldown 中的用途 | 替代了什么 |
|----------|---------------------|-----------|
| `oxc_parser` | AST 解析 | acorn (Rollup) / Go parser (esbuild) |
| `oxc_resolver` | 模块路径解析 | enhanced-resolve / node-resolve |
| `oxc_transformer` | TypeScript/JSX 转换、语法降级 | Babel / SWC / esbuild transform |
| `oxc_codegen` | AST → 代码生成 | 自研 |
| `oxc_minifier` | 代码压缩（mangler + minifier）| Terser / esbuild minify |
| `oxc_sourcemap` | Source Map 处理 | magic-string sourcemap |

这种**同团队维护的垂直整合架构**是 Rolldown 的核心技术优势——从解析到输出每一层都由同一团队优化，避免了跨项目的序列化/反序列化开销和 API 边界摩擦。

### 3.4 Runtime Helper 机制

Rolldown 内置了一组运行时辅助函数（runtime helpers），用于处理 CJS/ESM 互操作等场景：

| Helper | 用途 |
|--------|------|
| `__commonJSMin` / `__commonJS` | 包装 CJS 模块为可延迟执行的函数 |
| `__toESM` | CJS → ESM 互操作转换 |
| `__toCommonJS` | ESM → CJS 互操作转换 |
| `__exportAll` | 构造模块命名空间对象（用于 `export *` 和动态 import 的 namespace）|
| `__esmMin` / `__esm` | ESM 模块包装（处理循环依赖场景）|
| `__reExport` | 从外部模块重新导出 |
| `__require` | 自定义 require 实现 |

这些 helper 作为一个内置模块（`\0rolldown/runtime.js`）注入，在代码分割时会被分配到合适的 chunk 中。**helper 的 chunk 分配策略是一个容易出 bug 的区域**（见第六节案例）。

---

## 四、性能数据

### 4.1 基准测试

| 对比 | 结果 |
|------|------|
| Rolldown vs Rollup | **10–30× 更快** |
| Rolldown vs esbuild | 接近或超越（部分场景约 **2× 更快**）|
| Vue 核心源码构建 | Rolldown 比 Rollup 快 **7×** |
| WASM 环境（2.5k 模块）| esbuild: 22.19s, Vite(Rollup): 4.52s, **Rolldown: 0.61s** |

### 4.2 真实项目采用数据

| 项目/公司 | 改进 | 具体数据 |
|----------|------|---------|
| **Linear** | 构建时间 | 46s → **6s** |
| **Excalidraw** | 构建时间 | 22.9s → **1.4s**（16× 加速）|
| **GitLab** | 构建时间 + 内存 | 2.5min → **40s**，内存减少 **100×** |
| **PLAID Inc.** | 构建时间 | 1m20s → **5s**（16× 加速）|
| **Appwrite** | 构建时间 + 内存 | 12min+ → **3min**，内存减少 **4×** |
| **Beehiiv** | 构建时间 | 减少 **64%** |
| **Ramp** | 构建时间 | 减少 **57%** |
| **Mercedes-Benz.io** | 构建时间 | 减少 **38%** |

### 4.3 性能优化技术

RC 阶段的 109 个性能优化包括：
- **SIMD JSON 转义**：利用 CPU 向量指令加速字符串处理
- **并行 chunk 生成**：利用 Rust 的 Rayon 并行库
- **优化符号重命名**：更高效的 mangling 算法
- **WASM 构建优化**：Rust → WASM 比 Go → WASM 效率更高

---

## 五、生态与竞品格局

### 5.1 "Rust 重写前端工具链" 浪潮

Rolldown 是近年 Rust 重写 JS 工具链浪潮中的核心项目之一：

| 项目 | 领域 | 团队/公司 | 关系 |
|------|------|---------|------|
| **Rolldown** | 打包器 | VoidZero | 本项目 |
| **OXC** | 解析/转换/检查 | VoidZero | Rolldown 的基础设施 |
| **SWC** | 编译/转换 | Vercel | Turbopack 底层，与 Rolldown 定位不同 |
| **Turbopack** | 打包器 | Vercel | Next.js 专用，非通用方案 |
| **Biome** | Lint + Format | 社区 | 从 Rome 分叉，与 Rolldown 互补 |
| **Lightning CSS** | CSS 处理 | 社区 | Vite 8 中集成 |
| **Rspack** | 打包器 | 字节跳动 | webpack 兼容路线，与 Rolldown 走 Rollup 兼容路线不同 |

### 5.2 竞品定位矩阵

```
                       速度（原生性能）
                          ↑
                          │
            esbuild  ·    │    · Rolldown ← 目标：速度 + 生态兼容
                          │
                          │
         Turbopack  ·     │
                          │
         Rspack  ·        │
                          │
  ──────────────────────────────────────→ 生态兼容性 / 插件丰富度
                          │
                          │
            Rollup  ·     │    · webpack
                          │
```

### 5.3 VoidZero 统一工具链愿景

```
用户代码
  │
  ▼
┌─────────────────────────────────────┐
│         Vite（构建工具 / 开发服务器）    │  ← 用户接触的入口
├─────────────────────────────────────┤
│         Rolldown（打包器）              │  ← 统一 dev + prod 打包
├─────────────────────────────────────┤
│         Oxc（编译器基础设施）             │  ← parser / resolver / transformer / minifier
├─────────────────────────────────────┤
│         Vitest（测试框架）              │  ← 复用 Vite 的模块解析和转换
└─────────────────────────────────────┘

四层由同一团队（VoidZero）维护，共享 AST、resolver、模块互操作语义
```

---

## 六、实际 Bug 案例分析：Runtime Helper 循环依赖

### 6.1 Bug 描述

在我们的 jcdo 项目中升级到 rolldown rc.10 后，发现 1 个残留的循环依赖：

```
daemon-cli-XXXX.js  →  import { __exportAll } from "./index.js"
index.js            →  import { ... } from "./daemon-cli-XXXX.js"
```

ESM 的模块求值顺序导致 `__exportAll` 在被调用时还是 `undefined`。

### 6.2 根因分析

问题出在 `crates/rolldown/src/stages/generate_stage/chunk_optimizer.rs` 的 `optimize_facade_entry_chunks` 函数中：

1. **初始分割阶段**：runtime 模块被分配到 entry chunk（因为只有 entry 的 CJS 转换路径引用了 runtime helper）
2. **Facade 消除阶段**：动态导入的 barrel 模块的 facade entry 被消除，`__exportAll` 被添加到 common chunk 的 `depended_runtime_helper`
3. **循环依赖检测**：现有的 BFS 检测在 `to_temp_idx()` 返回 `None` 时（新创建的 common chunk 未注册到 temp graph），会跳过检查（`unwrap_or(false)`)，允许合并
4. **结果**：common chunk 需要从 entry chunk 导入 `__exportAll`，而 entry chunk 也从 common chunk 导入业务代码 → 循环

### 6.3 修复方案

在 `optimize_facade_entry_chunks` 中添加了安全网：当 runtime 模块在 entry chunk 中，且有其他 chunk 需要 runtime helper 时，将 runtime 提取到独立的 `rolldown-runtime` chunk：

```rust
// chunk_optimizer.rs, 在 runtime 模块分配逻辑之后
} else if let Some(current_runtime_chunk_idx) =
    chunk_graph.module_to_chunk[runtime_module_idx]
{
    let is_entry_chunk = matches!(
        chunk_graph.chunk_table[current_runtime_chunk_idx].kind,
        ChunkKind::EntryPoint { .. }
    );
    let has_external_runtime_dependents = runtime_dependent_chunks
        .iter()
        .any(|&chunk_idx| chunk_idx != current_runtime_chunk_idx);

    if is_entry_chunk && has_external_runtime_dependents {
        // 从 entry chunk 移除 runtime，创建独立 chunk
        // ... 提取逻辑
    }
}
```

### 6.4 相关 Issue

| Issue | 状态 | 描述 |
|-------|------|------|
| [#3650](https://github.com/rolldown/rolldown/issues/3650) | Closed | 同类循环依赖问题首次报告 |
| [#2654](https://github.com/rolldown/rolldown/issues/2654) | Open | 提议为每个 chunk 内联 runtime（更彻底方案）|
| [#8809](https://github.com/rolldown/rolldown/issues/8809) | Open | 我们提交的 issue |
| [#8810](https://github.com/rolldown/rolldown/pull/8810) | Open | 我们提交的修复 PR |

---

## 七、参与贡献指南

### 7.1 开发环境

```bash
# 克隆
git clone https://github.com/rolldown/rolldown.git
cd rolldown

# 安装 Rust 工具链（需要 nightly）
rustup install nightly

# 安装 Node 依赖
pnpm install

# 运行测试
cargo test -p rolldown --test integration

# 只运行特定测试
cargo test -p rolldown --test integration -- <测试名关键词>
```

### 7.2 测试体系

- **Fixture 测试**：每个测试是一个目录，包含 `_config.json`（打包配置）、源文件、`artifacts.snap`（快照）、可选的 `_test.mjs`（运行时验证）
- **路径**：`crates/rolldown/tests/rolldown/`，按主题分类（`issues/`、`code_splitting/`、`optimization/` 等）
- **发现机制**：通过 `_config.json` 文件自动发现（`#[fixture("./tests/rolldown/**/_config.json")]`）

### 7.3 代码贡献建议

基于我们修 bug 的经验：

1. **先理解 chunk 图模型**：Rolldown 的核心是 bit-based 模块分配 + chunk 依赖图优化，理解这个模型是理解大部分 bug 的前提
2. **关注 facade 消除逻辑**：`chunk_optimizer.rs` 是 edge case 最密集的区域
3. **测试覆盖**：新增修复务必添加 fixture 测试，即使无法完美复现 bug，也要覆盖相关代码路径
4. **提 issue 优先**：对于复杂 bug，附带最小复现的 issue 比直接 PR 更有价值

---

## 八、总结与展望

### 8.1 当前状态

- **Rolldown 1.0 RC**（2026-01），API 已稳定
- **Vite 8.0 已发布**（2026-03），Rolldown 作为默认打包器
- 边缘 case 持续修复中（如本文分析的 runtime helper 循环依赖）

### 8.2 未来方向

| 方向 | 状态 |
|------|------|
| Rolldown 1.0 正式版 | RC 期间收集反馈后发布 |
| 持久缓存（Persistent Cache）| 已在 RC 中实验性支持 |
| 每 chunk 内联 runtime（[#2654](https://github.com/rolldown/rolldown/issues/2654)）| 提议中，将彻底解决 runtime 循环依赖 |
| Full Bundle Mode | 实验性支持，dev 启动快 3×，全量热更新快 40% |
| WASM 浏览器运行 | 持续优化中，已远超 esbuild WASM 性能 |

### 8.3 对 JS 生态的影响

Rolldown 代表了前端工具链的一个重要转折点：**从 JS 自举的工具，转向 Rust 编写的原生工具**。结合 VoidZero 的垂直整合战略（Vite + Rolldown + Oxc + Vitest），JavaScript 生态正在经历一次由性能驱动的工具链统一。对于开发者而言，这意味着更快的构建、更一致的行为、更少的配置，以及一个由同一团队端到端维护的工具栈。

---

## 参考资料

- [Rolldown 官方文档](https://rolldown.rs/)
- [Rolldown GitHub 仓库](https://github.com/rolldown/rolldown)
- [Announcing Rolldown 1.0 RC — VoidZero](https://voidzero.dev/posts/announcing-rolldown-rc)
- [Vite 8 Beta: The Rolldown-powered Vite — Vite Blog](https://vite.dev/blog/announcing-vite8-beta)
- [Announcing VoidZero Inc. — VoidZero](https://voidzero.dev/posts/announcing-voidzero-inc)
- [Rolldown Integration Guide — Vite 7](https://v7.vite.dev/guide/rolldown)
- [Rust-Based Drop-in Replacement for Vite, Early Adopters Report 10X Faster Builds — InfoQ](https://www.infoq.com/news/2025/06/rolldown-vite-10x-faster-builds/)
- [VoidZero's Rolldown: Rust Bundler, Rollup Compatible — InfoQ](https://www.infoq.com/news/2025/11/rolldown-bundler-rust/)
- [Vite 8, Rolldown, and Oxc: Rust Is Taking Over the JavaScript Toolchain — DEV Community](https://dev.to/alexcloudstar/vite-8-rolldown-and-oxc-rust-is-taking-over-the-javascript-toolchain-m79)
- [Vite team boasts 10-30x faster builds with Rust-powered Rolldown — DevClass](https://www.devclass.com/development/2026/03/17/vite-team-boasts-10-30x-faster-builds-with-rust-powered-rolldown/5209472)
- [Our Seed Investment in VoidZero — Accel](https://www.accel.com/noteworthies/our-seed-investment-in-voidzero-evan-yous-bold-vision-for-javascript-tooling)
- [OXC 官方文档](https://oxc.rs/)
