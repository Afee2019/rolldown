# 002 - Runtime Helper 循环依赖 Bug 完整修复记录

> **日期：** 2026-03-20
> **文档版本：** v1.0
> **涉及项目：** [rolldown/rolldown](https://github.com/rolldown/rolldown)
> **触发项目：** jcdo（私有项目，CLI 工具）
> **Issue：** [#8809](https://github.com/rolldown/rolldown/issues/8809)
> **PR：** [#8810](https://github.com/rolldown/rolldown/pull/8810)

---

## 一、问题发现

### 1.1 背景

jcdo 项目使用 [tsdown](https://github.com/nicepkg/tsdown)（基于 Rolldown 的库打包工具）构建 CLI 应用。升级 Rolldown 到 rc.10 后，原本存在的 12 个循环依赖减少到 1 个——已取得重大改善，但残留的 1 个循环虽不在启动路径上，仍需修复。

### 1.2 现象

构建产物中，`daemon-cli-K2vHAa0H.js`（common chunk）从 `index.js`（entry chunk）导入 `__exportAll` runtime helper：

```javascript
// daemon-cli-K2vHAa0H.js
import { __exportAll } from "./index.js";
//                          ^^^^^^^^^ 从 entry 导入

var daemon_cli_exports = __exportAll({ ... });
```

同时，`index.js` 也从 `daemon-cli-K2vHAa0H.js` 导入业务代码：

```javascript
// index.js
import { registerDaemonCli } from "./daemon-cli-K2vHAa0H.js";
//                                 ^^^^^^^^^^^^^^^^^^^^^^^^ 从 common chunk 导入
```

**形成循环：**

```
index.js ──(static import)──→ daemon-cli-K2vHAa0H.js
    ↑                                    │
    └──(import __exportAll)──────────────┘
```

ESM 模块求值顺序导致 `daemon-cli-K2vHAa0H.js` 在被求值时，`__exportAll` 还是 `undefined`，运行时抛出：

```
TypeError: __exportAll is not a function
```

### 1.3 jcdo 中的临时 workaround

在深入 Rolldown 修复之前，我们在 jcdo 项目中编写了一个 post-build 脚本 `scripts/fix-rolldown-circular-imports.ts`，在 tsdown 构建后自动修补：

1. 扫描 `dist/` 中所有 chunk
2. 找到从 `./index.js` 导入 `__exportAll` 的 chunk
3. 将导入源重写为 `./rolldown-runtime-*.js`
4. 修补后循环依赖数量降为 0

---

## 二、问题定位

### 2.1 jcdo 中的模块关系图

```
index.js（entry）
  ├── (static) → gateway-cli/register.ts
  │                └── (static) → daemon-cli.ts    ← barrel: export * from ...
  ├── (static) → update-cli/register.ts
  │                └── (static) → daemon-cli.ts
  └── ...

completion-cli/register.ts
  └── (dynamic) → import("../daemon-cli.js")       ← 动态导入同一模块
```

关键点：
- **daemon-cli.ts** 是一个纯 re-export 的桶文件（barrel），使用 `export *`
- 它同时被**静态导入**（gateway-cli）和**动态导入**（completion-cli）
- 动态导入需要构造 namespace 对象，这触发了 `__exportAll` runtime helper

### 2.2 为什么 `__exportAll` 会在 entry chunk 中

追踪 Rolldown 的代码分割流程，定位到以下因果链：

**第一步：Bit-based 模块分配**

```
入口点（entry points）：
  bit 0: index.js（用户定义入口）
  bit 1: daemon-cli.js（动态导入入口）
  bit 2 ~ N: 其他动态导入...

模块的 bit 分配（determine_reachable_modules_for_entry）：
  index.js       → {0}
  gateway-cli.js → {0}
  daemon-cli.js  → {0, 1}    ← 从两条路径可达
  cjs-dep        → {0}        ← CJS 依赖，仅 entry 使用
  runtime module → {0}        ← 仅 entry 的 CJS 转换引用了 runtime
```

runtime 模块的 bits 为 `{0}`，与 entry chunk 相同 → **runtime 被分配到 entry chunk**。

**第二步：Chunk 生成**

```
Entry chunk (bit 0):  index.js, gateway-cli.js, cjs-dep, runtime module
Entry chunk (bit 1):  [空 — daemon-cli 在 common chunk 中] ← facade
Common chunk {0,1}:   daemon-cli.js, daemon-impl.js
```

**第三步：Facade 消除**

daemon-cli 的动态入口 chunk（bit 1）是空的（facade），其模块在 common chunk 中。`optimize_facade_entry_chunks` 消除此 facade，将 `__exportAll` 添加到 common chunk 的 `depended_runtime_helper`。

**第四步：跨 chunk 链接**

common chunk 需要 `__exportAll` → 查找 runtime 模块所在 chunk → 在 entry chunk 中 → 从 entry 导入。

entry chunk 需要 daemon-cli 业务代码 → 从 common chunk 导入。

**循环形成。**

### 2.3 现有防护为何失效

`chunk_optimizer.rs` 第 736–747 行有一个循环依赖检测：

```rust
// chunk_optimizer.rs:736-747
if temp_runtime_chunk_idx
    .and_then(|temp_runtime_idx| {
        let temp_target_idx = temp_chunk_opt_graph.to_temp_idx(target_chunk_idx)?;
        Some(
            temp_chunk_opt_graph
                .would_create_circular_dependency(temp_runtime_idx, temp_target_idx),
        )
    })
    // If runtime is not included before, it will not create circular dependency...
    // If either index has no temp counterpart, we conservatively allow the merge.
    .unwrap_or(false)
{
    continue; // 跳过此 facade 消除
}
```

此检测用 BFS 遍历 chunk 依赖图判断合并是否会产生环。**但当 `to_temp_idx(target_chunk_idx)` 返回 `None` 时（目标 common chunk 在 temp graph 初始化之后才创建，未注册），整个表达式走 `unwrap_or(false)` 分支 → 检测被跳过 → facade 消除被允许。**

这就是 rc.10 修了 11 个但漏了 1 个的原因：大多数 common chunk 在 temp graph 中有映射，但特定条件下新创建的 chunk 没有。

### 2.4 代码路径总结

```
code_splitting.rs
  └── split_chunks()
        ├── determine_reachable_modules_for_entry()  ← bit 传播
        ├── 模块分配到 chunks（包括 runtime → entry chunk）
        ├── try_insert_common_module_to_exist_chunk() ← 可能创建新 common chunk
        └── optimize_facade_entry_chunks()
              └── chunk_optimizer.rs
                    ├── find_facade_chunk_merge_ops()
                    │     └── would_create_circular_dependency() ← 可能被跳过
                    ├── facade 消除 → __exportAll 添加到 common chunk
                    └── runtime 模块放置逻辑
                          ├── 原有逻辑：仅处理 runtime 未分配的情况
                          └── 缺失逻辑：runtime 已在 entry chunk 的情况 ← BUG
```

---

## 三、修复方案

### 3.1 修复思路

在 `optimize_facade_entry_chunks` 的 runtime 模块放置逻辑中，补充处理 **runtime 已分配到 entry chunk** 的情况。当检测到：

1. runtime 模块已在 entry chunk 中（`is_entry_chunk`）
2. facade 消除后有其他 chunk 需要 runtime helper（`has_external_runtime_dependents`）

则将 runtime 模块从 entry chunk 提取到独立的 `rolldown-runtime` chunk，打断循环。

### 3.2 修复代码

**文件：** `crates/rolldown/src/stages/generate_stage/chunk_optimizer.rs`

在原有 runtime 未分配处理逻辑（行 1005–1031）之后，添加 `else if` 分支：

```rust
// 原有逻辑：runtime 未分配到任何 chunk
if chunk_graph.module_to_chunk[runtime_module_idx].is_none()
    && !runtime_dependent_chunks.is_empty()
{
    // ... 创建或复用 chunk，分配 runtime（已有代码，未修改）
}
// ===== 新增逻辑 =====
else if let Some(current_runtime_chunk_idx) =
    chunk_graph.module_to_chunk[runtime_module_idx]
{
    // runtime 已在某个 chunk 中（通常是 entry chunk）。
    // 如果 facade 消除向其他 chunk 添加了 runtime helper 依赖，
    // 那些 chunk 需要从 runtime 所在的 chunk 导入 helper。
    // 当 runtime 所在的 chunk 也从那些 dependent chunk 导入时，循环产生。
    //
    // 仅当 runtime 在 entry chunk 时才需要提取——entry chunk 会依赖 common chunk，
    // 形成环。若 runtime 已在 dedicated common chunk（如手动分包），则无环风险。
    let is_entry_chunk = matches!(
        chunk_graph.chunk_table[current_runtime_chunk_idx].kind,
        ChunkKind::EntryPoint { .. }
    );
    let has_external_runtime_dependents = runtime_dependent_chunks
        .iter()
        .any(|&chunk_idx| chunk_idx != current_runtime_chunk_idx);

    if is_entry_chunk && has_external_runtime_dependents {
        // 从 entry chunk 移除 runtime 模块
        chunk_graph.chunk_table[current_runtime_chunk_idx]
            .modules
            .retain(|&m| m != runtime_module_idx);

        // 创建独立的 runtime chunk
        let runtime_chunk = Chunk::new(
            Some("rolldown-runtime".into()),
            None,
            index_splitting_info[runtime_module_idx].bits.clone(),
            vec![],
            ChunkKind::Common,
            input_base.clone(),
            None,
        );
        let new_runtime_chunk_idx = chunk_graph.add_chunk(runtime_chunk);
        chunk_graph.module_to_chunk[runtime_module_idx] = Some(new_runtime_chunk_idx);
        chunk_graph.chunk_table[new_runtime_chunk_idx].modules.push(runtime_module_idx);
        chunk_graph.chunk_table[new_runtime_chunk_idx]
            .depended_runtime_helper
            .insert(self.link_output.metas[runtime_module_idx].depended_runtime_helper);
    }
}
```

### 3.3 设计决策与权衡

| 决策 | 理由 |
|------|------|
| **仅对 entry chunk 提取** | common chunk 不会从其依赖者导入，不存在环风险。限制触发条件可避免不必要的 chunk 拆分 |
| **创建独立 chunk 而非内联** | 与 manual code splitting 的 `extract_runtime_chunk` 行为一致；runtime 代码极小（<1KB），额外 chunk 成本可忽略 |
| **不修改 facade 消除逻辑** | facade 消除本身的循环检测是正确的，只是在 temp graph 不完整时有盲区；本修复是安全网而非替代 |
| **不修改 bit 传播逻辑** | runtime 的 bit 分配反映了真实依赖关系，强制扩展 bit 会引入不正确的共享语义 |

### 3.4 迭代过程

修复并非一次到位，经历了以下迭代：

**第一版（过于激进）—— 10 个测试失败：**

```rust
// 条件：只要有 external runtime dependents 就提取
if has_external_runtime_dependents { ... }
```

问题：
- manual code splitting 的测试中，runtime 已在独立 common chunk（由 `extract_runtime_chunk` 创建），不需要再次提取
- 从仅含 runtime 的 chunk 中移除 runtime 后，该 chunk 变为空 common chunk，在 chunk 排序时 `a.modules[0]` 引发 `index out of bounds` panic

**第二版（最终版）—— 0 个新增失败：**

```rust
// 条件：runtime 在 entry chunk + 有 external dependents 才提取
if is_entry_chunk && has_external_runtime_dependents { ... }
```

新增 `is_entry_chunk` 守卫后：
- manual code splitting（runtime 在 common chunk）不受影响
- 仅在真正有环风险的场景触发

---

## 四、测试验证

### 4.1 新增测试用例

**目录：** `crates/rolldown/tests/rolldown/issues/runtime_helper_circular_dependency/`

**文件结构：**

```
runtime_helper_circular_dependency/
├── _config.json        ← 打包配置（双入口）
├── _test.mjs           ← 运行时验证（检测循环导入）
├── entry.js            ← 主入口：使用 CJS dep + 静态导入 gateway
├── cjs-dep.cjs         ← CJS 模块：强制 runtime helper 进入 entry chunk
├── gateway.js          ← 静态导入 daemon barrel
├── daemon.js           ← barrel 模块：export * from './daemon-impl.js'
├── daemon-impl.js      ← 实际实现
├── completion.js       ← 第二入口：动态导入 daemon（触发 __exportAll）
└── artifacts.snap      ← 快照：记录预期输出
```

**测试配置（`_config.json`）：**

```json
{
  "config": {
    "input": [
      { "name": "entry", "import": "entry.js" },
      { "name": "completion", "import": "completion.js" }
    ]
  }
}
```

**运行时验证（`_test.mjs`）：**

```javascript
// 检测循环依赖：任何非 entry chunk 不应同时满足：
// 1. 从 entry.js 导入（获取 runtime helper）
// 2. 被 entry.js 导入（提供业务代码）
for (const file of files) {
  if (file !== 'entry.js' && content.includes('from "./entry.js"')) {
    const entryImportsFromThis = entryContent.includes(`from "./${file}"`);
    assert(
      !entryImportsFromThis,
      `Circular dependency: ${file} imports from entry.js AND entry.js imports from ${file}`,
    );
  }
}
```

### 4.2 全量回归测试

```bash
$ cargo test -p rolldown --test integration -- rolldown_fixture

test result: 748 passed; 4 failed; 10 ignored
```

4 个失败为**预先存在的问题**（缺少 `cjs-module-lexer` npm 依赖），在 clean main 分支上同样失败：

```bash
# 验证：在 clean main 上同样失败
$ git stash
$ cargo test ... -- 'cjs_module_lexer_compat__exports'
test result: FAILED. 0 passed; 1 failed  ← 与修复无关
$ git stash pop
```

### 4.3 关键测试覆盖

| 测试类别 | 数量 | 状态 |
|---------|------|------|
| 新增 runtime helper 循环依赖测试 | 1 | 通过 |
| code_splitting 测试 | ~50 | 全部通过 |
| chunk_merging 优化测试 | 8 | 全部通过（含 dynamic_entry_merged_in_common_chunk 系列）|
| manual code splitting 测试 | ~20 | 全部通过 |
| 全量 fixture 测试 | 748 | 全部通过 |

---

## 五、提交与发布

### 5.1 Git 操作

```bash
# 创建分支
git checkout -b fix/runtime-helper-circular-dependency

# 暂存修改文件
git add crates/rolldown/src/stages/generate_stage/chunk_optimizer.rs \
        crates/rolldown/tests/rolldown/issues/runtime_helper_circular_dependency/

# 提交
git commit -m "fix: extract runtime module from entry chunk to prevent circular dependencies"

# 推送到 fork
git push -u origin fix/runtime-helper-circular-dependency
```

### 5.2 Issue 提交

**Issue [#8809](https://github.com/rolldown/rolldown/issues/8809)：**

标题：`Runtime helper circular dependency when __exportAll is in entry chunk during code-splitting`

内容包含：
- Bug 描述与最小复现结构
- 构建产物中循环导入的具体表现
- 根因分析（4 步因果链）
- 与 #3650、#2654 的关联

### 5.3 PR 提交

**PR [#8810](https://github.com/rolldown/rolldown/pull/8810)：**

标题：`fix: extract runtime module from entry chunk to prevent circular dependencies`

内容包含：
- Summary（修复要点）
- Test plan（748 测试通过 + 新增回归测试）
- 关联 issue（Fixes #8809）

### 5.4 变更统计

```
 10 files changed, 169 insertions(+)

 修改文件：
   chunk_optimizer.rs                     +42 行（核心修复）

 新增文件（测试用例）：
   _config.json                           +8 行
   _test.mjs                              +31 行
   artifacts.snap                         +63 行
   cjs-dep.cjs                            +3 行
   completion.js                          +5 行
   daemon-impl.js                         +2 行
   daemon.js                              +2 行
   entry.js                               +7 行
   gateway.js                             +6 行
```

---

## 六、相关 Issue 与更彻底方案

### 6.1 相关 Issue 矩阵

| Issue | 状态 | 描述 | 与本 Bug 的关系 |
|-------|------|------|---------------|
| [#3650](https://github.com/rolldown/rolldown/issues/3650) | Closed | 首次报告 advancedChunks 导致的 runtime 循环依赖 | 同类问题，修复了大部分但非全部 |
| [#2654](https://github.com/rolldown/rolldown/issues/2654) | Open | 提议为每个 chunk 内联 runtime | 更彻底的方案，可从根本消除此类问题 |
| [#7874](https://github.com/rolldown/rolldown/issues/7874) | Open (1.4) | `export *` 输出不利于下游 tree-shaking | 相关但不同问题 |
| [#8809](https://github.com/rolldown/rolldown/issues/8809) | Open | 本次提交的 issue | 当前 Bug |
| [#8810](https://github.com/rolldown/rolldown/pull/8810) | Open | 本次提交的 PR | 当前修复 |

### 6.2 更彻底的方案

**方案 A（本 PR）：安全网式提取**
- 检测 runtime 在 entry chunk + 外部依赖者 → 提取到独立 chunk
- 优点：改动小、风险低、兼容现有行为
- 缺点：是补救措施而非根治

**方案 B（[#2654](https://github.com/rolldown/rolldown/issues/2654)）：每 chunk 内联 runtime**
- 每个需要 runtime helper 的 chunk 都内联一份 helper 代码
- 优点：彻底消除跨 chunk runtime 导入，不存在循环可能
- 缺点：增加总体积（runtime 极小，约 <1KB，影响可忽略），需要更大改动

**方案 C：修复 temp graph 注册缺口**
- 确保 `try_insert_common_module_to_exist_chunk` 创建的所有 chunk 都在 temp graph 中注册
- 优点：修复根因（循环检测的盲区）
- 缺点：需要深入理解 temp graph 与 chunk graph 的同步机制

---

## 七、经验总结

### 7.1 调试技巧

1. **从构建产物反推**：先分析 `dist/` 中的循环导入关系，确定涉及哪些 chunk 和 symbol
2. **追踪 bit 传播**：理解模块的 bit 分配是理解 chunk 分割的关键（`determine_reachable_modules_for_entry`）
3. **关注 facade 消除**：`chunk_optimizer.rs` 中的 facade 消除逻辑是边缘 case 最密集的区域
4. **善用快照测试**：Rolldown 的 `artifacts.snap` 让你一眼看出 chunk 内容和导入关系
5. **迭代式修复**：先写最简单的修复，通过测试失败来理解守卫条件应该多严格

### 7.2 Rolldown 代码分割的关键心智模型

```
                        编译时确定                    运行时确定
                    ┌──────────────┐              ┌──────────────┐
                    │  bit 传播     │              │  ESM 求值顺序  │
                    │  chunk 分配   │              │  import 链解析 │
                    │  facade 消除  │──→ 输出文件 ──→│  循环时 undefined│
                    │  runtime 放置 │              │               │
                    └──────────────┘              └──────────────┘
                         ↑                              ↑
                    可控、可修复                     后果严重、难调试
```

关键原则：**编译时的 chunk 拆分决策必须保证运行时不产生导致 `undefined` 的循环导入。** 现有的 `would_create_circular_dependency` BFS 检测覆盖了大部分场景，但在 temp graph 不完整时有盲区，本修复作为安全网补充了这一缺口。

### 7.3 对 Rolldown 贡献者的建议

1. **修改 chunk_optimizer.rs 时务必运行全量测试**——这个文件的改动容易引发级联失败
2. **理解三层 chunk 图**：chunk_graph（最终图）、temp_chunk_opt_graph（优化时的临时图）、bit-based 分配（初始图），三者的同步是 bug 的常见源头
3. **manual code splitting 有独立的 runtime 提取逻辑**（`extract_runtime_chunk`），改 runtime 放置时注意两条路径的一致性
4. **Facade 消除有 5 种场景**（见 `find_facade_chunk_merge_ops` 注释），每种的 runtime 交互不同

---

## 附录 A：关键源码位置索引

| 文件 | 行号 | 功能 |
|------|------|------|
| `chunk_optimizer.rs` | 659–750 | `find_facade_chunk_merge_ops`：查找可消除的 facade chunk |
| `chunk_optimizer.rs` | 736–747 | 循环依赖 BFS 检测（`would_create_circular_dependency`）|
| `chunk_optimizer.rs` | 830–1074 | `optimize_facade_entry_chunks`：主优化入口 |
| `chunk_optimizer.rs` | 906–993 | Facade 消除循环：添加 `__exportAll` 到 target chunk |
| `chunk_optimizer.rs` | 1005–1074 | Runtime 模块放置逻辑（含本次修复）|
| `code_splitting.rs` | 785–895 | `split_chunks`：初始代码分割 |
| `code_splitting.rs` | 898–930 | `determine_reachable_modules_for_entry`：bit 传播 |
| `manual_code_splitting.rs` | 224–257 | `extract_runtime_chunk`：手动分包的 runtime 提取 |
| `chunk_graph.rs` | 63–72 | `add_module_to_chunk`：模块分配到 chunk |
| `compute_cross_chunk_links.rs` | 300–305 | 根据 `depended_runtime_helper` 解析跨 chunk 导入 |
| `module_finalizers/mod.rs` | 547–574 | `__exportAll` 调用代码生成 |
| `rolldown_common/.../runtime_helper.rs` | — | `RuntimeHelper` bitflags 定义（ExportAll = bit 11）|

## 附录 B：复现用最小项目结构

如需在独立项目中复现此 bug，构造以下结构：

```
project/
├── src/
│   ├── entry.js          ← import CJS dep + import gateway
│   ├── cjs-dep.cjs       ← exports.helper = function() {}
│   ├── gateway.js        ← import { start } from './daemon.js'
│   ├── daemon.js         ← export * from './daemon-impl.js'
│   ├── daemon-impl.js    ← export function start() {}
│   └── completion.js     ← export async function complete() { await import('./daemon.js') }
├── rolldown.config.js
│   └── input: ['src/entry.js', 'src/completion.js']
└── package.json
```

触发条件：
1. entry.js 使用 CJS 模块（`__commonJSMin` → runtime 进入 entry chunk）
2. daemon.js 使用 `export *`（barrel）
3. daemon.js 同时被静态导入（gateway）和动态导入（completion）
4. daemon.js 因 multi-entry 被分到 common chunk
5. 动态入口 facade 被消除，`__exportAll` 被添加到 common chunk
6. common chunk 从 entry 导入 `__exportAll`，entry 从 common chunk 导入业务代码 → 循环
