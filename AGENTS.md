# SEOBatch

Shopify 嵌入式 App — 批量 SEO 编辑器，基于 React Router v7 + Polaris Web Components + Prisma（SQLite）。

## Project

- **栈**：React Router v7 / React 18 / TypeScript 5 / Vite 6 / Prisma (SQLite)
- **入口**：`app/root.tsx`（HTML shell）；`app/routes/app.tsx`（嵌入式 App 布局 + 导航）
- **配置**：`shopify.app.toml`（客户端 ID、scope、webhook）；Prisma schema 在 `prisma/schema.prisma`
- **Shopify CLI** 管理隧道、环境变量和部署

## Commands

所有命令在项目根目录下执行：

| 命令 | 用途 |
|------|------|
| `npm run dev` | 启动本地开发（Shopify CLI 隧道 + Vite HMR） |
| `npm run build` | 生产构建 |
| `npm run typecheck` | TypeScript 类型检查（`react-router typegen && tsc --noEmit`） |
| `npm run lint` | ESLint |
| `npm run setup` | Prisma generate + migrate deploy |
| `npm run start` | 生产环境启动（`react-router-serve ./build/server/index.js`） |
| `npm run deploy` | 部署到 Shopify 托管 |

## Architecture

- **路由** — 基于文件系统的扁平路由（`@react-router/fs-routes`）。`app/routes/app.*.tsx` → `/app/*`。认证路由：`auth.$.tsx`。Webhook 路由：`webhooks.app.*.tsx`。
- **认证** — `app/shopify.server.ts` 统一导出 `authenticate`（`authenticate.admin(request)` 获取 GraphQL client）、`login`、`registerWebhooks`。
- **GraphQL** — 通过 `admin.graphql(query, { variables })` 调用 Shopify Admin API，返回 JSON。Mutation 和 query 写在 `#graphql` 模板字符串中。
- **UI** — Polaris Web Components（`<s-page>`、`<s-section>`、`<s-button>`、`<s-link>`、`<s-paragraph>`、`<s-stack>`、`<s-box>`、`<s-heading>`）。Toast 通知：`shopify.toast.show()`（来自 `useAppBridge()`）。
- **数据流** — loader 获取数据 → useState 管理前端状态 → useFetcher 提交到 action → action 调用 GraphQL → 返回结果。修改标记通过 `modified` 布尔字段跟踪。
- **数据库** — Prisma + SQLite，仅存储 Session 数据。业务数据通过 Shopify Admin API 直接操作。

## Conventions

- **认证**：所有受保护路由的 loader/action 必须以 `const { admin } = await authenticate.admin(request)` 开头。
- **导航**：导航链接在 `app/routes/app.tsx` 的 `<s-app-nav>` 中定义。新页面路由 → `app/routes/app.<name>.tsx`，导航链接 → `<s-link href="/app/<name>">`。
- **样式**：使用内联 `style` 对象或 CSS Modules（`.module.css`）。缩进 2 空格，UTF-8，LF 换行（`.editorconfig`）。
- **类型**：GraphQL 响应类型手动定义（`interface`）。共享类型在 `app/types.ts`。`import type` 用于仅类型导入。
- **SEO 常量**：`SEO_TITLE_MAX = 60`、`SEO_DESC_MAX = 160` 在 `app/types.ts` 中集中定义。
- **组件**：UI 组件放在 `app/components/`，每个文件一个导出函数。文件路径 = 路由路径映射（flat routes）。
- **错误处理**：loader 用 `throw new Response(...)` 触发 ErrorBoundary；action 用 `data(..., { status })` 返回错误。ErrorBoundary 用 `boundary.error(useRouteError())`。

## Notes

（快速笔记暂存区）
