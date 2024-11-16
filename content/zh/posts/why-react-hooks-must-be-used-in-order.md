+++
title = "深入源码解析：为什么 React Hook 必须按顺序使用" 
date = 2024-11-01
draft = false 
tags = ["React", "Hook"] 
+++

## 背景

React 的官方文档中有一条规则：

> 不要在循环，条件或嵌套函数中调用 Hook， 确保总是在你的 React 函数的最顶层以及任何 return 之前调用他们。

<!--more-->

但是为什么呢？

## 让源码告诉你

### Hook 是怎么存储的

从 [源码中的描述中](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js#L268C1-L269C1) 我们可以知道，Hooks 是以链表的形式存储在 Fiber 的 `memoizedState` 字段中。

- current Hook list 链表是属于 current fiber 的。
- workInProgress Hook list 是属于 workInProgress fiber 的。

[Hook 的存储结构](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js#L197)是一个链表，每个 Hook 都有一个 next 指针指向下一个 Hook。

```ts
export type Hook = {
  memoizedState: any;
  baseState: any;
  baseQueue: Update<any, any> | null;
  queue: any;
  next: Hook | null;
};
```

大概长这样：

```
┌────────────────────┐     ┌────────────────────┐     ┌────────────────────┐
│ Hook 1             │     │ Hook 2             │     │ Hook 3             │
│                    │     │                    │     │                    │
│ memoizedState: any │     │ memoizedState: any │     │ memoizedState: any │
│ baseState: any     │     │ baseState: any     │     │ baseState: any     │
│ baseQueue: Update  │     │ baseQueue: Update  │     │ baseQueue: Update  │
│ queue: any         │     │ queue: any         │     │ queue: any         │
│ next: Hook  ───────┼────>│ next: Hook  ───────┼────>│ next: Hook  ───────> null
└────────────────────┘     └────────────────────┘     └────────────────────┘

```

### 初次渲染

```ts
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,

    baseState: null,
    baseQueue: null,
    queue: null,

    next: null,
  };

  if (workInProgressHook === null) {
    // This is the first hook in the list
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

在初次渲染的时候，React 会调用 `mountWorkInProgressHook` 函数构建 Hooks 链表。
放入 `currentlyRenderingFiber.memoizedState` 中。

### 更新

```ts
function updateWorkInProgressHook(): Hook {
  // This function is used both for updates and for re-renders triggered by a
  // render phase update. It assumes there is either a current hook we can
  // clone, or a work-in-progress hook from a previous render pass that we can
  // use as a base.
  let nextCurrentHook: null | Hook;
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    nextCurrentHook = currentHook.next;
  }

  let nextWorkInProgressHook: null | Hook;
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // There's already a work-in-progress. Reuse it.
    workInProgressHook = nextWorkInProgressHook;
    nextWorkInProgressHook = workInProgressHook.next;

    currentHook = nextCurrentHook;
  } else {
    // Clone from the current hook.

    if (nextCurrentHook === null) {
      const currentFiber = currentlyRenderingFiber.alternate;
      if (currentFiber === null) {
        // This is the initial render. This branch is reached when the component
        // suspends, resumes, then renders an additional hook.
        // Should never be reached because we should switch to the mount dispatcher first.
        throw new Error(
          "Update hook called on initial render. This is likely a bug in React. Please file an issue."
        );
      } else {
        // This is an update. We should always have a current hook.
        throw new Error("Rendered more hooks than during the previous render.");
      }
    }

    currentHook = nextCurrentHook;

    const newHook: Hook = {
      memoizedState: currentHook.memoizedState,

      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,

      next: null,
    };

    if (workInProgressHook === null) {
      // This is the first hook in the list.
      currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
    } else {
      // Append to the end of the list.
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }
  return workInProgressHook;
}
```

在组件触发更新的时候，React 会调用 `updateWorkInProgressHook` 函数来更新 Hooks 链表。
阅读代码后不难发现：

React 会从 `currentHook` 的节点中获取数据，然后将数据更新到 `workInProgressHook` 的节点中。

### 为什么要按顺序使用 Hook

所以不难推导出，如果我们在循环、条件或嵌套函数中调用 Hook，那么在更新的时候，React 就无法正确的找到对应的 Hook 节点，导致数据错乱。

假设我们有这样一个组件：

```jsx
const firstRender = true;
function MyComponent() {
  const [count, setCount] = useState(0);
  if (firstRender) {
    const [name, setName] = useState("Alice");
  }
  const [age, setAge] = useState(18);

  firstRender = false;
  ...
}
```

1. 在初次渲染的时候，hooks 链表是这样的：

```
┌──────────────────────────┐           ┌──────────────────────────┐           ┌──────────────────────────┐
│ count                    │           │ name                     │           │ age                      │
│                          │           │                          │           │                          │
│ memoizedState: 0         │           │ memoizedState: "Alice"   │           │ memoizedState: 18        │
│ baseState: 0             │           │ baseState: "Alice"       │           │ baseState: 18            │
│ baseQueue: null          │           │ baseQueue: null          │           │ baseQueue: null          │
│ queue: null              │           │ queue: null              │           │ queue: null              │
│ next: Hook ──────────────┼──────────>│ next: Hook ──────────────┼──────────>│ next: Hook ─────────────> null
└──────────────────────────┘           └──────────────────────────┘           └──────────────────────────┘
```

2. 在第二次渲染的时候，hook name 消失了，根据链表的特性，hook age 的数据是从 current.count.next 中获取的，所以这次渲染的 age 的数据就变成了 name 的数据。导致数据错乱。

```
┌──────────────────────────┐           ┌──────────────────────────┐
│ count                    │           │ age                      │
│                          │           │                          │
│ memoizedState: 0         │           │ memoizedState: "Alice"   │
│ baseState: 0             │           │ baseState: "Alice"       │
│ baseQueue: null          │           │ baseQueue: null          │
│ queue: null              │           │ queue: null              │
│ next: Hook ──────────────┼──────────>│ next: Hook ──────────────┼──────────>null
└──────────────────────────┘           └──────────────────────────┘
```

## 总结

所以，如果我们在循环、条件或嵌套函数中调用 Hook，那么在更新的时候，React 就有可能无法正确的找到对应的 Hook 节点，导致数据错乱。这也是为什么 React 的官方文档中有这样一条规则。
