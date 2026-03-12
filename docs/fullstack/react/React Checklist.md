> **定位**：全栈岗位前端加分项，侧重"能讲清原理 + 能做合理技术选型"。  
**当前水平**：用过 Hooks 写过完整项目，但原理不太清楚。  
**学习周期**：建议 3-4 周完成。  
**版本覆盖**：React 18 基础 + React 19 全部新特性 + React 19.2（Activity、useEffectEvent 等）
>

---

## 核心原理（先建立全局视角）
React 所有知识点背后有 5 个核心原理，理解了它们，下面的知识点大多可以推导：

| # | 核心原理 | 一句话描述 | 典型体现 |
| --- | --- | --- | --- |
| 1 | **声明式 UI = f(state)** | 你描述"要什么"，React 负责"怎么做" | JSX、状态驱动渲染、React vs jQuery |
| 2 | **用 JS 计算代替 DOM 操作** | 虚拟 DOM 的本质是把昂贵的 DOM 操作变成便宜的 JS 对比 | 虚拟 DOM、Diff 算法、批量更新 |
| 3 | **组件化 = 隔离 + 组合** | 独立封装状态和 UI，通过 props 组合出复杂界面 | 组件树、Props 单向数据流、组合优于继承 |
| 4 | **调度与优先级** | 不是所有更新都一样急，React 可以中断低优先级工作 | Fiber 架构、并发模式、时间切片 |
| 5 | **服务端与客户端的边界是可移动的** | 组件的渲染位置（服务端/客户端）是一种架构选择，而不是固定的 | RSC、'use client'、'use server'、Streaming |


---

## 第一层：底层原理（地基）
> 理解了这层，上面的机制和 API 都能解释"为什么这样设计"。
>

### 1.1 虚拟 DOM 与 Reconciliation ⭐⭐⭐
**本质**：虚拟 DOM 是一棵 JS 对象树，描述"UI 应该长什么样"。每次状态变化，React 生成新的虚拟 DOM 树，和旧树对比（Diff），算出最小的 DOM 操作集合，批量应用到真实 DOM。

**为什么需要它**：直接操作 DOM 代价高（触发回流/重绘），且手动管理 DOM 更新容易出错。虚拟 DOM 让开发者只关心状态，React 自动计算最优更新路径。

**深入细节**：

+ 虚拟 DOM 节点就是一个普通 JS 对象：`{ type: 'div', props: { children: [...] }, key: null }`
+ 每次 `setState` 后，React 并不立即操作 DOM，而是先在内存中构建新的虚拟 DOM 树
+ Diff 完成后，React 把所有 DOM 变更打包成一个"Effect List"，在 Commit 阶段一次性执行
+ 虚拟 DOM 的另一个重要价值是**跨平台**——同一套虚拟 DOM 可以渲染到 DOM（react-dom）、Native（react-native）、Canvas 等

**关联**：

+ 向上连接核心原理 2（用 JS 计算代替 DOM 操作）
+ 向下支撑 Diff 算法、批量更新、性能优化（shouldComponentUpdate / memo）
+ 与 Fiber 关联：Fiber 是 Reconciliation 的具体实现架构
+ 与 RSC 的关联：Server Component 输出的不是 HTML 而是序列化的虚拟 DOM 描述（RSC Payload）

**过关标准**：

- [ ] 能画出"状态变化 → 新虚拟 DOM → Diff → Effect List → 批量 DOM 操作"的完整流程
- [ ] 能解释为什么虚拟 DOM 不一定比直接操作 DOM 快（答：多了一层 JS 计算开销，但它的价值在于可维护性和跨平台能力）
- [ ] 能说出 React Diff 的三个核心假设：同层比较、不同类型直接替换、key 标识同级节点
- [ ] 能解释虚拟 DOM 为什么能实现跨平台渲染

### 1.2 Fiber 架构 ⭐⭐⭐
**本质**：Fiber 是 React 的工作单元（unit of work）。React 16 之前的 Reconciliation 是递归的、不可中断的（Stack Reconciler）；Fiber 把渲染工作拆成一个个小任务，每个 Fiber 节点就是一个任务，可以暂停、恢复、丢弃。

**为什么需要它**：Stack Reconciler 一旦开始就必须走完整棵树，大组件树会阻塞主线程导致卡顿（比如一个 10000 项的列表更新会让页面冻结几百毫秒）。Fiber 让 React 可以在每个小任务之间检查"有没有更紧急的事"（如用户输入），实现了可中断渲染。

**深入细节**：

+ Fiber 节点的结构：`type`（组件类型）、`stateNode`（对应的 DOM 节点或组件实例）、`child/sibling/return` 三个指针形成链表
+ Fiber 树是双缓冲的：current tree（当前显示在屏幕上的）和 workInProgress tree（正在构建的）
+ 渲染分两个阶段：
    - **Render 阶段**（可中断）：遍历 Fiber 树，计算变化，纯计算无副作用。React 用 `requestIdleCallback` 的思路（实际自己实现了调度器）在浏览器空闲时执行
    - **Commit 阶段**（不可中断）：把计算好的变化应用到 DOM，执行副作用（useEffect、useLayoutEffect）
+ Hooks 的存储位置：每个 Fiber 节点有一个 `memoizedState` 属性，指向一个 Hooks 链表。useState、useEffect 等 Hook 的状态就存在这里
+ 优先级调度：React 内部有一个 Scheduler，为不同类型的更新分配优先级（如用户输入 > 网络响应 > 动画 > 数据获取）

**关联**：

+ 向上连接核心原理 4（调度与优先级）
+ 支撑 Concurrent Mode、useTransition、useDeferredValue
+ 每个 Fiber 节点保存了组件的 state 和 effect 链表（这就是 Hooks 的存储位置）
+ 双缓冲 + 可中断 = React 18/19 并发特性的基础
+ 与 Java 后端的类比：Fiber 的调度类似线程池 + 优先级队列

**过关标准**：

- [ ] 能解释 Stack Reconciler 的问题和 Fiber 如何解决
- [ ] 能说出 Fiber 的两个阶段：Render（可中断，纯计算）和 Commit（不可中断，操作 DOM）
- [ ] 知道 Fiber 节点的基本结构：type、stateNode、child/sibling/return 指针
- [ ] 能解释双缓冲（current tree vs workInProgress tree）的作用
- [ ] 能说出 Hooks 在 Fiber 上的存储方式（memoizedState 链表）
- [ ] 理解优先级调度的基本概念（不同更新类型有不同优先级）

### 1.3 JSX 与 React.createElement ⭐⭐
**本质**：JSX 是语法糖，编译后变成 `React.createElement(type, props, children)` 调用（React 17+ 改用 `jsx()` 函数，由编译器自动注入，不再需要手动导入 React）。返回值就是虚拟 DOM 节点（一个普通 JS 对象）。

**为什么需要它**：直接写 `createElement` 嵌套太深、可读性差。JSX 让你用类 HTML 的语法描述 UI，编译器帮你转换。

**深入细节**：

+ JSX 中的 `{}` 本质是 JS 表达式，所以不能写语句（if/for），但可以用三元表达式、&&、map 等
+ JSX 中的属性名用 camelCase（className、onClick）是因为它映射的是 DOM 的 JS API 而非 HTML 属性
+ React 17+ 的 New JSX Transform：`import { jsx } from 'react/jsx-runtime'` 由编译器自动注入，减小了 bundle 体积
+ 条件渲染的几种方式：三元、&&（注意 0 && 的陷阱）、提前 return、变量赋值
+ React 19 中 ref 可以作为普通 prop 传递，不再需要 forwardRef

**关联**：

+ 向上连接核心原理 1（声明式 UI）
+ 向下：JSX 的产物就是虚拟 DOM 节点，进入 Reconciliation 流程
+ React 19 变化：ref 作为 prop 直接传递，简化了 ref 转发

**过关标准**：

- [ ] 能手写一段 JSX 对应的 createElement 调用
- [ ] 理解 JSX 中 `{}` 的本质（JS 表达式），以及为什么不能写语句
- [ ] 知道 React 17+ New JSX Transform 的变化和好处
- [ ] 知道 React 19 中 ref 作为 prop 传递的变化

### 1.4 React Server Components (RSC) ⭐⭐⭐
**本质**：RSC 是 React 19 正式稳定的新组件类型——在服务端执行，永远不发到客户端。它的输出不是 HTML 而是一种序列化的组件树描述（RSC Payload），客户端 React 拿到这个描述后和客户端组件树合并渲染。

**与传统 SSR 的核心区别**：

+ 传统 SSR：服务端渲染完整 HTML → 客户端加载**全量** JS → hydration。JS bundle 包含所有组件代码
+ RSC：Server Component 的代码**永远不发到客户端**，JS bundle 只包含 Client Component 的代码。零客户端 JS 开销
+ 传统 SSR 每次请求都要重新渲染；RSC 的输出可以缓存和流式传输

**为什么需要它**：

+ 减小客户端 JS bundle——Server Component 的代码和依赖库（如 markdown 解析器、数据库 ORM）都不打包到客户端
+ 直接在服务端访问数据库/文件系统，不需要 API 层
+ 解决 SSR 的 hydration 成本问题——Server Component 不需要 hydration

**深入细节**：

+ 'use client' 指令的含义：不是"这个组件在客户端渲染"，而是"这是 Server/Client 边界，它及其导入的子组件需要包含在客户端 bundle 中"
+ 'use server' 指令的含义：标记一个函数为 Server Action，可以从客户端调用
+ Server Component 的限制：不能用 useState、useEffect 等有状态 Hooks，不能使用浏览器 API
+ Client Component 可以渲染 Server Component（通过 children prop 传入），但不能直接 import Server Component
+ 数据流方向：Server → Client 通过 props（可序列化数据），Client → Server 通过 Server Actions

**关联**：

+ 向上连接核心原理 5（服务端与客户端的边界是可移动的）
+ RSC 是 Next.js App Router 的基石——所有组件默认是 Server Component
+ 与 Streaming 关联：RSC Payload 可以流式传输
+ 与 Suspense 关联：Server Component 中可以用 async/await，配合 Suspense 实现 Streaming

**过关标准**：

- [ ] 能解释 Server Component 和 Client Component 的区别
- [ ] 能说出 Server Component 的限制
- [ ] 能准确解释 'use client' 指令的含义（边界声明，不是渲染位置）
- [ ] 能画出 Server Component 和 Client Component 的数据流动方式
- [ ] 能解释 RSC Payload 是什么，为什么不是 HTML
- [ ] 能说出 Client Component 如何使用 Server Component（children pattern）

---

## 第二层：核心机制（工具箱）
> 面试追问最密集的层，也是日常开发最常用的。
>

### 2.1 组件与 Props ⭐⭐
**本质**：组件是"接受 Props、返回 React 元素"的函数（或类）。Props 是父组件传给子组件的只读数据，遵循单向数据流。

**为什么这样设计**：单向数据流让数据变化可预测——出了 bug 你只需要从上往下追，不用担心子组件偷改了父组件的状态。

**深入细节**：

+ Props 是浅冻结的——你不能直接修改 `props.name = 'xxx'`，但如果 props 传了一个对象，对象内部是可变的（React 不做深冻结）
+ children 是特殊的 prop，可以是任何东西（字符串、元素、函数、数组）
+ 组合模式（Composition）vs 继承：React 官方推荐组合，几乎从不推荐继承
+ React 19 的变化：ref 可以作为普通 prop 传递给函数组件，不再需要 `forwardRef` 包装

**关联**：

+ 向上连接核心原理 3（组件化 = 隔离 + 组合）
+ 与 State 的区别：Props 是外部传入的，State 是内部管理的
+ 和 TypeScript 结合：用 interface/type 定义 Props 类型
+ 与 React 19 的关联：ref 作为 prop 传递简化了组件 API

**过关标准**：

- [ ] 能清晰区分 Props 和 State
- [ ] 能解释为什么 Props 是只读的（单向数据流原则）
- [ ] 了解 children prop 和组合模式（Composition）
- [ ] 知道 React 19 中 ref 作为 prop 的变化
- [ ] 能说出 Composition 优于 Inheritance 的原因

### 2.2 State 与状态更新机制 ⭐⭐⭐
**本质**：State 是组件内部管理的数据，变化会触发重新渲染。React 的状态更新是异步批量的——多个 setState 会合并成一次渲染。

**为什么是异步批量**：如果每个 setState 都立即触发一次渲染，连续调用 3 次就渲染 3 次，性能浪费。批量合并后只渲染 1 次。

**深入细节**：

+ React 18 之前：只有 React 事件处理函数中的 setState 是批量的，setTimeout/Promise 中不是
+ React 18+：自动批处理（Automatic Batching）——**所有**场景中的 setState 都自动批处理，包括 Promise、setTimeout、原生事件
+ 状态更新的两种方式：
    - 直接值：`setCount(5)` —— 多次调用以最后一次为准
    - 函数式更新：`setCount(prev => prev + 1)` —— 多次调用会依次执行，适用于依赖前值的场景
+ 不可变更新：React 通过 `Object.is` 比较新旧 state，如果引用没变就不会重渲染。所以更新对象/数组必须创建新引用
+ 状态初始化：`useState(computeInitialState)` 传函数可以避免每次渲染都执行昂贵的初始化计算

**关联**：

+ 向上连接核心原理 1（UI = f(state)，state 变 → UI 变）
+ 向下：useState 是最基础的状态 Hook
+ 跨组件状态共享 → 引出状态提升、Context、状态管理库
+ 与 Java 并发的类比：React 的批量更新类似于批量提交事务；不可变更新类似于 CopyOnWrite

**过关标准**：

- [ ] 能解释 `setState` / `useState` 为什么是异步的（批量优化）
- [ ] 知道 React 18 的自动批处理——所有场景都自动批处理
- [ ] 能说出 `useState(prev => prev + 1)` 函数式更新和 `useState(count + 1)` 的区别，以及各自适用场景
- [ ] 能解释为什么更新对象/数组时必须创建新引用
- [ ] 能说出 `useState(() => computeExpensiveValue())` 懒初始化的用法

### 2.3 Hooks 体系 ⭐⭐⭐
**本质**：Hooks 是让函数组件拥有状态和副作用能力的函数。它们本质上是挂在 Fiber 节点上的链表，按调用顺序一一对应。

**为什么需要它**：Class 组件的逻辑复用（HOC、render props）太复杂，生命周期方法把不相关的逻辑混在一起（如 componentDidMount 里同时有数据获取和事件订阅）。Hooks 按"功能"而非"生命周期"组织代码。

**核心 Hooks 掌握清单（含 React 19 新增）**：

| Hook | 本质 | 典型场景 | 版本 |
| --- | --- | --- | --- |
| `useState` | 在 Fiber 上存一个值，变化触发重渲染 | 表单输入、开关状态 | 16.8+ |
| `useEffect` | 在 Commit 阶段后异步执行副作用 | 数据获取、订阅、DOM 操作 | 16.8+ |
| `useLayoutEffect` | 在 Commit 阶段后同步执行，阻塞绘制 | DOM 测量、防闪烁 | 16.8+ |
| `useRef` | 在 Fiber 上存一个值，变化不触发重渲染 | 访问 DOM 节点、存上一次值 | 16.8+ |
| `useMemo` | 缓存计算结果，依赖不变就不重新计算 | 昂贵计算、引用稳定性 | 16.8+ |
| `useCallback` | 缓存函数引用，依赖不变就不创建新函数 | 传给子组件的回调 | 16.8+ |
| `useContext` | 订阅 Context 的值 | 主题、语言、认证信息 | 16.8+ |
| `useReducer` | 用 reducer 模式管理复杂状态 | 多字段表单、状态机 | 16.8+ |
| `useTransition` | 将状态更新标记为"非紧急" | 搜索输入与结果列表解耦 | 18+ |
| `useDeferredValue` | 延迟派生一个值的更新 | 控制不了状态更新源时的降级方案 | 18+ |
| `useId` | 生成稳定的唯一 ID，SSR/CSR 一致 | 表单 label + input 关联 | 18+ |
| `useActionState` | 管理 Action 的状态（pending/result） | 表单提交状态管理 | **19** |
| `useFormStatus` | 获取父表单的提交状态 | 提交按钮 loading 态 | **19（react-dom）** |
| `useOptimistic` | 在异步操作完成前显示乐观更新 | 点赞、评论发送 | **19** |
| `use` | 在 render 中读取 Promise/Context | 数据获取、条件读取 Context | **19** |
| `useEffectEvent` | 从 Effect 中提取非响应式逻辑 | Effect 中读取最新值但不作为依赖 | **19.2** |


**Hooks 规则与底层原因**：

+ **只在顶层调用**（不能在条件/循环中）：因为 Hooks 是链表，依赖调用顺序。第一次渲染建立链表，后续渲染按顺序读取。如果条件改变导致调用顺序变化，链表就错位了
+ **只在函数组件或自定义 Hook 中调用**：普通函数没有 Fiber 节点来存储 Hooks 状态

**闭包陷阱**：

+ 问题：useEffect/useCallback 中捕获的是创建时的 state 值，如果依赖数组没有正确设置，会一直使用旧值
+ 解决方案：正确设置依赖数组、使用 ref 存最新值、使用函数式更新 `setState(prev => ...)`
+ React 19.2 的解决方案：`useEffectEvent` 可以从 Effect 中提取非响应式逻辑

**关联**：

+ Hooks 的调用规则 → 因为 Hooks 是链表，依赖调用顺序
+ useMemo/useCallback 与 React.memo 配合 → 性能优化
+ useEffect 的依赖数组 → 闭包陷阱
+ React 19 的 Actions 系统 → useActionState + useFormStatus + useOptimistic 组合

**过关标准**：

- [ ] 能解释 Hooks 为什么不能在条件语句中调用（链表 + 调用顺序）
- [ ] 能说出 useEffect 的执行时机（Commit 后异步执行）和 useLayoutEffect 的区别（同步，阻塞绘制）
- [ ] 能解释 useRef 为什么变化不触发重渲染
- [ ] 能识别常见的闭包陷阱并给出至少两种解决方案
- [ ] 能说出 React 19 新增的 4 个 Hook 各自的作用和使用场景
- [ ] 能解释 `use` Hook 与传统 Hooks 的区别（可以在条件中调用）

### 2.4 React 19 Actions 系统 ⭐⭐⭐
**本质**：React 19 引入了 Actions 的概念——在 Transition 中使用的异步函数。Actions 自动管理 pending 状态、错误处理、乐观更新和表单重置，大幅减少表单处理的样板代码。

**核心 API 组合**：

| API | 作用 | 使用位置 |
| --- | --- | --- |
| `<form action={fn}>` | 表单提交时自动调用 fn，支持渐进式增强 | react-dom |
| `useActionState(fn, initialState)` | 返回 `[state, dispatchAction, isPending]`，管理 Action 的完整生命周期 | react |
| `useFormStatus()` | 在表单的子组件中获取提交状态（pending、data、method） | react-dom |
| `useOptimistic(state, updateFn)` | 在异步操作未完成时显示乐观更新，失败自动回滚 | react |
| `useTransition` | 将任何状态更新标记为非紧急 Transition，可用于触发 Action | react |


**之前 vs 现在的对比**：

之前（React 18）：

```plain
// 需要手动管理 3 个 state + fetch 调用
const [data, setData] = useState(null)
const [error, setError] = useState(null)
const [isPending, setIsPending] = useState(false)

async function handleSubmit(formData) {
  setIsPending(true)
  try {
    const result = await submitForm(formData)
    setData(result)
  } catch (e) {
    setError(e)
  } finally {
    setIsPending(false)
  }
}
```

现在（React 19）：

```plain
const [state, submitAction, isPending] = useActionState(
  async (prevState, formData) => {
    const result = await submitForm(formData)
    return result
  },
  null
)
```

**useOptimistic 的工作流**：

1. 用户操作触发 → 立即用 `addOptimistic(newValue)` 更新 UI
2. 异步操作在后台进行
3. 成功 → 乐观值被真实值替换
4. 失败 → React 自动回滚到原始值

**关联**：

+ 与 Server Actions 配合：Server Action 就是一个用 'use server' 标记的 async 函数，可以直接作为 `<form action>` 的值
+ 与渐进式增强的关系：`<form action>` 在 JS 加载前就能提交（原生表单行为）
+ 与 Next.js 的关系：Next.js 的 Server Actions 就是基于这套 API

**过关标准**：

- [ ] 能说出 Actions 系统解决了什么问题（减少表单处理的样板代码）
- [ ] 能写出 useActionState 的基本用法
- [ ] 能解释 useFormStatus 为什么必须在 `<form>` 的子组件中使用（不能在同一组件中）
- [ ] 能说出 useOptimistic 的工作流程和自动回滚机制
- [ ] 能解释 `<form action={fn}>` 如何支持渐进式增强

### 2.5 事件系统 ⭐⭐
**本质**：React 使用合成事件（SyntheticEvent）系统。事件不是绑定在具体 DOM 节点上，而是通过事件委托统一绑定在 root 节点（React 17+ 是 root 容器而非 document）。

**为什么这样做**：统一跨浏览器行为、减少事件监听器数量、配合 Fiber 的优先级调度。

**深入细节**：

+ React 17 之前事件委托到 document，17+ 改为 root 容器——这样多个 React 应用可以共存
+ 合成事件对象是池化复用的（React 17 之前），17+ 已废弃池化
+ `e.stopPropagation()` 阻止的是 React 合成事件的冒泡，不是原生事件
+ 事件优先级与 Fiber 调度关联：不同事件触发不同优先级的更新
    - 离散事件（click、input）→ 同步优先级
    - 连续事件（scroll、mousemove）→ 较低优先级
+ React 19 中，ref 回调函数支持返回清理函数（类似 useEffect 的 cleanup）

**关联**：

+ 与浏览器原生事件的区别
+ 与 Fiber 调度关联：不同事件有不同优先级
+ React 19 ref cleanup：ref 回调返回清理函数，组件卸载时自动调用

**过关标准**：

- [ ] 知道合成事件和原生事件的区别
- [ ] 能解释事件委托到 root 容器（而非 document）的好处
- [ ] 知道 `e.stopPropagation()` 在 React 中的行为
- [ ] 了解事件优先级与调度的关系
- [ ] 知道 React 19 ref callback cleanup 的用法

### 2.6 Reconciliation 与 Diff 算法细节 ⭐⭐⭐
**本质**：React 的 Diff 不是通用 O(n³) 树对比，而是基于三个假设的 O(n) 启发式算法：

1. **不同类型的元素产生不同的树**——type 不同直接卸载旧树、重建新树
2. **同层级比较，不跨层级移动**——只比较同一层的节点
3. **通过 key 识别同级列表中的节点**——key 帮助 React 追踪节点的"身份"

**深入细节**：

+ **单节点 Diff**：先比较 key，再比较 type。key 和 type 都相同才复用
+ **多节点 Diff（列表）**：React 用两轮遍历——第一轮处理更新（key 相同、type 相同/不同），第二轮处理新增/删除/移动
+ **key 的作用机制**：
    - 没有 key 时按 index 对比，列表头部插入一项 → 后面所有项都被认为"变了"
    - 有 key 时按 key 匹配，React 能识别"这个节点只是移动了位置"
    - 用 index 做 key 的问题：增删排序时 index 变化 → 组件状态错误复用（如输入框内容串位）
+ **key 的高级用法**：给组件一个变化的 key 可以强制重新挂载（重置内部状态）

**关联**：

+ 向上连接虚拟 DOM（Diff 是虚拟 DOM 的核心步骤）
+ 向下影响性能优化（key 的正确使用是最基础的优化）
+ 为什么不能用 index 做 key → 当列表有增删、排序时，index 会导致错误复用

**过关标准**：

- [ ] 能说出 Diff 的三个假设
- [ ] 能解释 key 的作用以及为什么不推荐 index 做 key
- [ ] 能画出列表增删时有 key 和无 key 的对比过程
- [ ] 能说出用 key 强制重新挂载组件的技巧
- [ ] 能解释多节点 Diff 的两轮遍历策略

---

## 第三层：实战应用（场景）
> 结合业务场景的应用能力，面试中常以"你怎么做的"形式出现。
>

### 3.1 状态管理方案选择 ⭐⭐⭐
**本质**：当组件树变深、共享状态变多时，Props 逐层传递（Props Drilling）变得不可维护。不同方案本质上是在"响应式粒度"和"复杂度"之间做取舍。

**方案对比**：

| 方案 | 适用场景 | 核心思路 | 优势 | 劣势 |
| --- | --- | --- | --- | --- |
| useState + Props 传递 | 状态只涉及 2-3 层 | 最简单 | 零额外概念 | 层级多了 Props Drilling |
| useContext | 低频变化的全局数据 | 依赖注入 | 跳过中间组件 | Context 值变化导致所有消费者重渲染 |
| useReducer + Context | 中等复杂度 | Redux 思想轻量版 | 状态逻辑集中 | 样板代码、Context 性能问题 |
| Zustand | 全局状态、中到大规模 | 极简 API、发布-订阅 | 简洁、选择性订阅 | 第三方依赖 |
| Redux Toolkit | 大规模应用、团队协作 | 严格单向数据流、中间件 | 可预测、DevTools 强大 | 学习曲线陡、样板代码多 |
| Jotai/Recoil | 细粒度原子状态 | 原子化状态 | 最细粒度的重渲染控制 | 概念较新 |


**Context 性能问题的深入分析**：

+ 问题：Context 值变化 → **所有**消费该 Context 的组件重渲染，即使只用了其中一个字段
+ 原因：React 对 Context 值做的是引用比较（Object.is），不是深比较
+ 解决方案：
    - 拆分 Context（把频繁变化和不变的分开）
    - 使用 `useMemo` 稳定 Provider 的 value
    - 使用 `React.memo` 包裹消费组件
    - 使用第三方状态管理库（天然支持选择性订阅）
+ React 19.2 的 `<Activity>`：可以用来管理后台活动的状态，配合状态管理使用

**关联**：

+ Context 的性能问题 → 引出为什么需要专门的状态管理库
+ 与 Java 后端的类比：状态管理像是前端的"数据层"

**过关标准**：

- [ ] 能说出 Context 的性能问题（值变化 → 所有消费者重渲染）
- [ ] 能给出 Context 性能问题的 3 种以上解决方案
- [ ] 能根据场景推荐合适的方案并说出理由
- [ ] 了解至少两个第三方状态管理库的核心 API 和设计哲学
- [ ] 能解释 Zustand 的选择性订阅为什么比 Context 性能好

### 3.2 React 性能优化 ⭐⭐⭐
**本质**：React 性能优化的核心思路是"减少不必要的渲染"和"减少渲染的计算量"。

**React 的默认渲染行为**：父组件重渲染 → 所有子组件无条件重渲染（不管 props 有没有变）。这是 React 的设计决策，不是 bug。

**优化手段分层**：

**第一级：避免不必要的渲染触发**

| 手段 | 解决什么问题 | 原理 | 注意事项 |
| --- | --- | --- | --- |
| `React.memo` | 父组件重渲染导致子组件不必要重渲染 | 浅比较 Props，没变就跳过 | 有对比开销，简单组件不值得 |
| `useMemo` | 每次渲染都重复昂贵计算 | 缓存计算结果 | 滥用反而增加内存占用和对比开销 |
| `useCallback` | 每次渲染创建新函数引用，导致 memo 失效 | 缓存函数引用 | 必须配合 memo 使用才有意义 |
| 正确使用 key | 列表渲染时不必要的 DOM 操作 | 帮助 Diff 正确识别节点 | 不用 index，用稳定唯一标识 |


**React 19/Compiler 对优化的影响**：

+ React Compiler（1.0 已稳定）自动在构建时分析组件，自动插入等效的 useMemo/useCallback/memo
+ 启用 Compiler 后，大部分手动 memo 可以移除，代码更简洁
+ Compiler 的限制：不是所有场景都能自动优化（如组件不纯、读取了可变外部变量、使用了 Date.now() 等）
+ 建议策略：启用 Compiler → 移除多余的手动 memo → 只在 Compiler 无法处理的边界情况手动优化

**第二级：减少渲染的计算量**

| 手段 | 解决什么问题 | 工具 |
| --- | --- | --- |
| 代码分割 | 首屏加载太大 | `React.lazy` + `Suspense` |
| 虚拟化列表 | 渲染超长列表 DOM 过多 | `react-window`、`@tanstack/virtual` |
| 避免内联对象/函数 | 每次渲染创建新引用 | 提到组件外或用 useMemo |


**第三级：利用并发特性（React 18+/19）**

| 手段 | 解决什么问题 |
| --- | --- |
| `useTransition` | 非紧急更新不阻塞用户输入 |
| `useDeferredValue` | 延迟派生值的更新 |
| `<Suspense>` + Streaming | 慢数据源不阻塞快内容的显示 |


**过关标准**：

- [ ] 能说出"父组件重渲染 → 子组件一定重渲染"这个默认行为
- [ ] 能解释 memo + useCallback 为什么必须配合使用
- [ ] 能说出 React Compiler 如何改变了性能优化的策略
- [ ] 能说出 React Compiler 的限制（哪些情况无法自动优化）
- [ ] 能说出 React.lazy 和 Suspense 的配合用法
- [ ] 知道不要过度优化——能量化证明有问题再优化
- [ ] 能说出虚拟化列表的原理和适用场景

### 3.3 自定义 Hooks ⭐⭐
**本质**：自定义 Hook 是用 `use` 开头的函数，内部调用其他 Hooks，实现逻辑复用。本质是把状态逻辑从组件中抽离出来，但每次调用都有独立的状态。

**为什么需要它**：Class 时代用 HOC 和 render props 实现复用，嵌套深、调试难。自定义 Hook 用函数组合的方式替代，更直观。

**常见模式**：

| Hook | 功能 | 核心实现 |
| --- | --- | --- |
| `useForm` | 表单状态管理 | useState + 验证逻辑 |
| `useFetch` | 数据请求 + loading + error | useState + useEffect + AbortController |
| `useDebounce` | 输入防抖 | useState + useEffect + setTimeout |
| `useLocalStorage` | 本地存储同步 | useState + useEffect + window.addEventListener('storage') |
| `useMediaQuery` | 响应式断点 | useState + matchMedia |
| `usePrevious` | 获取上一次的值 | useRef + useEffect |


**设计自定义 Hook 的原则**：

+ 单一职责：一个 Hook 只做一件事
+ 返回值要稳定：用 useCallback 稳定回调函数
+ 支持清理：useEffect 中返回 cleanup 函数
+ 考虑竞态：异步操作用 AbortController 或 flag 避免竞态

**关联**：

+ 自定义 Hook 和普通函数的区别：内部调用了 Hooks，受 Hooks 规则约束
+ 与 React 19 的 `use` Hook 配合：自定义 Hook 中可以用 `use` 读取 Promise

**过关标准**：

- [ ] 能手写一个 `useFetch`（包含 loading、error、data、abort）
- [ ] 能手写一个 `useDebounce`
- [ ] 理解每次调用自定义 Hook 都有独立的状态
- [ ] 知道自定义 Hook 的设计原则（单一职责、清理、竞态处理）

### 3.4 表单处理 ⭐⭐
**本质**：React 表单分受控组件（Controlled）和非受控组件（Uncontrolled）。

**React 19 的表单处理革命**：

| 方式 | 特点 | 适用场景 |
| --- | --- | --- |
| 受控组件（useState） | 每个输入都有 state，实时验证 | 简单表单、需要即时反馈 |
| 非受控组件（useRef） | DOM 自管理，按需读取 | 性能敏感的大表单 |
| react-hook-form | 基于非受控，减少重渲染 | 复杂表单 |
| **React 19 form action** | `<form action={fn}>`，自动管理状态 | 新项目，特别是配合 Server Actions |
| **useActionState** | 管理 Action 的完整生命周期 | 表单提交 + 状态反馈 |


**React 19 表单处理链路**：

```plain
<form action={serverAction}>
  └─ 用户提交
      └─ useActionState 管理 pending/result/error
          └─ useFormStatus 让子组件读取表单状态
              └─ useOptimistic 提供即时反馈
```

**过关标准**：

- [ ] 能解释受控和非受控组件的区别及各自适用场景
- [ ] 了解 react-hook-form 的基本用法和它为什么比纯受控表单性能好
- [ ] 能用 React 19 的 Actions API 写一个完整的表单提交流程
- [ ] 能解释 React 19 表单处理相比传统方式减少了哪些样板代码

---

## 第四层：高阶能力（诊断/设计）
> 区分"能用"和"用得好"，全栈岗位不要求太深但能答上来是明显加分项。
>

### 4.1 React 18+ 并发特性 ⭐⭐
**本质**：React 18 引入的并发特性让 React 可以同时准备多个版本的 UI，优先展示紧急更新，延后非紧急更新。React 19 默认启用并发渲染。

**核心 API**：

| API | 作用 | 场景 | React 版本 |
| --- | --- | --- | --- |
| `useTransition` | 将状态更新标记为非紧急 | 搜索输入 vs 结果列表 | 18+ |
| `useDeferredValue` | 延迟派生一个值 | 不可控的状态更新源 | 18+ |
| `Suspense` | 声明式处理异步加载 | 代码分割、数据获取 | 16.6+（18 扩展） |
| `startTransition` | 函数版 useTransition | 非组件中使用 | 18+ |


**React 19 对 Transition 的增强**：

+ Transition 中可以使用 async 函数，自动管理 pending 状态
+ Transition 中的错误自动被 Error Boundary 捕获
+ 配合 useOptimistic 实现乐观更新

**过关标准**：

- [ ] 能举出 useTransition 的使用场景（搜索框输入 + 结果列表）
- [ ] 理解 Suspense 的工作机制（组件 throw Promise → 最近的 Suspense 捕获 → 显示 fallback）
- [ ] 了解并发渲染不是多线程，而是时间切片（在单线程上分片执行）
- [ ] 能说出 React 19 中 Transition 支持 async 函数的意义

### 4.2 React 19.2 新特性 ⭐⭐
**Activity（原 Offscreen）**：

+ 本质：`<Activity mode="visible|hidden">` 让你控制 UI 的可见性，hidden 模式下组件用 `display: none` 隐藏但保持状态
+ 场景：Tab 切换保持状态、预渲染即将访问的页面、后退导航保留表单输入
+ 与条件渲染的区别：条件渲染会卸载组件丢失状态，Activity 保持状态

**useEffectEvent**：

+ 本质：从 Effect 中提取"非响应式"逻辑——这些逻辑需要读取最新值，但不应该作为 Effect 的依赖
+ 场景：Effect 中使用最新的回调/值，但不想让它触发 Effect 重新执行
+ 解决的问题：之前要么把值加到依赖数组（导致 Effect 过于频繁执行），要么用 ref 手动存最新值（繁琐）

**Partial Pre-rendering（PPR）**：

+ 服务端可以预渲染页面的静态部分，延迟渲染动态部分
+ 与 Next.js 16 的 Cache Components 深度集成

**React Performance Tracks**：

+ Chrome DevTools 中新增 React 专用性能面板
+ 显示 Scheduler 优先级、渲染时间、更新阻塞等信息

**过关标准**：

- [ ] 能说出 Activity 的使用场景和与条件渲染的区别
- [ ] 能说出 useEffectEvent 解决了什么问题
- [ ] 了解 PPR 的基本概念

### 4.3 React Compiler ⭐⭐⭐
**本质**：React Compiler 是一个构建时工具（Babel 插件），自动分析组件和 Hooks 的数据流，在安全的地方自动插入等效的 memo/useMemo/useCallback，无需手动优化。

**工作原理**：

+ 构建时进行纯度分析（purity analysis）：检查组件是否无副作用
+ 对每个 prop 进行独立的等值检查
+ 如果组件不纯（如读取了 Date.now()、Math.random()、可变 ref），会"bail out"回退到全量渲染

**实际影响**：

+ 可以移除大部分手动 useMemo/useCallback/React.memo
+ 代码更简洁、可读性更好
+ 测试代码减少（不需要 mock 那么多 memo 逻辑）
+ Meta 内部数据：使用 Compiler 后 Instagram 页面加载提速 ~12%
+ Compiler 1.0 已稳定，Next.js 16 内置支持

**何时还需要手动 memo**：

+ 第三方库要求稳定引用
+ Compiler 无法分析的边界情况（跨组件边界的信息隐藏）
+ useEffect 依赖中的函数引用（某些情况 Compiler 仍然需要手动 useCallback）

**过关标准**：

- [ ] 能说出 React Compiler 的工作原理（构建时纯度分析 + 自动 memo）
- [ ] 知道 Compiler 的限制（哪些情况会 bail out）
- [ ] 能说出启用 Compiler 后开发策略的变化
- [ ] 了解 Compiler 1.0 的稳定性状态和生产案例

### 4.4 渲染性能诊断 ⭐⭐
**诊断工具**：

+ React DevTools Profiler：看每次渲染中各组件耗时、渲染原因
+ React DevTools "Highlight updates"：可视化哪些组件在重渲染
+ Chrome DevTools Performance 面板：看主线程阻塞情况
+ **React 19.2 Performance Tracks**：Chrome DevTools 中的 React 专用面板，显示 Scheduler 优先级和渲染轨道
+ `<React.StrictMode>` 的双重渲染：帮助发现副作用问题

**诊断流程**：

1. 用户反馈卡顿 → Performance 面板录制
2. 看 Long Task（>50ms）在哪个组件
3. 用 Profiler 查看该组件的渲染次数和耗时
4. 确认是否不必要的重渲染（Highlight updates）
5. 根据原因选择优化手段（memo/拆分组件/虚拟化）

**过关标准**：

- [ ] 知道用 Profiler 定位不必要的重渲染
- [ ] 知道 StrictMode 为什么会让 useEffect 执行两次（开发模式下检测副作用）
- [ ] 能描述一个完整的性能诊断流程
- [ ] 了解 React 19.2 Performance Tracks 的作用

### 4.5 React 设计模式 ⭐⭐
**常见模式**：

| 模式 | 本质 | 适用场景 | 现状 |
| --- | --- | --- | --- |
| **组合模式** | 用 children 和 slots 替代继承 | 通用 | React 官方推荐，最常用 |
| **Render Props** | 通过 props 传递渲染函数 | 需要共享渲染逻辑时 | Hooks 出现后较少使用 |
| **HOC** | 包装组件添加功能 | 横切关注点（如认证） | Hooks 出现后较少使用 |
| **Compound Components** | 多个组件协作暴露统一 API | Tab、Accordion、Menu | 组件库常用 |
| **Provider Pattern** | Context + Provider | 全局配置 | 依然常用 |
| **Container/Presentational** | 逻辑与 UI 分离 | 大组件拆分 | Hooks 后弱化，但思想仍有价值 |
| **State Reducer** | 暴露 reducer 让使用者控制状态逻辑 | 高度可定制的组件 | 组件库设计常用 |


**React 19 对模式的影响**：

+ RSC 引入了新的模式：Server/Client 组件边界的设计
+ Actions 减少了对表单相关 HOC/Render Props 的需求
+ Compiler 减少了对 memo 相关优化模式的需求

**过关标准**：

- [ ] 能说出 Composition 优于 Inheritance 的原因
- [ ] 了解 Compound Components 的使用场景（举例 Tabs）
- [ ] 能说出 Server/Client 组件边界的设计原则

---

## 面试场景题
### 基础追问
1. "React 的虚拟 DOM 一定比直接操作 DOM 快吗？为什么？"
2. "说说 key 的作用？为什么不推荐用 index？给一个会出 bug 的例子。"
3. "useEffect 的依赖数组为空和不传有什么区别？"
4. "useState 在一个事件处理函数里连续调用三次，渲染几次？React 18 之前和之后有区别吗？"
5. "React 19 中 ref 的使用方式有什么变化？"

### 原理追问
6. "Hooks 为什么不能在条件语句里调用？底层是怎么存储的？"
7. "React 的合成事件和原生事件有什么区别？事件绑定在哪里？"
8. "什么是闭包陷阱？举个 useEffect 中的例子，怎么解决？React 19.2 的 useEffectEvent 如何帮助解决？"
9. "React Compiler 是什么？它如何改变了性能优化的策略？什么情况下还需要手动 memo？"
10. "解释一下 Server Component 和 Client Component 的区别？'use client' 指令到底做了什么？"

### 场景设计
11. "一个页面上有一个搜索框和一个结果列表，输入时列表要实时过滤。怎么优化性能？"
    - 期望回答：防抖 + useTransition / useDeferredValue + 虚拟列表（大数据量时）
12. "你的应用有一个全局的主题切换功能，怎么设计？"
    - 期望回答：Context + Provider，说出 Context 的重渲染问题及应对（拆分 Context、memo、或用 Zustand）
13. "一个表单有 20 个字段，每个字段变化都导致整个表单重渲染，怎么优化？"
    - 期望回答：非受控组件 / react-hook-form / 拆分子组件 + memo / React 19 Actions
14. "用 React 19 的 Actions API 重写一个点赞功能，要求即时反馈（乐观更新），失败回滚。"
    - 期望回答：useOptimistic + useActionState + Server Action
15. "你的组件需要在隐藏时保持状态（如 Tab 切换），React 19.2 提供了什么方案？"
    - 期望回答：`<Activity mode="hidden">`，解释与条件渲染的区别

### 综合设计
16. "设计一个实时协作文档编辑器的前端架构，包括状态管理、性能优化、用户体验。"
    - 考察点：状态管理选型、乐观更新、虚拟化、WebSocket 集成、错误处理

---

## 实践计划
| 方式 | 具体做法 | 覆盖知识点 | 预计时间 |
| --- | --- | --- | --- |
| 本地实验 | 用 React DevTools Profiler 分析一个现有项目的渲染性能 | 性能诊断、memo、重渲染 | 2h |
| 本地实验 | 手写 `useFetch`（含 loading/error/abort）和 `useDebounce` | 自定义 Hooks、闭包、依赖数组、竞态 | 3h |
| 本地实验 | 用 React 19 Actions API 写一个完整的表单（CRUD + 乐观更新） | useActionState、useOptimistic、useFormStatus | 4h |
| 本地实验 | 启用 React Compiler，对比前后的渲染次数和 bundle 大小 | React Compiler、性能优化策略变化 | 2h |
| 项目中实践 | 在个人全栈项目中用 Context + Zustand 做状态管理 | 状态管理方案对比 | 持续 |
| Side Project | 搭一个带搜索、筛选、虚拟列表的数据展示页 | 性能优化全链路、useTransition、useDeferredValue | 1 周 |
| Side Project | 用 React 19 + Activity 做一个多 Tab 仪表盘 | Activity、状态保持、并发特性 | 3-5h |


