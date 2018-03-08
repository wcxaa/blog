---
title: webapck Tapable
date: 2018-03-08
---

# Tapable

Tapable 是一个小型的库，允许你对一个 javascript 模块添加和应用插件。它可以被继承或混入到其他模块中。类似于 NodeJS 的 EventEmitter 类，专注于自定义事件的触发和处理。

webpack 两个核心对象 compiler 和 compilation 都是继承自该对象，基于该库编写的 webpack，具有很好的拓展性，整个系统非常有弹性。但 webpack3 使用的 Tapable 版本还没到 1.0.0，此版本插件系统比较混乱，难于调试，webpack4 使用了升级了的 Tapable 1.0.0，此版本极大的改善了这个问题。

下面我就来分析 webpack 4.1.0 使用的 Tapable 1.0.0：（请先查看[README](https://github.com/webpack/tapable/blob/v1.0.0/README.md)）

## exports

Tapable exports 了这些 class:

```js
import {
  SyncHook,
  SyncBailHook,
  SyncWaterfallHook,
  SyncLoopHook,
  AsyncParallelHook,
  AsyncParallelBailHook,
  AsyncSeriesHook,
  AsyncSeriesBailHook,
  AsyncSeriesWaterfallHook,
  HookMap,
  MultiHook,
  Tapable
} from "tapable";
// AsyncSeriesLoopHook 没有export
```

webpack 中 Sync 开头的 Hook 都是通过 call 来调用，一旦出错就 throw error，停止执行剩下的插件；Async 开头的 Hook 都是通过 callAsync 调用，一旦出错就调用 callback(error) ； 目前没发现通过 promise 调用的。

### SyncHook

内部调用 callTapsSeries ，同步顺序的调用注册过的插件

```js
// 这样使用:
const hook = new SyncHook(["test", "arg2"]);
const mock = jest.fn();
hook.tap("D", mock);
hook.call("1", 2);
expect(mock).toHaveBeenLastCalledWith("1", 2);
```

```js
// tap, tapPromise, tapAsync 可以调整插件的顺序
const hook = new SyncHook();
const calls = [];
hook.tap("A", () => calls.push("A"));
hook.tap(
  {
    name: "B",
    before: "A"
  },
  () => calls.push("B")
);

calls.length = 0;
hook.call();
expect(calls).toEqual(["B", "A"]);

hook.tap(
  {
    name: "C",
    before: ["A", "B"]
  },
  () => calls.push("C")
);

calls.length = 0;
hook.call();
expect(calls).toEqual(["C", "B", "A"]);

hook.tap(
  {
    name: "D",
    before: "B"
  },
  () => calls.push("D")
);

calls.length = 0;
hook.call();
expect(calls).toEqual(["C", "D", "B", "A"]);

hook.tap(
  {
    name: "E",
    stage: -5
  },
  () => calls.push("E")
);
hook.tap(
  {
    name: "F",
    stage: -3
  },
  () => calls.push("F")
);

calls.length = 0;
hook.call();
expect(calls).toEqual(["E", "F", "C", "D", "B", "A"]);
```

### SyncBailHook

内部调用 callTapsSeries ，同步顺序的调用注册过的插件，遇到第一个返回值不为 undefined 的插件就停止执行剩下的插件，返回插件返回值。

```js
// 这样使用：
const hook = new SyncBailHook(["a", "b"]);
hook.tap("A", (a, b) => [a, b]);
expect(hook.call(1, 2)).toEqual([1, 2]);
```

### SyncWaterfallHook

内部调用 callTapsSeries ，同步顺序的调用注册过的插件，正如其名字，它是瀑布式的调用，后一个插件可以通过第一个参数获得前一个插件的返回值（第一个获得 call 时设置的初始值），执行结束会将最后一个插件的返回值返回。

### SyncLoopHook

内部调用 callTapsLooping , callTapsLooping 内部调用 callTapsSeries ，同步顺序循环的调用注册过的插件，循环条件是无论插件类型为 sync or async or promise , 只要插件返回值不为 undefined ，都会从头重新循环，挨个顺序调用插件。在当前的 webpack 版本中没发现有使用 SyncLoopHook。

### AsyncParallelHook

内部调用 callTapsParallel ，异步并行调用注册过的插件，当插件全部完成时执行回调。

### AsyncParallelBailHook

内部调用 callTapsParallel ，异步并行调用注册过的插件，遇到第一个回调有返回值的插件就调用 Hook 的回调。在当前的 webpack 版本中没发现有使用 AsyncParallelBailHook。

### AsyncSeriesHook

内部调用 callTapsSeries ，异步顺序的调用注册过的插件

### AsyncSeriesBailHook

内部调用 callTapsSeries ，异步顺序的调用注册过的插件，遇到第一个回调有返回值的插件就停止执行剩下的插件，调用 Hook 的回调。

### AsyncSeriesWaterfallHook

内部调用 callTapsSeries ，异步顺序的调用注册过的插件，正如其名字，它是瀑布式的调用，后一个插件可以通过第一个参数获得前一个插件的返回值（第一个获得 callAsync 时设置的初始值），执行结束可以 Hook 的回调中获得最后一个插件的返回值。

### Tapable

内部有个 SyncBailHook 类型的 Hook，此类是为了兼容之前版本的写法。

# Conclusion

对 webpack 的基础库 Tapable 了解后，我们就可以仔细分析 webpack 源码了。
