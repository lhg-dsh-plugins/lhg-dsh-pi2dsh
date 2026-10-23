# BUGFIX — renderMcpPatch 把字符串裸输出为 YAML 纯量，解析后类型被改写

## Bug 标题

`src/mcp-config.ts` 的 `yamlScalar()` 只用字符集正则判断字符串能否裸输出为 YAML 纯量（plain scalar），未排除会被 YAML 解析器读成非字符串的取值。形如数字、布尔、null、日期的字符串（`"8080"`、`"true"`、`"null"`、`"2026-10-23"`）全部通过字符集校验、裸输出进 cordis patch；patch 被宿主解析后变成 number / boolean / null / 时间戳，静默破坏本模块自己声明的字符串契约（`PiMcpServer.args?: string[]`、`env`/`headers?: Record<string, string>`）——例如 MCP 服务器的命令行参数 `--port 8080` 落到 patch 里是整数 8080、环境变量 `DEBUG=false` 落成布尔 false。

## 证据链（修复前 → 修复后）

最小复现（离线，本仓直接依赖的 `yaml` 包作解析口径）：构造 `.mcp.json`，服务器 `args: ["--port", "8080", "--mode", "true"]`、`env: { RETRIES: "3", DEBUG: "false", EMPTYISH: "null", PLAIN: "abc" }`，经 `convertPiMcpConfig` + `renderMcpPatch` 渲染后用 YAML 解析读回。

修复前输出（一行摘要）：`args: "8080" (number), "true" (boolean); env: RETRIES=3 (number), DEBUG=false (boolean), EMPTYISH=null`，复现脚本 exit 1。

`git stash` 对照：同一复现脚本在基线（stash 掉修复）下 exit 1、打入修复后 exit 0，往返类型全部恢复为 string，普通字符串（如 `PLAIN: abc`、`--port`）仍保持裸输出、渲染形态不变。

## 根因

YAML 纯量的类型由解析器的标量解析规则决定，不由字符集决定。旧实现 `yamlScalar` 的唯一判据是 `/^[A-Za-z0-9@._\/:-]+$/`，该字符集恰好完整覆盖了数字字面量、布尔/null 词与 ISO 日期，于是本应为字符串的值被系统性地裸输出。

## 改动文件

- `src/mcp-config.ts`（唯一逻辑改动）：新增 `yamlBareSafe()`——在原字符集判据之外，再排除非字符串词（true/false/yes/no/on/off/y/n/null/~，含 YAML 1.1 词集）、`.inf`/`.nan`、数值字面量（含十六进制/八进制/二进制/科学计数法/下划线分隔）、ISO 时间戳；不安全的字符串改走原有的 `JSON.stringify` 引号路径。映射键同理（字符集内但会被改类型的键加引号），同一根因一并处理。
- `tests/mcp-config.spec.ts`：新增 1 条回归测试（渲染 → YAML 解析回读 → 断言 args/env 全为字符串）；上游既有测试未改。
- 契约动作（非逻辑）：`package.json` 版本 0.27.0 → 0.27.1、`repository.url` 改指本落点仓；README 顶部追加 fork 说明；新增本文件与 `UPSTREAM.md`；LICENSE 原样。上游运行时从 `package.json` 读版本（`src/source.ts`），无其他版本戳常量；`pnpm-lock.yaml` 不含根包版本字段，无待同步项。

## 复验结果

- 复现脚本：修复前 exit 1 → 修复后 exit 0（stash 对照复核）。
- `pnpm check`（tsc --noEmit）：exit 0。
- `pnpm check:docs`：exit 0（capability docs up to date）。
- 全量 vitest：374/374 通过（基线 373 + 新增回归 1），零回归。

## 验证边界（如实说明）

- 未在真实 DSH 宿主上做 patch 装载端到端验证；解析口径以本仓直接依赖的 `yaml` 包（YAML 1.2）为准，修复的排除词集对 js-yaml（YAML 1.1 布尔词集）同样保守覆盖。
- 上游 open issue #7（DSH 0.2.0-rc.2 安装期 peer 拒绝）与 #8（DSH 0.1.7-rc.2 子代理空 result）属宿主版本兼容问题：HEAD 已迁移到 DSH 0.2.1-rc.2 且 peer 范围已更新，不在本版修复范围，未动。
- 上游 AGENTS.md 要求改动兼容行为时同步其 architecture mapping 文档与 `src/compatibility.ts` 清单；本 fork 按最小改动契约只动 bug 相关文件，未同步该文档体系，特此记录差异。
