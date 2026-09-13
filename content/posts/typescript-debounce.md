---
title: 用 TypeScript 写一个可即时取消的防抖函数
date: 2026-08-28
tags: [TypeScript, JavaScript]
description: 从闭包讲起，实现带 cancel / flush 能力的类型安全防抖，并深入 this 绑定、leading edge、React 集成与单元测试。
---

防抖（debounce）是前端高频场景的基础工具：**在事件停止触发一段时间后，才执行最后一次回调**。

## 为什么需要防抖

输入框的实时搜索、窗口 resize、按钮连点，这些场景的事件频率远超我们的处理能力。防抖把"连续触发"折叠成"一次执行"，是最廉价的性能优化手段之一。

## 防抖 vs 节流

两者常被混淆，先分清：

| | 防抖 debounce | 节流 throttle |
| --- | --- | --- |
| 行为 | 停止触发后延迟执行**最后一次** | 固定间隔内**最多执行一次** |
| 场景 | 输入搜索、窗口 resize 结束 | 滚动监听、拖拽、游戏频率限制 |
| 类比 | 电梯等人：没人进就关门 | 限流：每 100ms 最多放一次 |

```text
连续点击 10 次（间隔 50ms）：
  debounce(300ms)  → 停止后第 300ms 执行 1 次
  throttle(300ms)  → 前 300ms 内最多执行 1 次
```

## 第一版：最简实现（trailing edge）

```typescript
function debounce<F extends (...args: any[]) => void>(
  fn: F,
  delay = 300
): (...args: Parameters<F>) => void {
  let timer: ReturnType<typeof setTimeout> | null = null;

  return (...args: Parameters<F>) => {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

**关键在泛型 `F`**：`Parameters<F>` 保留了原函数参数类型，调用方无需任何类型断言——这也是 `ReturnType` / `Parameters` 这套内置工具类型的核心用法。

## 第二版：支持取消与立即执行（flush）

光靠闭包收尾还不够，业务上常需要：

- `cancel()`：组件卸载时作废未执行的调用
- `flush()`：立即执行积压的最后一次（比如表单提交前）

```typescript
interface Debounced<F extends (...args: any[]) => void> {
  (...args: Parameters<F>): void;
  cancel(): void;
  flush(): void;
}

function debounce<F extends (...args: any[]) => void>(
  fn: F,
  delay = 300
): Debounced<F> {
  let timer: ReturnType<typeof setTimeout> | null = null;
  let lastArgs: Parameters<F> | null = null;

  const debounced = (...args: Parameters<F>) => {
    lastArgs = args;                       // 记下最新参数
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      timer = null;
      fn(...args);
    }, delay);
  } as Debounced<F>;

  debounced.cancel = () => {
    if (timer) clearTimeout(timer);
    timer = null;
    lastArgs = null;                       // 连参数一起清掉
  };

  debounced.flush = () => {
    if (timer && lastArgs) {               // 有积压才立即执行
      clearTimeout(timer);
      timer = null;
      fn(...lastArgs);
      lastArgs = null;
    }
  };

  return debounced;
}
```

注意 `flush` 里要保留 `lastArgs` 在调用前读取——否则 `clearTimeout` 之后再次读取会拿到 null。

## 进阶：leading edge（先执行一次，再防抖）

默认版本是"停止后才执行"（trailing）。但有时想要"**第一次立即触发展，后续节流**"——典型的抢购按钮、防重复提交：

```typescript
function debounceLeading<F extends (...args: any[]) => void>(
  fn: F,
  delay = 300
): Debounced<F> {
  let timer: ReturnType<typeof setTimeout> | null = null;
  let lastArgs: Parameters<F> | null = null;

  const debounced = (...args: Parameters<F>) => {
    lastArgs = args;
    if (timer !== null) return;            // 还在窗口期内 → 忽略

    fn(...args);                           // 首次立即执行
    timer = setTimeout(() => {
      timer = null;
      // 结束后若还有新输入，可在 next tick 再触发（可选：trailing 补发）
    }, delay);
  } as Debounced<F>;

  // cancel / flush 略，与上一版类似
  return debounced;
}
```

> 完整 Lodash 支持 `{ leading: true, trailing: true }` 双模式，本文聚焦核心思路，两个方向各给一版。

## this 绑定的坑

上两版用 `fn(...args)` 调用，会丢失 `this`。如果原函数依赖 `this`（对象方法）：

```typescript
const obj = {
  value: 42,
  print() { console.log(this.value); }
};
const d = debounce(obj.print, 100);   // ❌ 调用时 this 是 undefined（严格模式）
d();
```

**修复**：在调用时显式回传 `this`：

```typescript
type DebounceFn<F> = F & { cancel(): void; flush(): void };

function debounce<F extends (this: unknown, ...args: any[]) => void>(
  fn: F,
  delay = 300
): F & { cancel(): void; flush(): void } {
  let timer: ReturnType<typeof setTimeout> | null = null;

  function debounced(this: unknown, ...args: Parameters<F>) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);  // ✅ 用 apply 回传 this
  }

  debounced.cancel = () => { if (timer) clearTimeout(timer); timer = null; };
  debounced.flush = function (this: unknown) { /* 同样 apply */ };

  return debounced as typeof debounced & { cancel(): void; flush(): void };
}

const d = debounce(obj.print, 100);
obj.d = d;
obj.d();        // ✅ this 指向 obj，输出 42
```

## React 中集成（useRef 保引用稳定）

React 组件里直接 `const onSearch = debounce(...)` 会导致**每次渲染都新建防抖函数**，定时器状态丢失。正确做法是 `useRef` 存一份：

```tsx
import { useRef, useCallback } from "react";

function SearchBox() {
  const debouncedRef = useRef<ReturnType<typeof debounce> | null>(null);

  // 只在挂载时创建一次
  if (!debouncedRef.current) {
    debouncedRef.current = debounce((keyword: string) => {
      fetch(`/api/search?q=${keyword}`);
    }, 500);
  }

  // 卸载时清理
  useEffect(() => {
    const d = debouncedRef.current!;
    return () => d.cancel();
  }, []);

  return <input onChange={e => debouncedRef.current!(e.target.value)} />;
}
```

> 更省事的方式：用 `useRef` 存最新回调（`useLatest`），避免闭包陈旧值；或直接引入 `lodash/debounce` 封装成 `useDebouncedCallback`。

## 使用示例

```typescript
const onSearch = debounce((keyword: string) => {
  console.log("请求搜索接口：", keyword);
}, 500);
```

输入框每次 keyup 都调用 `onSearch(value)`，但只有停顿 500ms 后才会真正发出请求；组件卸载时调用 `onSearch.cancel()`，避免回调在组件销毁后触发导致内存泄漏。

```typescript
// flush 场景：搜索框失焦/提交时立即拿到结果
input.addEventListener("keyup", () => onSearch(input.value));
input.addEventListener("blur", () => onSearch.flush());  // 不等防抖结束
```

## 单元测试（fake timers）

验证防抖逻辑，用 `vi.useFakeTimers()`（Vitest）或 Jest 的 fake timers：

```typescript
import { vi, describe, it, expect } from "vitest";

describe("debounce", () => {
  it("只在停止触发后执行一次", () => {
    vi.useFakeTimers();
    const fn = vi.fn();
    const d = debounce(fn, 300);

    d();
    vi.advanceTimersByTime(100);
    d();
    vi.advanceTimersByTime(100);
    d();                          // 最后一次触发
    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(300);  // 超过延迟
    expect(fn).toHaveBeenCalledTimes(1);
  });

  it("cancel 后不再执行", () => {
    vi.useFakeTimers();
    const fn = vi.fn();
    const d = debounce(fn, 300);

    d();
    d.cancel();
    vi.advanceTimersByTime(500);
    expect(fn).not.toHaveBeenCalled();
  });

  it("flush 立即执行积压调用", () => {
    vi.useFakeTimers();
    const fn = vi.fn();
    const d = debounce(fn, 300);

    d(1);
    d.flush();
    expect(fn).toHaveBeenCalledWith(1);
  });
});
```

## 小结

一个成熟的防抖函数 = **闭包管理定时器 + 泛型保留类型 + cancel/flush 控制流**。掌握这三个层次的实现，你不仅能封装出类型安全的工具，也能在 React、组件卸载、表单提交等场景里正确使用它。

**延伸阅读**：
- 节流（throttle）实现与防抖几乎同构：换成"记录上次执行时间戳"即可
- Lodash 的 `debounce` 支持 `leading/trailing/maxWait`，生产项目可直接引入