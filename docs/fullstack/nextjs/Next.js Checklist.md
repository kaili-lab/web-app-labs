> **定位**：全栈岗位前端加分项，侧重"理解架构决策和渲染模型"，能做合理技术选型。  
**当前水平**：用过 App Router，了解 RSC 等新特性，但体系不够完整。  
**学习周期**：建议 3-4 周完成。  
**版本覆盖**：Next.js 15 基础 + Next.js 16 全部新特性 + Next.js 16.1  
**前置依赖**：React checklist 中的虚拟 DOM、Hooks、React 19 Actions、RSC 等知识。
>

---

## 核心原理（先建立全局视角）
Next.js 所有知识点背后有 5 个核心原理：

| # | 核心原理 | 一句话描述 | 典型体现 |
| --- | --- | --- | --- |
| 1 | **渲染位置是一种选择** | 同样的组件，可以选择在服务端、客户端、构建时渲染，各有权衡 | SSR、SSG、ISR、RSC、Cache Components |
| 2 | **服务端优先，客户端按需** | 默认在服务端完成尽可能多的工作，只有需要交互的部分才发到客户端 | Server Components 默认、'use client' 边界 |
| 3 | **约定优于配置** | 文件系统即路由，特殊文件名即功能 | App Router 文件约定、proxy.ts |
| 4 | **渐进式增强** | 页面先可见（HTML），再可交互（JS hydration），逐步增强 | Streaming SSR、Selective Hydration、Suspense |
| 5 | **缓存是显式的、可组合的** | 开发者明确声明什么需要缓存，而不是框架隐式缓存一切 | Next.js 16 Cache Components、'use cache' 指令 |


---

## 第一层：底层原理（地基）
> 理解渲染模型是理解 Next.js 一切设计决策的基础。
>

### 1.1 渲染模式全景 ⭐⭐⭐
**本质**：Web 应用的渲染本质上是"把数据变成 HTML"的过程。区别在于这个过程发生在哪里、什么时候。

| 渲染模式 | 在哪里渲染 | 什么时候渲染 | HTML 到达浏览器时 | 适用场景 | JS 开销 |
| --- | --- | --- | --- | --- | --- |
| CSR（纯客户端） | 浏览器 | 用户请求时 | 空壳，需要 JS 执行后才有内容 | 纯后台管理系统 | 最大 |
| SSR | 服务器 | 每次请求时 | 完整 HTML，需要 hydration | 个性化内容、实时数据 | 大（全量 hydration） |
| SSG | 服务器 | 构建时 | 完整 HTML，直接可用 | 博客、文档、营销页 | 小 |
| ISR | 服务器 | 构建时 + 定期重新生成 | 完整 HTML，后台按需更新 | 电商商品页、新闻 | 小 |
| Streaming SSR | 服务器 | 请求时，分块发送 | 渐进式接收 HTML | 首屏有慢数据源 | 大（但感知更快） |
| **PPR + Cache Components** | 服务器 | 静态部分构建时 + 动态部分请求时 | 静态 shell 立即到达 + 动态部分流式补充 | **Next.js 16 推荐模式** | **最优（按需）** |


**为什么有这么多模式**：因为没有银弹——每种模式在首屏速度、SEO、交互性、服务器成本之间做不同的取舍。Next.js 的价值在于让你在同一个应用中按页面/按组件选择最合适的模式。

**Next.js 16 的范式转变**：

+ Next.js 15 及之前：必须在页面级别选择 static 或 dynamic
+ Next.js 16 Cache Components：一个页面内可以混合 static + cached + dynamic，用 `"use cache"` 指令显式声明缓存
+ 不再需要 `getServerSideProps` / `getStaticProps` / `experimental.ppr`

**关联**：

+ 向上连接核心原理 1（渲染位置是一种选择）
+ SSR 和 CSR 的 hydration 问题 → 引出 RSC 的动机
+ Cache Components = PPR 的完整实现 → 引出 Next.js 16 的缓存模型
+ 与 Java 后端的类比：SSR 类似 Thymeleaf/JSP 的服务端模板渲染，CSR 类似前后端分离

**过关标准**：

- [ ] 能画出 CSR、SSR、SSG、ISR 的请求流程图
- [ ] 能说出每种模式的优劣势和适用场景
- [ ] 理解 hydration 的含义
- [ ] 能解释 PPR + Cache Components 如何打破"整个页面要么 static 要么 dynamic"的限制
- [ ] 能用一句话总结 Next.js 从 12 到 16 的渲染演进

### 1.2 React Server Components 在 Next.js 中的实现 ⭐⭐⭐
**本质**：Next.js App Router 默认所有组件都是 Server Component。RSC 在 Next.js 中不只是一个组件类型，而是整个架构的基础——路由、数据获取、缓存都建立在 RSC 之上。

**Server Component vs Client Component 的决策树**：

```plain
需要 useState/useEffect/useRef？ → Client Component
需要浏览器 API（window/document）？ → Client Component
需要事件监听器（onClick 等）？ → Client Component
只需要展示数据、不需要交互？ → Server Component
需要直接访问数据库/API？ → Server Component
有大型依赖（markdown 解析器等）？ → Server Component
```

**'use client' 边界的设计原则**：

+ 把 'use client' 尽量推到组件树的叶子节点
+ 一个常见的错误：在很高的层级加 'use client'，导致整个子树都变成 Client Component
+ 正确做法：Server Component 作为容器获取数据，通过 children/props 传给 Client Component
+ 示例：

```plain
// ❌ 整个页面变成 Client Component
'use client'
export default function Page() { ... }

// ✅ 只有交互部分是 Client Component
// page.tsx (Server Component)
export default async function Page() {
  const data = await fetchData()
  return <InteractiveWidget data={data} /> // 'use client' 只在这里
}
```

**关联**：

+ 向上连接核心原理 2（服务端优先，客户端按需）
+ 数据获取策略建立在 RSC 之上
+ 'use client' 边界直接影响客户端 JS bundle 大小
+ 与 React 19 的 RSC 稳定化对齐

**过关标准**：

- [ ] 能画出 Server Component 和 Client Component 的决策树
- [ ] 能解释 'use client' 边界的设计原则（推到叶子节点）
- [ ] 知道 Server Component 可以 import Client Component，反过来不行（但可以通过 children 传递）
- [ ] 能给出一个把 'use client' 从高层推到叶子的重构示例

### 1.3 Hydration 与 Streaming ⭐⭐⭐
**本质**：

+ **Hydration**：服务端生成的 HTML 是"死的"（没有事件绑定），客户端 React 需要重新遍历组件树，将事件监听器和交互逻辑绑定到已有 DOM 上，使其"活"过来
+ **Streaming SSR**：不是等整个页面渲染完再发送，而是先发送已完成的部分，慢的部分后续通过流式传输补上
+ **Selective Hydration**：React 可以优先 hydrate 用户正在交互的部分

**传统 SSR 的瀑布问题**：

```plain
获取数据A ─────────┐
获取数据B ──────────────────┐
获取数据C ─────────────────────────┐
                                    │ 渲染 HTML
                                    │ 发送完整 HTML
                                    │ 下载全量 JS
                                    │ Hydration
                                    ▼ 可交互
```

必须等所有数据都获取完才能开始，TTFB 取决于最慢的数据源。

**Streaming SSR 的解决方案**：

```plain
发送 Shell HTML（导航、布局）────────> 立即显示
获取数据A 完成 → 流式发送 ChunkA ──> 替换 loading
获取数据C 完成 → 流式发送 ChunkC ──> 替换 loading
获取数据B 完成 → 流式发送 ChunkB ──> 替换 loading（最慢的最后到）
Selective Hydration ─────────────> 用户交互部分优先 hydrate
```

**Hydration Mismatch 的常见原因和解决**：

| 原因 | 示例 | 解决方案 |
| --- | --- | --- |
| 使用时间相关 API | `new Date()`, `Date.now()` | 用 `useEffect` 在客户端设置时间 |
| 使用浏览器 API | `window.innerWidth` | 用 `useEffect` 或 `typeof window` 检查 |
| 使用随机数 | `Math.random()` | 使用 `useId` 或在 `useEffect` 中生成 |
| 条件渲染基于客户端状态 | `localStorage.getItem()` | 用 `useEffect` + useState 延迟渲染 |
| 第三方库注入不同内容 | 某些 CSS-in-JS 库 | 确保服务端和客户端配置一致 |


**关联**：

+ 向上连接核心原理 4（渐进式增强）
+ Streaming 依赖 React Suspense（Suspense 边界就是 Streaming 的分割点）
+ Next.js 中 `loading.tsx` 底层就是 Suspense
+ Cache Components 与 Streaming 配合：静态 shell 立即发送，动态部分流式补充

**过关标准**：

- [ ] 能解释 hydration 不匹配的原因和常见场景
- [ ] 能画出 Streaming SSR 的流程
- [ ] 理解 Suspense 在 Streaming 中的角色（分割点）
- [ ] 知道 Selective Hydration 的含义（优先 hydrate 用户交互的部分）
- [ ] 能给出 Hydration Mismatch 的 3 种以上排查方案

---

## 第二层：核心机制（工具箱）
> App Router 的日常开发核心机制，面试高频考点。
>

### 2.1 App Router 文件系统路由 ⭐⭐⭐
**本质**：`app/` 目录下的文件夹结构即路由结构。每个路由段由文件夹定义，特殊文件名承担不同职责。

**特殊文件约定**：

| 文件 | 职责 | 本质 | 注意事项 |
| --- | --- | --- | --- |
| `page.tsx` | 路由的 UI，有它该路由才可访问 | 入口组件 | 必须是默认导出 |
| `layout.tsx` | 包裹当前路由及子路由的共享 UI | 不重新渲染的外壳 | 导航时保持状态，不触发重新挂载 |
| `loading.tsx` | 加载状态 UI | 自动包裹在 Suspense 中 | 即 Streaming 的分割点 |
| `error.tsx` | 错误边界 UI | 自动包裹在 ErrorBoundary 中 | 必须是 Client Component |
| `not-found.tsx` | 404 UI | 调用 `notFound()` 时渲染 | - |
| `template.tsx` | 类似 layout 但导航时重新挂载 | 每次导航都创建新实例 | 用于需要重置状态的场景 |
| `route.ts` | API 路由端点 | Route Handler | 不能和 page.tsx 同级 |
| `default.tsx` | 平行路由的默认 UI | 平行路由的 fallback | 平行路由高级功能 |


**高级路由特性**：

| 特性 | 语法 | 用途 | 示例 |
| --- | --- | --- | --- |
| 路由分组 | `(group)` | 组织代码不影响 URL | `(marketing)/about/page.tsx` → `/about` |
| 动态路由 | `[param]` | URL 参数 | `[id]/page.tsx` → `/123` |
| Catch-all | `[...slug]` | 匹配多级路径 | `[...slug]/page.tsx` → `/a/b/c` |
| 平行路由 | `@slot` | 同一布局中渲染多个页面 | `@modal`、`@sidebar` |
| 拦截路由 | `(.)` | 拦截导航显示不同内容 | `(..)photo/[id]` 弹窗预览 |


**layout vs template 深入对比**：

+ layout：导航时不重新挂载 → 内部的 state 保持，useEffect 不重新执行。适用于导航栏、侧边栏
+ template：导航时重新挂载 → state 重置，useEffect 重新执行。适用于页面进入动画、需要每次重置的表单
+ Next.js 16 中 `<Activity>` 可以与 layout 配合，在隐藏路由时保持状态

**关联**：

+ 向上连接核心原理 3（约定优于配置）
+ layout 不重新渲染 → 这是通过 RSC 实现的，layout 是 Server Component
+ loading.tsx 底层就是 Suspense → 连接 Streaming
+ 与 Pages Router 的对比：Pages Router 基于 `pages/` 目录，没有 layout 嵌套、没有 RSC

**过关标准**：

- [ ] 能说出每个特殊文件的作用
- [ ] 能解释 layout 和 template 的区别
- [ ] 能说出路由分组的用法
- [ ] 能解释平行路由和拦截路由的使用场景（如 Instagram 风格的图片预览弹窗）
- [ ] 能说出 Next.js 16 中 params 变为异步 Promise 的变化

### 2.2 数据获取策略 ⭐⭐⭐
**本质**：Next.js App Router 中，数据获取发生在 Server Component 内，直接用 async/await，不需要 `getServerSideProps` 等特殊函数。

**Next.js 15 vs 16 的数据获取演进**：

| 版本 | 默认行为 | 控制缓存 |
| --- | --- | --- |
| Next.js 14 | fetch 默认缓存（等效 SSG） | `{ cache: 'no-store' }` 禁用 |
| Next.js 15 | fetch 默认不缓存（等效 SSR） | 显式设置缓存策略 |
| Next.js 16 | 所有数据获取默认动态执行 | 用 `"use cache"` 指令显式缓存 |


**Next.js 16 Cache Components 数据获取模式**：

```typescript
// 1. 默认行为：动态执行（每次请求时获取）
export default async function Page() {
  const data = await fetchData() // 每次请求都执行
  return <div>{data}</div>
}

// 2. 缓存整个页面
"use cache"
export default async function Page() {
  const data = await fetchData() // 构建时执行，结果缓存
  return <div>{data}</div>
}

// 3. 缓存单个组件
async function UserProfile({ userId }) {
  "use cache"
  const user = await fetchUser(userId) // 结果按 userId 缓存
  return <div>{user.name}</div>
}

// 4. 缓存单个函数
async function getProducts() {
  "use cache"
  cacheLife('hours')
  cacheTag('products')
  return await db.products.findAll()
}
```

**缓存控制 API（Next.js 16）**：

| API | 作用 | 使用场景 |
| --- | --- | --- |
| `cacheLife('hours')` | 设置缓存生命周期（内置 profile：max/days/hours/minutes/seconds） | 不同数据的刷新频率不同 |
| `cacheTag('products')` | 给缓存打标签，用于精确失效 | 数据变更后按标签失效 |
| `updateTag('products')` | **Server Action 专用**，立即失效并刷新（读你所写语义） | 用户修改数据后立即看到更新 |
| `revalidateTag('products', 'max')` | 标记缓存为 stale + SWR 行为 | 后台更新，允许短暂延迟 |
| `refresh()` | **Server Action 专用**，刷新非缓存数据 | 刷新实时指标等动态数据 |


**请求去重（Request Memoization）**：

+ 同一次渲染过程中，多个组件 fetch 同一个 URL，React 自动去重，只发一次请求
+ 只对 GET 请求生效
+ 作用域是单次渲染，跨请求不共享

**关联**：

+ 向上连接渲染模式（通过缓存策略切换 SSG/SSR/ISR）
+ 与 Server Actions 的分工：数据获取用 `"use cache"` + Server Component，数据变更用 Server Actions
+ Next.js 16 的缓存理念：从"默认缓存一切"到"默认动态，显式缓存"

**过关标准**：

- [ ] 能说出 Next.js 14 → 15 → 16 数据获取默认行为的变化
- [ ] 能用 `"use cache"` + `cacheLife` + `cacheTag` 写出一个完整的缓存示例
- [ ] 能区分 `updateTag` 和 `revalidateTag` 的区别（即时 vs SWR）
- [ ] 能解释请求去重（Request Memoization）的机制和限制
- [ ] 能说出为什么 Next.js 16 从"默认缓存"转向"默认动态"

### 2.3 Server Actions ⭐⭐⭐
**本质**：Server Actions 是在服务端执行的异步函数，可以直接在客户端组件中调用。用 `'use server'` 标记。Next.js 在编译时自动生成一个 API 端点，客户端调用时走 POST 请求。

**为什么需要它**：

+ 消除手动创建 API 路由的样板代码
+ 支持渐进式增强——没有 JS 的情况下表单也能提交
+ 和 React 19 的 Actions 系统深度集成

**深入细节**：

+ 安全性：Server Actions 的参数是用户输入，**必须做验证**（就像后端 Controller 接参数一样）

```typescript
'use server'
import { z } from 'zod'

const schema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
})

export async function createUser(formData: FormData) {
  const result = schema.safeParse(Object.fromEntries(formData))
  if (!result.success) return { error: result.error.flatten() }
  // ...
}
```

+ Server Actions 可以调用 `updateTag()` / `revalidateTag()` / `refresh()` 来更新缓存
+ Server Actions 与 React 19 的 `useActionState` / `useOptimistic` 配合使用

**与 Route Handlers 的分工**：

| 场景 | 推荐方案 | 原因 |
| --- | --- | --- |
| 表单提交、数据变更 | Server Actions | 更简洁，支持渐进式增强 |
| 第三方 webhook | Route Handlers | 需要处理原始请求 |
| 公开 API 给外部调用 | Route Handlers | 需要 RESTful 接口 |
| 文件上传 | Route Handlers | 需要处理 FormData 流 |
| 实时推送 | Route Handlers（SSE） | 需要长连接 |


**关联**：

+ 与 React 19 Actions 系统的关系：Server Action 就是一个 async function + 'use server'，可以作为 `<form action>` 的值
+ 与 Next.js 16 `updateTag` 的配合：Server Action 中调用 updateTag 实现"写后读"
+ 与后端 API 的类比：Server Actions ≈ RPC，Route Handlers ≈ REST

**过关标准**：

- [ ] 能说出 Server Actions 的工作原理
- [ ] 知道 Server Actions 中必须做输入验证
- [ ] 能区分 Server Actions 和 Route Handlers 的适用场景
- [ ] 能写一个 Server Action + useActionState + useOptimistic 的完整示例
- [ ] 知道 Server Actions 可以调用缓存失效 API

### 2.4 Next.js 16 缓存架构 ⭐⭐⭐
**本质**：Next.js 16 重构了整个缓存体系，核心理念是"默认动态，显式缓存"。

**缓存层级（从近到远）**：

| 缓存层 | 位置 | 作用 | Next.js 16 变化 |
| --- | --- | --- | --- |
| Request Memoization | 服务端（单次请求内） | 去重相同 fetch 请求 | 不变 |
| `"use cache"` | 服务端 | 缓存页面/组件/函数的输出 | **新增**，替代旧的 fetch 缓存和 PPR |
| Full Route Cache | 服务端 | 缓存构建时的静态路由 | 通过 Cache Components 控制 |
| Router Cache | 客户端内存 | 缓存已访问的路由段 | 增量预取 + 布局去重 |


**"use cache" 的三个层级**：

1. **文件级**：`"use cache"` 放在文件顶部 → 缓存该文件所有导出函数
2. **组件级**：在组件函数体内 → 缓存该组件的渲染输出
3. **函数级**：在普通函数体内 → 缓存函数返回值

**缓存 key 自动生成**：

+ 编译器自动根据 Build ID + 函数位置 + 输入参数生成缓存 key
+ 不需要手动指定缓存 key（与 Redis 不同）
+ 参数必须是可序列化的

**缓存失效策略对比**：

| 策略 | API | 行为 | 适用场景 |
| --- | --- | --- | --- |
| 按时间过期 | `cacheLife('hours')` | 到期后下次请求触发重新生成 | 新闻、商品列表 |
| 按标签失效（SWR） | `revalidateTag('tag', 'max')` | 标记为 stale，后台重新生成 | CMS 内容更新 |
| 按标签失效（即时） | `updateTag('tag')` | 立即失效并刷新，用户看到最新数据 | 用户操作后的反馈 |
| 按路径失效 | `revalidatePath('/products')` | 失效整个路径的缓存 | 简单场景 |


**"use cache" 的限制**：

+ 不能在缓存函数内部直接访问 `cookies()`、`headers()`、`searchParams` → 必须在外部读取后作为参数传入
+ 传入的参数必须可序列化（不能传函数、DOM 节点等）
+ `"use cache"` 运行在隔离环境中，无法访问运行时上下文

**关联**：

+ 与后端缓存的类比：`"use cache"` ≈ `@Cacheable`、`cacheTag` ≈ 缓存 key、`updateTag` ≈ 缓存失效
+ 向上连接核心原理 5（缓存是显式的、可组合的）
+ 与 React 19.2 PPR 的关系：Cache Components 是 PPR 的上层抽象

**过关标准**：

- [ ] 能说出 Next.js 16 "默认动态，显式缓存"的理念
- [ ] 能说出 `"use cache"` 的三个层级
- [ ] 能区分 `updateTag` 和 `revalidateTag` 并说出各自场景
- [ ] 能解释缓存 key 的自动生成机制
- [ ] 能说出 `"use cache"` 的限制（不能访问运行时 API）
- [ ] 能对比 Next.js 14/15/16 的缓存行为差异

### 2.5 proxy.ts（原 middleware.ts） ⭐⭐
**本质**：Next.js 16 将 `middleware.ts` 重命名为 `proxy.ts`，明确其职责——在请求到达路由之前做网络层拦截。运行在 Node.js 运行时（不再是 Edge）。

**变化对比**：

|  | middleware.ts（Next.js 15） | proxy.ts（Next.js 16） |
| --- | --- | --- |
| 运行时 | Edge Runtime（受限的 V8） | **Node.js Runtime（完整 Node API）** |
| 导出函数名 | `middleware` | `proxy` |
| 定位 | 模糊——可能是服务器中间件、也可能是 Edge 函数 | 明确——网络代理层 |
| 状态 | 可用（16 中已废弃） | 推荐 |


**典型用途**：认证检查、国际化路由重定向、A/B 测试、地理位置重定向。

**设计原则**：

+ proxy.ts 应该轻量——只做路由级决策（重定向、重写、头部修改）
+ 业务逻辑放在 Server Component 和 Route Handler 中
+ Node.js Runtime 的好处：可以使用完整的 Node.js API 和 npm 包，调试更方便

**关联**：

+ 与 Java 的 Filter/Interceptor 类比：都是请求到达业务逻辑之前的拦截层
+ 与 Route Handlers 的区别：proxy 是全局的跨路由拦截，Route Handlers 是具体的端点
+ 迁移简单：只需重命名文件和导出函数名

**过关标准**：

- [ ] 知道 proxy.ts 的由来和与 middleware.ts 的区别
- [ ] 能说出 Node.js Runtime vs Edge Runtime 的差异
- [ ] 能说出 2-3 个常见使用场景
- [ ] 知道 proxy.ts 应该保持轻量的设计原则

### 2.6 Next.js 16 路由增强 ⭐⭐
**布局去重（Layout Deduplication）**：

+ 问题：页面有 50 个 `<Link>` 指向不同产品页，旧版每个 Link 的预取都包含共享 layout 数据 → 50 次重复下载
+ 解决：Next.js 16 自动识别共享 layout，只下载一次
+ 影响：预取的总传输量大幅减少

**增量预取（Incremental Prefetching）**：

+ 只预取缓存中还没有的部分（而非整个页面）
+ 预取生命周期管理：Link 离开视口时取消预取、悬停时优先预取、数据失效后重新预取
+ 权衡：请求数量可能增加，但总传输量减少

**过关标准**：

- [ ] 能说出布局去重和增量预取解决的问题
- [ ] 理解这些优化是自动的，不需要代码修改

---

## 第三层：实战应用（场景）
> 面试中以"你项目里怎么做的"形式出现。
>

### 3.1 项目架构与目录组织 ⭐⭐⭐
**Next.js 16 推荐目录结构**：

```plain
app/
├── (marketing)/              # 路由分组：营销页（独立 layout）
│   ├── layout.tsx
│   ├── page.tsx              # 首页
│   └── about/page.tsx
├── (dashboard)/              # 路由分组：后台（独立 layout + 认证）
│   ├── layout.tsx            # 后台布局（侧边栏、顶栏）
│   ├── page.tsx              # 仪表盘首页
│   ├── settings/page.tsx
│   └── @modal/(.)edit/page.tsx  # 拦截路由：编辑弹窗
├── api/                      # Route Handlers
│   └── webhook/route.ts
├── layout.tsx                # 根布局
├── not-found.tsx
└── error.tsx
src/
├── components/
│   ├── ui/                   # 基础 UI 组件（Button、Input 等）
│   └── features/             # 业务组件（UserCard、ProductList 等）
├── lib/                      # 工具函数、配置、数据库客户端
├── actions/                  # Server Actions（按领域拆分）
│   ├── user.ts
│   └── product.ts
├── types/                    # TypeScript 类型
└── hooks/                    # 自定义 Hooks（客户端用）
proxy.ts                      # 网络代理（原 middleware.ts）
```

**Server/Client 组件边界设计**：

+ 原则：尽量把 'use client' 推到叶子节点
+ 数据获取：在最需要数据的 Server Component 中获取，利用请求去重
+ 交互逻辑：抽到独立的 Client Component，通过 props 接收 Server Component 的数据
+ 示例：产品详情页 → Server Component 获取数据 + Client Component 处理加入购物车按钮

**过关标准**：

- [ ] 能说出路由分组的好处（共享布局、组织代码、认证分区）
- [ ] 能解释"把 'use client' 推到叶子节点"的原因和做法
- [ ] 能说出 Server Actions 的组织方式（按领域拆分到 actions/ 目录）

### 3.2 认证与授权 ⭐⭐⭐
**多层认证架构**：

| 层 | 职责 | 工具 |
| --- | --- | --- |
| proxy.ts | 路由级保护——未登录重定向到登录页 | 检查 cookie/token |
| layout.tsx | 展示级控制——根据用户角色显示不同 UI | 读取 session |
| Server Component | 数据级控制——只返回该用户有权看的数据 | 查询时过滤 |
| Server Actions | 操作级控制——验证用户有权执行该操作 | 检查权限 |


**常见认证方案**：

| 方案 | 特点 | 适用场景 |
| --- | --- | --- |
| NextAuth.js (Auth.js) | 开箱即用，支持多 Provider | 中小型项目 |
| Clerk | 托管式认证，UI 组件丰富 | 快速集成 |
| 自建 JWT + Cookie | 完全控制，灵活 | 企业项目、有特殊需求 |
| Lucia Auth | 轻量、Session-based | 需要控制但不想太复杂 |


**过关标准**：

- [ ] 能说出认证逻辑在 Next.js 各层的职责
- [ ] 了解至少一个认证方案的基本用法
- [ ] 知道 Server Actions 中必须验证权限（不能假设前端已做过认证）

### 3.3 Next.js 性能优化 ⭐⭐⭐
**优化手段全景**：

| 手段 | 解决什么问题 | Next.js 提供的能力 | Next.js 16 变化 |
| --- | --- | --- | --- |
| 图片优化 | 图片太大、格式不优 | `<Image>` 组件：自动 WebP/AVIF、懒加载、尺寸优化 | 更安全的默认配置 |
| 字体优化 | 字体闪烁（FOUT） | `next/font`：构建时内联字体 | - |
| 代码分割 | JS bundle 太大 | 路由级自动分割 + `dynamic()` 组件级分割 | Turbopack 更快 |
| 静态优先 | 不必要的服务端计算 | `"use cache"` 显式缓存 | Cache Components 替代旧的 SSG/ISR |
| Streaming | 首屏被慢数据源阻塞 | Suspense + loading.tsx | 与 Cache Components 配合 |
| 减少客户端 JS | Client Component 太多 | 审视 'use client' 边界 | React Compiler 自动优化 |
| 构建速度 | 构建太慢 | Turbopack | **默认启用，2-5x 更快** |
| 开发体验 | 冷启动慢 | Turbopack 文件系统缓存 | **16.1 稳定** |


**Turbopack（Next.js 16 默认构建工具）**：

+ 替代 Webpack，基于 Rust 编写
+ 开发模式：最高 10x 更快的 Fast Refresh
+ 生产构建：2-5x 更快
+ Next.js 16 中已是默认构建工具，不需要配置
+ 如果有自定义 webpack 配置，可以回退：`next build --webpack`

**React Compiler 集成（Next.js 16 稳定）**：

```typescript
// next.config.ts
const nextConfig = {
  reactCompiler: true, // 从 experimental 提升为稳定配置
}
```

+ 自动优化组件渲染，减少不必要的 re-render
+ 需要安装 `babel-plugin-react-compiler`
+ 注意：启用 Compiler 会增加构建时间（依赖 Babel）

**与 Core Web Vitals 的对应**：

+ **LCP**（最大内容绘制）：图片优化、Streaming、Cache Components
+ **CLS**（累积布局偏移）：`next/font`、`<Image>` 的 width/height
+ **INP**（交互到下一次绘制）：减少客户端 JS、React Compiler、useTransition

**过关标准**：

- [ ] 能说出 `<Image>` 组件相比原生 `<img>` 的 5 个优势
- [ ] 能解释 `next/font` 如何消除字体闪烁
- [ ] 能说出 Turbopack 的定位和性能数据
- [ ] 知道如何在 Next.js 16 中启用 React Compiler
- [ ] 能将优化手段与 Core Web Vitals 指标对应

### 3.4 API 设计与 Next.js DevTools MCP ⭐⭐
**Route Handlers vs Server Actions 选择**：

| 场景 | 推荐 | 原因 |
| --- | --- | --- |
| 表单提交、数据变更 | Server Actions | 简洁，支持渐进式增强 |
| 第三方 webhook | Route Handlers | 需要处理原始请求 |
| 公开 API | Route Handlers | 需要 RESTful 接口 |
| 文件上传 | Route Handlers | 需要处理流 |
| AI 集成 | Route Handlers（SSE） | 流式响应 |


**Next.js DevTools MCP（Next.js 16 新增）**：

+ 是什么：Model Context Protocol 集成，让 AI 编程工具理解你的 Next.js 应用结构
+ 功能：路由感知、统一日志（浏览器+服务器）、自动错误访问、页面上下文
+ Next.js 16.1 增加了 `get_routes` 能力，AI 可以映射整个应用结构
+ 适用于 Cursor、Claude Code 等 AI 编程工具

**过关标准**：

- [ ] 能根据场景说出用 Server Actions 还是 Route Handlers
- [ ] 能写一个简单的 Route Handler（GET + POST）
- [ ] 了解 DevTools MCP 的用途

---

## 第四层：高阶能力（诊断/设计）
> 全栈岗位加分项，能答上来说明对 Next.js 理解深入。
>

### 4.1 部署与运行时选择 ⭐⭐
**部署选项**：

| 部署方式 | 支持的特性 | 适用场景 | 注意事项 |
| --- | --- | --- | --- |
| Vercel | 全特性 | 最省心 | 成本随流量增长 |
| Node.js 自托管 | 大部分特性 | 私有部署 | 需要自己管理缓存 |
| Docker 容器 | 大部分特性 | K8s 环境 | 需要配置 `output: 'standalone'` |
| 静态导出 | 仅 SSG | 纯静态站 | 不支持 SSR、Server Actions、proxy 等 |


**Next.js 16 Build Adapters API（alpha）**：

+ 允许部署平台创建自定义适配器，hook 进构建过程
+ 目标：让 Cloudflare、AWS 等平台更方便地支持 Next.js

**Next.js 16 版本要求**：

+ Node.js 20.9+（18 不再支持）
+ TypeScript 5.1+
+ 浏览器：Chrome 111+、Edge 111+、Firefox 111+、Safari 16.4+

**过关标准**：

- [ ] 能说出 Vercel 和自托管的区别
- [ ] 知道静态导出的限制
- [ ] 知道 Next.js 16 的最低版本要求

### 4.2 Next.js 版本演进与技术选型 ⭐⭐
**Next.js 12 → 16 核心演进线**：

| 版本 | 核心变化 | 渲染模型变化 |
| --- | --- | --- |
| 12 | Middleware、SWC 编译器 | SSR/SSG/ISR |
| 13 | App Router（beta）、RSC | 引入 Server/Client Component |
| 14 | App Router 稳定、Server Actions | 默认缓存一切 |
| 15 | fetch 默认不缓存、async params | 从默认缓存转向 |
| **16** | Cache Components、Turbopack 默认、proxy.ts | **默认动态 + 显式缓存** |


**与其他方案的对比**：

| 方案 | 定位 | 选 Next.js 的理由 | 选对方的理由 |
| --- | --- | --- | --- |
| Vite + React | 纯 CSR SPA | 需要 SEO、SSR、全栈 | 纯后台系统 |
| Remix | 全栈 React 框架 | 社区生态更大、Vercel 支持 | 更偏好 Web 标准 |
| Astro | 内容为主的静态站 | 需要复杂交互 | 内容/文档站 |


**过关标准**：

- [ ] 能说出 Next.js 12-16 的核心演进线
- [ ] 能根据项目需求做出合理的框架选型

### 4.3 常见问题与排查 ⭐⭐
**高频问题排查手册**：

| 问题 | 原因 | 排查方向 | Next.js 16 变化 |
| --- | --- | --- | --- |
| Hydration Mismatch | 服务端/客户端渲染不一致 | 检查 Date.now()、window、随机数 | 不变 |
| 数据没更新 | 缓存未失效 | 检查 `"use cache"` + cacheLife + updateTag/revalidateTag | **新的缓存体系** |
| Server Component 中用了 Hooks | SC 不支持 useState 等 | 加 'use client' 或拆分交互部分 | 不变 |
| 打包体积大 | 'use client' 边界太高 | `@next/bundle-analyzer` 分析 | React Compiler 帮助减少 memo 代码 |
| 构建太慢 | Webpack 瓶颈 | 确认使用 Turbopack | **16 默认 Turbopack** |
| proxy.ts 不生效 | 还在用 middleware.ts | 重命名为 proxy.ts + 导出 proxy 函数 | **16 新增** |
| `"use cache"` 内访问 cookies() | 缓存函数不能访问运行时 API | 在外部读取后作为参数传入 | **16 新增** |


**Next.js 16 的 Breaking Changes 清单**：

+ Node.js 18 不再支持，最低 20.9
+ `middleware.ts` 废弃，改用 `proxy.ts`
+ `next lint` 命令移除，改用 ESLint 或 Biome 直接运行
+ AMP 支持完全移除
+ `experimental.ppr` 移除，改用 `cacheComponents: true`
+ `serverRuntimeConfig` / `publicRuntimeConfig` 移除，改用环境变量
+ `next/image` 默认配置更安全

**过关标准**：

- [ ] 遇到 Hydration Mismatch 知道怎么排查
- [ ] 遇到缓存问题知道从 `"use cache"` / cacheLife / updateTag 排查
- [ ] 知道 Next.js 16 的主要 Breaking Changes

---

## 面试场景题
### 基础追问
1. "Next.js 的 SSR、SSG、ISR 分别是什么？Next.js 16 的 Cache Components 如何统一了它们？"
2. "Server Component 和 Client Component 有什么区别？什么时候用 'use client'？"
3. "App Router 和 Pages Router 有什么区别？为什么要迁移？"
4. "Next.js 的 `<Image>` 组件做了什么优化？"
5. "Next.js 16 的 proxy.ts 是什么？和 middleware.ts 有什么区别？"

### 原理追问
6. "解释一下 RSC 和传统 SSR 的区别？RSC 解决了什么问题？"
7. "Next.js 16 的缓存模型是怎样的？`'use cache'` 是什么？和 Next.js 15 的 fetch 缓存有什么区别？"
8. "什么是 Hydration Mismatch？举个例子，怎么解决？"
9. "`updateTag` 和 `revalidateTag` 有什么区别？分别在什么场景使用？"
10. "Turbopack 是什么？为什么要替换 Webpack？"

### 场景设计
11. "你要做一个电商网站，首页、商品列表页、商品详情页、购物车页分别用什么渲染策略？"
    - 期望回答（Next.js 16）：
    - 首页：`"use cache"` + `cacheLife('hours')` 缓存静态部分 + Suspense 包裹动态推荐
    - 列表页：`"use cache"` + `cacheTag('products')` + 分类/筛选用 Client Component
    - 详情页：`"use cache"` + `cacheTag('product-${id}')` + 评论区 Streaming
    - 购物车：纯动态（不缓存），Server Component 读 session + Client Component 处理交互
12. "你的 Next.js 应用打包后客户端 JS 太大，怎么优化？"
    - 期望回答：分析 'use client' 边界 + dynamic import + 审查第三方库 + 启用 React Compiler
13. "你的项目需要用户登录后才能访问某些页面，怎么设计？"
    - 期望回答：proxy.ts 做路由保护 + layout 中读 session 控制 UI + Server Actions 验证权限
14. "用户修改了数据，但页面显示的还是旧数据，你怎么排查？"
    - 期望回答：检查是否用了 `"use cache"` → 检查是否调用了 `updateTag` / `revalidateTag` → 检查 Router Cache → 检查 cacheLife 配置
15. "如果你要从 Next.js 15 迁移到 16，需要注意什么？"
    - 期望回答：Node.js 版本 → middleware.ts 改 proxy.ts → 缓存策略迁移到 Cache Components → 运行 codemod → Turbopack 兼容性检查

---

## 实践计划
| 方式 | 具体做法 | 覆盖知识点 | 预计时间 |
| --- | --- | --- | --- |
| 本地实验 | 在同一个页面用 `"use cache"` 创建静态 + 缓存 + 动态三种内容 | Cache Components、Streaming、cacheLife | 3h |
| 本地实验 | 写一个 Server Action + useActionState + updateTag 的完整数据变更流 | Server Actions、缓存失效、乐观更新 | 4h |
| 本地实验 | 故意制造 Hydration Mismatch，用 DevTools 排查 | Hydration、Server/Client 边界 | 2h |
| 本地实验 | 用 `@next/bundle-analyzer` 分析打包体积，对比 'use client' 边界调整前后 | 性能优化、组件边界设计 | 2h |
| 本地实验 | 将 middleware.ts 迁移为 proxy.ts，测试认证流程 | proxy.ts、认证、迁移 | 1h |
| 项目中实践 | 在个人全栈项目中实现完整认证流（proxy + Server Component + Server Actions） | 认证、多层安全、缓存 | 持续 |
| Side Project | 做一个带 Cache Components 的博客（静态文章 + 动态评论 + 实时浏览量） | Cache Components 全链路、Streaming、updateTag | 1 周 |


---

## 与 React Checklist 的关联
| Next.js 知识点 | 依赖的 React 知识点 | 双向连接 |
| --- | --- | --- |
| RSC / Server Component | 虚拟 DOM、组件模型 | RSC Payload 是虚拟 DOM 的序列化形式 |
| Streaming SSR | Suspense | Suspense 边界 = Streaming 分割点 |
| 'use client' 边界设计 | 组件化、Hooks 体系 | 哪些 Hooks 只能在 Client Component 用 |
| Cache Components | 状态更新与重渲染 | 缓存失效触发重渲染 |
| Server Actions | React 19 Actions 系统 | useActionState + useFormStatus + useOptimistic |
| 性能优化 | React.memo、React Compiler | Next.js 16 内置 React Compiler 支持 |
| proxy.ts | 事件系统、请求拦截 | 类比 Java Filter/Interceptor |
| Activity (React 19.2) | 状态保持、条件渲染 | Tab 切换保持状态场景 |


---

## Next.js 16 迁移 Checklist
如果你正在从 Next.js 15 迁移到 16，以下是关键步骤：

- [ ] 升级 Node.js 到 20.9+
- [ ] 升级 TypeScript 到 5.1+
- [ ] 运行 `npx @next/codemod@canary upgrade latest`
- [ ] 将 `middleware.ts` 重命名为 `proxy.ts`，导出函数名改为 `proxy`
- [ ] 将 `experimental.ppr` 配置替换为 `cacheComponents: true`
- [ ] 将 `next lint` 脚本替换为直接运行 ESLint 或 Biome
- [ ] 检查 `next/image` 配置——默认值可能变化
- [ ] 移除 AMP 相关代码（如有）
- [ ] 将 `serverRuntimeConfig` / `publicRuntimeConfig` 迁移到 `.env` 文件
- [ ] 确认 Turbopack 兼容性（如有自定义 webpack 配置可用 `--webpack` 回退）
- [ ] （可选）启用 React Compiler：`reactCompiler: true`
- [ ] （可选）逐步将数据获取迁移到 `"use cache"` 模式

