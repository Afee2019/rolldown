# 003 - jcdo 项目 Rolldown 循环依赖临时 Workaround

> **日期：** 2026-03-20
> **文档版本：** v1.0
> **状态：** ⚠️ 临时措施，待上游修复后移除
> **上游 Issue：** [rolldown/rolldown#8809](https://github.com/rolldown/rolldown/issues/8809)（Open）
> **上游 PR：** [rolldown/rolldown#8810](https://github.com/rolldown/rolldown/pull/8810)（Closed，维护者倾向根因修复）

---

## 一、Workaround 概述

### 1.1 问题回顾

Rolldown rc.10 在 code-splitting 时，将 `__exportAll` runtime helper 内联到 entry chunk（`index.js`），导致其他 chunk 从 `index.js` 导入该 helper，形成循环依赖。ESM 求值顺序使得 `__exportAll` 在调用时为 `undefined`。

详细根因分析见 [002-Runtime-Helper循环依赖Bug完整修复记录.md](./002-Runtime-Helper循环依赖Bug完整修复记录.md)。

### 1.2 Workaround 方案

在 tsdown 构建之后，立即运行一个 post-build 脚本：

1. 在 `dist/` 中找到 `rolldown-runtime-*.js` 独立 chunk
2. 扫描所有 chunk，找到从 `./index.js` 导入 `__exportAll` 的文件
3. 将导入源从 `./index.js` 重写为 `./rolldown-runtime-*.js`

---

## 二、实现细节

### 2.1 脚本文件

**路径：** `~/dev/jcdo/scripts/fix-rolldown-circular-imports.ts`

```typescript
/**
 * Post-build fix for rolldown circular __exportAll imports.
 *
 * rolldown may place the __exportAll runtime helper inside index.js and then
 * have chunks import it back from "./index.js", creating a circular dependency
 * that blows up at runtime (TypeError: __exportAll is not a function).
 *
 * This script rewrites those imports to point at the standalone
 * rolldown-runtime chunk instead.
 *
 * See: https://github.com/rolldown/rolldown/issues/3650
 */

import fs from "node:fs";
import path from "node:path";

const distDir = path.resolve(import.meta.dirname, "../dist");

// 1. Find the rolldown-runtime chunk
const files = fs.readdirSync(distDir).filter((f) => f.endsWith(".js"));
const runtimeChunk = files.find((f) => f.startsWith("rolldown-runtime-"));
if (!runtimeChunk) {
  // No standalone runtime chunk — nothing to fix.
  process.exit(0);
}

// 2. Scan for chunks that import { t as __exportAll } from "./index.js"
const pattern = /from\s+"\.\/index\.js"/;
let fixed = 0;

for (const file of files) {
  if (file === "index.js" || file === runtimeChunk) {
    continue;
  }

  const filePath = path.join(distDir, file);
  const content = fs.readFileSync(filePath, "utf8");

  // Only patch lines that import __exportAll from index.js
  if (!content.includes("__exportAll") || !pattern.test(content)) {
    continue;
  }

  const updated = content.replace(
    /import\s*\{([^}]*__exportAll[^}]*)\}\s*from\s*"\.\/index\.js"/g,
    `import {$1} from "./${runtimeChunk}"`,
  );

  if (updated !== content) {
    fs.writeFileSync(filePath, updated, "utf8");
    fixed++;
    console.log(`[fix-rolldown-circular] patched ${file}`);
  }
}

if (fixed > 0) {
  console.log(`[fix-rolldown-circular] fixed ${fixed} file(s)`);
} else {
  console.log("[fix-rolldown-circular] no circular __exportAll imports found");
}
```

### 2.2 构建流程集成

**路径：** `~/dev/jcdo/package.json` → `scripts.build`

脚本被插入到 `tsdown` 之后、后续构建步骤之前：

```
tsdown && node --import tsx scripts/fix-rolldown-circular-imports.ts && pnpm ui:build && ...
```

完整 build 命令链：

```
pnpm canvas:a2ui:bundle
  && tsdown                                                    ← Rolldown 打包
  && node --import tsx scripts/fix-rolldown-circular-imports.ts ← ⚠️ Workaround
  && pnpm ui:build
  && pnpm build:plugin-sdk:dts
  && node --import tsx scripts/write-plugin-sdk-entry-dts.ts
  && node --import tsx scripts/canvas-a2ui-copy.ts
  && node --import tsx scripts/copy-hook-metadata.ts
  && node --import tsx scripts/copy-export-html.ts
  && node --import tsx scripts/write-build-info.ts
  && node --import tsx scripts/write-cli-compat.ts
```

### 2.3 脚本行为

| 场景 | 行为 |
|------|------|
| 不存在 `rolldown-runtime-*.js` | 静默退出（`process.exit(0)`），无需修补 |
| 存在 runtime chunk 但无循环导入 | 输出 `no circular __exportAll imports found` |
| 检测到循环导入 | 修补文件，输出 `patched <filename>`，最后汇总 `fixed N file(s)` |

当前 jcdo 构建中实际修补了 **1 个文件**（`daemon-cli-*.js`）。

---

## 三、验证方法

### 3.1 构建后检查

```bash
# 完整构建
pnpm build

# 检查是否还有循环导入
grep -r '__exportAll.*from.*"\.\/index\.js"' dist/
# 预期：无输出（已修补）

# 检查修补后的导入指向
grep -r '__exportAll' dist/ --include='*.js' | grep -v index.js | grep -v rolldown-runtime
# 预期：无输出（所有 __exportAll 导入都指向 rolldown-runtime）
```

### 3.2 运行时验证

```bash
# Gateway 服务启动测试（触发 daemon-cli 路径）
node dist/index.js gateway status
# 预期：正常输出服务状态，无 TypeError

# HTTP 健康检查
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8255/
# 预期：200
```

---

## 四、移除条件与清理步骤

### 4.1 何时移除

满足以下**任一**条件时应移除此 workaround：

1. **上游修复合并：** [rolldown/rolldown#8809](https://github.com/rolldown/rolldown/issues/8809) 被关闭（Fixed），且修复包含在 jcdo 使用的 rolldown 版本中
2. **Rolldown 升级后验证通过：** 升级 rolldown/tsdown 后，构建产物中不再出现 `__exportAll` 从 `./index.js` 导入的情况
3. **方案 B 实施：** [rolldown/rolldown#2654](https://github.com/rolldown/rolldown/issues/2654)（每 chunk 内联 runtime）被实现，从根本上消除跨 chunk runtime 导入

### 4.2 清理步骤

```bash
# 1. 升级 rolldown/tsdown 到包含修复的版本
pnpm update tsdown

# 2. 构建并验证不再需要修补
pnpm build
# 观察输出：如果显示 "no circular __exportAll imports found"，说明已修复

# 3. 确认无循环导入
grep -r '__exportAll.*from.*"\.\/index\.js"' dist/
# 确认无输出

# 4. 运行时验证
node dist/index.js gateway status

# 5. 移除 workaround 脚本
mv scripts/fix-rolldown-circular-imports.ts scripts/fix-rolldown-circular-imports.ts.backup

# 6. 从 package.json build 命令中移除脚本调用
# 将:
#   tsdown && node --import tsx scripts/fix-rolldown-circular-imports.ts && pnpm ui:build
# 改为:
#   tsdown && pnpm ui:build

# 7. 提交
git add -A && git commit -m "chore: remove rolldown circular import workaround (fixed upstream)"
```

### 4.3 监控检查清单

| 检查项 | 操作 |
|--------|------|
| GitHub Issue #8809 状态 | `gh issue view 8809 -R rolldown/rolldown --json state` |
| Rolldown 新版本 changelog | 检查是否提及 runtime/circular/chunk 相关修复 |
| 构建输出 | 观察 `[fix-rolldown-circular]` 日志行——`fixed 0 file(s)` 连续出现说明已不需要 |
| tsdown/rolldown 版本 | 记录当前使用版本：rolldown rc.10，待升级时对照 |

---

## 五、风险评估

### 5.1 Workaround 自身的风险

| 风险 | 等级 | 说明 |
|------|------|------|
| 正则匹配过宽 | 低 | 仅匹配包含 `__exportAll` 且来自 `./index.js` 的 import，误伤概率极低 |
| runtime chunk 命名变更 | 低 | 依赖 `rolldown-runtime-` 前缀，这是 Rolldown 的稳定命名约定 |
| 多个 runtime helper 混合导入 | 低 | 正则 `{([^}]*__exportAll[^}]*)}` 保留了同一 import 语句中的其他 binding |
| 构建性能开销 | 可忽略 | 仅扫描 dist/ 中的 JS 文件，毫秒级完成 |

### 5.2 不移除的长期风险

- 脚本本身无害（检测到无需修补时静默通过），但会增加构建流程的认知负担
- 未来如果 Rolldown 变更 chunk 命名或 import 语法格式，脚本可能失效但不会报错（静默跳过）
- 建议在上游修复后及时清理，避免成为"永远的临时方案"
