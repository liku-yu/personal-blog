---
title: 深入理解 useEffect 的依赖数组
date: 2026-08-20
tags: [React, 前端]
description: 依赖数组是很多 React Bug 的源头，这篇文章从 Object.is 判定规则讲起，覆盖闭包陷阱、竞态问题与 exhaustive-deps 的正确用法。
---

`useEffect` 的依赖数组（dependency array）是 React 文档里反复强调"不要骗它"的东西。理解它的判定规则，能少踩 80% 的坑——剩下 20% 是竞态问题，本文也一并讲清楚。

## 依赖数组是什么

```tsx
useEffect(() => {
  // 副作用逻辑
  return () => {
    // 清理逻辑
  };
}, [dep1, dep2]);
```

**依赖数组只做一件事：告诉 React 什么时候重新执行副作用。** 每次渲染后，React 会比较新旧依赖数组，只要有一个元素用 `Object.is` 判断不相等，就：

1. 先执行上一次的**清理函数**（如果有）
2. 再执行新的副作用

### Object.is 判定的细节

`sameValueZero`（`Object.is` 的变体）意味着：

| 比较 | 结果 | 解释 |
| --- | --- | --- |
| `1 === 1` | 相等 | 原始值按值比较 |
| `NaN === NaN` | **相等**（`Object.is` 视 NaN 相等） | 与 `===` 不同 |
| `{} === {}` | **不相等** | 对象按引用比较 |
| `fn1 === fn2`（每次渲染新建） | **不相等** | 函数按引用比较 |

> 关键推论：**依赖数组里放对象 / 函数 / 数组，每次渲染都是新引用 → 每次都会触发 effect**。这是海量 Bug 的来源。

## 三个常见陷阱

### 陷阱 1：忘记加依赖（闭包陷阱）

```tsx
const [count, setCount] = useState(0);

useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []);            // ❌ 依赖数组为空，count 永远闭包住初始值
```

`count` 永远是首次渲染闭包里的 0，`setCount(count + 1)` 会一直把状态设成 1。

**正确做法**：用函数式更新，让 setState 自己读到最新值：

```tsx
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);            // ✅ 不引用 count 本身，无需加入依赖
```

**或**：把 `count` 加进依赖数组（每次 count 变化重建定时器）：

```tsx
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, [count]);       // ✅ 逻辑直白但定时器频繁重建
```

### 陷阱 2：依赖每次渲染都变（引用不稳定）

```tsx
useEffect(() => {
  fetchData(props.obj);
}, [props.obj]);   // ❌ props.obj 每次都是新引用 → 无限循环
```

父组件只要重新渲染，`props.obj` 就是新对象，effect 永远达不到"稳定"状态，**反复触发甚至死循环**。

**解法**：用 `useMemo` 稳定引用，或依赖原始值：

```tsx
const obj = useMemo(() => ({ id: props.id, mode: props.mode }), [props.id, props.mode]);
useEffect(() => { fetchData(obj); }, [obj]);

// 或更简单——直接依赖最底层的原始值
useEffect(() => { fetchDataById(props.id); }, [props.id]);
```

### 陷阱 3：函数依赖导致反复重跑

```tsx
// 每次渲染都新建 onSave → effect 每次都重跑
useEffect(() => {
  save(onSave());          // ❌ onSave 引用不稳定
}, [onSave]);
```

**解法**是 `useCallback` 稳定函数：

```tsx
const onSave = useCallback(() => {
  save(data);
}, [data]);              // 只有 data 变化时 onSave 才换新

useEffect(() => {
  onSave();
}, [onSave]);
```

## 竞态问题（最容易忽略的坑）

异步请求 + 用户快速切换，旧请求可能**后返回**并覆盖新结果：

```tsx
const [data, setData] = useState(null);

useEffect(() => {
  let cancelled = false;          // ① 用标志位标记"已过期"
  fetch(`/api/user/${id}`)
    .then(res => res.json())
    .then(json => {
      if (!cancelled) setData(json);   // ② 只有未过期才写状态
    });
  return () => { cancelled = true; }; // ③ 清理时把旧请求作废
}, [id]);
```

**原理**：effect 重新执行前，React 会先调用上一次的清理函数——把 `cancelled` 置 true，旧请求的 then 回调即使晚到也不会污染状态。

> 这是"不要骗依赖"之外的进阶版：**即使依赖填对了，异步竞态依然存在**。接口请求、轮询、订阅这三类副作用务必带上清理逻辑。

## StrictMode 下 effect 执行两次（开发模式）

React 18+ 的 `<StrictMode>` 在**开发模式**下会故意挂载 → 卸载 → 再挂载，让 effect 跑两遍：

```tsx
useEffect(() => {
  console.log("effect 执行");
  return () => console.log("清理");
}, []);
// 开发模式输出：effect → 清理 → effect
// 生产模式输出：effect
```

这是**故意的**，目的是暴露"忘记写清理函数"的问题（比如订阅、监听器没解绑）。不要惊慌，也不要靠 `useRef` 强行跳过——写对清理函数，生产环境只会跑一次。

## exhaustive-deps：让 ESLint 帮你把关

`eslint-plugin-react-hooks` 的 `exhaustive-deps` 规则会**自动检查依赖是否写全**：

```bash
npm install -D eslint-plugin-react-hooks
```

```json
// .eslintrc.json
{
  "plugins": ["react-hooks"],
  "rules": { "react-hooks/exhaustive-deps": "warn" }
}
```

当它提示 `React Hook useEffect has missing dependencies`，**不要靠 `// eslint-disable` 蒙混**——那等于把 bug 留下来。正确流程：

1. 看提示缺了谁
2. 问自己：这个值真的应该触发 effect 吗？
3. 是 → 加进依赖；不是 → 用 `useCallback`/`useMemo` 稳定它，或函数式更新化掉

> 旧 React 时代有人教"依赖数组删掉就是只在挂载时跑一次"——那是误用。真正"只在挂载跑一次"的场景应该没有外部依赖，写了依赖就不该删。

## 与 useLayoutEffect 的区别

| | `useEffect` | `useLayoutEffect` |
| --- | --- | --- |
| 执行时机 | 浏览器**绘制后**（异步） | DOM 更新后、绘制**前**（同步） |
| 适用场景 | 数据请求、订阅、埋点 | 测量 DOM、同步改样式 |
| 闪烁风险 | 无 | 会阻塞绘制 |
| SSR | 不执行 | **警告**（SSR 下不执行） |

```tsx
// 要在绘制前同步调整 DOM —— 用 useLayoutEffect
useLayoutEffect(() => {
  const width = el.getBoundingClientRect().width;
  setLayout({ width });
}, []);
```

绝大多数场景用 `useEffect` 就够；只有"改完 DOM 之前必须测量/修正"才需要 `useLayoutEffect`。

## 结论

> 依赖数组不是"性能优化开关"，而是"同步契约"。不要靠删依赖来"修"问题，要搞清楚副作用真正依赖什么。

把依赖数组当作函数参数来思考，大部分疑难 Bug 都能迎刃而解：

1. **依赖什么就写什么**，原始值优先
2. **对象/函数用 useCallback / useMemo 稳定引用**
3. **有副作用就有清理**，异步请求务必防竞态
4. 交给 `exhaustive-deps` 提醒，但理解它为什么提醒