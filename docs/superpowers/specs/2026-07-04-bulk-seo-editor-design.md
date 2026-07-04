# 批量 SEO 编辑器 — 设计规格

> 基于 SEOBatch 项目，实现 Shopify 产品的批量 SEO 编辑功能。

## 1. 概述

在 Shopify 嵌入式 App 中增加一个批量 SEO 编辑器页面，商家可以在一张表格中同时查看和编辑多个产品的 Meta Title、Meta Description 和 URL Handle，并提供实时搜索引擎预览和字符长度提示。

### 核心需求

| 需求 | 说明 |
|------|------|
| 编辑对象 | 产品（Products） |
| 编辑字段 | Meta Title、Meta Description、URL Handle |
| 编辑方式 | 行内编辑表格（Inline Table Editing） |
| 预览 | 实时 Google 搜索结果片段预览 |
| 验证 | Meta Title ≤60 字符、Meta Description ≤160 字符提示 |
| 保存 | 一键批量保存所有修改 |
| 数据规模 | ≤500 个产品 |

## 2. 方案对比

### 方案 A（⭐ 推荐 — 已选定）

**行内编辑表格**：所有产品在一个表格中，每行可直接编辑 SEO 字段，右侧实时显示搜索引擎预览。

优点：
- 最直观的批量编辑体验
- 小规模数据（<500）加载流畅
- 实时 Google Snippet 预览
- 一键保存全部修改

### 方案 B（备选）

**筛选 → 勾选 → 批量编辑**：先筛选产品，勾选目标后进入独立编辑界面。

### 方案 C（备选）

**双面板设计**：左侧列表 + 右侧详情编辑 + 支持模板批量应用。

## 3. 页面设计

### 3.1 布局结构

```
┌──────────────────────────────────────────────────┐
│  🔍 SEO 批量编辑                          头部   │
│  共 128 个产品 | 已修改 5 项  [🔍搜索...] [💾保存] │
├────────┬──────────┬──────────┬──────┬────────────┤
│ 产品    │ Meta Title│ Meta Desc│ Handle│ 搜索引擎  │
│ (图片+名)│ (≤60)   │ (≤160)   │      │ 预览     │
├────────┼──────────┼──────────┼──────┼────────────┤
│ 🖼️ T恤 │ [输入框] │ [输入框]  │ [...]│ Google     │
│        │ 28 chars✓│ 95 chars✓│      │ snippet    │
├────────┼──────────┼──────────┼──────┼────────────┤
│ 🖼️ 卫衣│ [输入框⚠️]│ [输入框⚠️] │ [...]│ Google     │
│        │ 72 chars │ 198 chars│      │ snippet    │
├────────┼──────────┼──────────┼──────┼────────────┤
│ 🖼️ 外套│ [未设置] │ [未设置]  │ [...]│ 未设置提示 │
│        │ ⚠️       │ ⚠️       │      │            │
└────────┴──────────┴──────────┴──────┴────────────┘
```

### 3.2 表格列说明

| 列 | 类型 | 说明 |
|----|------|------|
| 产品 | 只读 | 产品缩略图 + 标题 + SKU，不可编辑 |
| Meta Title | 文本输入框 | 实时显示字符数，超过 60 显示橙色警告 |
| Meta Description | 文本输入框 | 实时显示字符数，超过 160 显示橙色警告 |
| URL Handle | 文本输入框 | 产品 URL 别名，可编辑 |
| 搜索引擎预览 | 只读 | 模拟 Google 搜索结果卡片，实时反映编辑内容 |

### 3.3 交互细节

- 编辑任意字段后，该行标记为"已修改"，顶部"已修改"计数更新
- 搜索引擎预览随输入实时更新
- 未设置 SEO 字段显示占位符"未设置"和橙色提示
- 保存按钮仅在有关修改时可用（若无修改则禁用）

## 4. 数据流

### 4.1 加载

```
App 加载 → loader 调用 Shopify Admin GraphQL
  → query products(first: 250) { nodes { id, title, handle, seo { title, description }, featuredImage { url } } }
  → 如有分页，自动请求下一页
  → 返回 products 数组给前端
  → 前端渲染表格
```

### 4.2 编辑

```
用户编辑表格中的字段
  → React state 同步更新对应产品的 SEO 数据
  → 标记该产品为 "modified"
  → 实时更新搜索引擎预览
  → 更新顶部 "已修改" 计数
```

### 4.3 保存

```
用户点击 "保存全部修改"
  → 遍历所有标记为 modified 的产品
  → 对每个产品调用 productUpdate(input: { id, seo: { title, description }, handle })
  → 显示保存进度（如 "正在保存 3/5"）
  → 所有完成后汇总结果
  → toast 通知 "已成功更新 N 个产品的 SEO 信息"
  → 若有失败项，显示 "N 个产品保存失败" 并提供重试按钮
```

### 4.4 使用的 GraphQL

**查询产品（loader）**：
```graphql
query LoadProducts($first: Int!, $after: String) {
  products(first: $first, after: $after, sortKey: TITLE) {
    nodes {
      id
      title
      handle
      seo { title description }
      featuredImage { url }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**更新产品（action）**：
```graphql
mutation UpdateProductSEO($input: ProductUpdateInput!) {
  productUpdate(product: $input) {
    product { id seo { title description } handle }
    userErrors { field message }
  }
}
```

## 5. 路由与文件

### 新增文件

| 文件 | 说明 |
|------|------|
| `app/routes/app.seo-bulk.tsx` | 批量 SEO 编辑器主页面，包含 loader/action/UI |

### 修改文件

| 文件 | 变更 |
|------|------|
| `app/routes/app.tsx` | 导航栏新增 `<s-link href="/app/seo-bulk">🔍 SEO 批量编辑</s-link>` |

### 访问范围（Scopes）

当前 `shopify.app.toml` 已有 `write_products`，足够支持产品 SEO 字段的读写，无需新增 scope。

## 6. 界面状态

| 状态 | 处理 |
|------|------|
| Loading | 骨架屏（Skeleton），提示"正在加载产品数据..." |
| 空状态 | 商店无产品时显示"暂无产品，请先在 Shopify 后台创建产品" |
| 保存成功 | `shopify.toast.show("已成功更新 N 个产品的 SEO 信息")` |
| 保存失败 | 汇总失败项显示 + "重试失败项" 按钮 |

## 7. 错误处理

| 场景 | 处理方式 |
|------|---------|
| 字段长度超限 | 实时显示红色警告和字符计数，不阻止保存 |
| GraphQL 错误 | 逐产品报告失败原因，保存后汇总展示 |
| Handle 冲突 | 前端在校验时检测，保存时如 Shopify 返回错误则单独标记 |
| 网络超时 | 显示超时错误提示，已保存的产品不受影响 |

## 8. 测试策略

- **单元测试**：编辑器状态管理（modified 标记、字符计数、预览生成）
- **集成测试**：loader 数据加载、action 批量保存逻辑
- **手动测试**：在测试商店中真实安装 App，验证编辑和保存流程

## 9. 未来扩展

- 支持更多资源类型（产品系列、页面、博客文章）
- 支持批量模板应用（如 "所有产品 Meta Title = {product.title} | Store Name"）
- 支持 CSV 导入/导出 SEO 数据
- 支持 AI 自动生成 SEO 元数据
