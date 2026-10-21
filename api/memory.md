# 内存 (Memory)

memory 模块用于在脚本中读取 AutoJs6 进程自身的内存诊断数据, 包括 Java 堆与原生堆用量, 存活的脚本引擎, 已结束但尚未被回收的脚本运行时, 未释放的图像与脚本线程等, 以便检查脚本反复运行后内存是否持续增长.

数据来源与 AutoJs6 "设置 > 关于 > 开发者选项 > 内存诊断" 页面相同. 此模块只读取诊断数据, 不提供释放其他脚本资源的能力.

`memory` 与 `$memory` 指向同一个模块对象.

> 注: 读取设备整体的内存信息 (总内存, 可用内存) 请使用 [device.getTotalMem](device#m-gettotalmem) 与 [device.getAvailMem](device#m-getavailmem).

---

<p style="font: bold 2em sans-serif; color: #FF7043">memory</p>

---

## [m] snapshot

### snapshot()

**`6.8.0`**

- <ins>**returns**</ins> {{
    - javaHeapUsed: [number](dataTypes#number);
    - javaHeapTotal: [number](dataTypes#number);
    - javaHeapMax: [number](dataTypes#number);
    - nativeHeapAllocated: [number](dataTypes#number);
    - nativeHeapSize: [number](dataTypes#number);
    - pss: [number](dataTypes#number);
    - engines: [number](dataTypes#number);
    - retainedRuntimes: [number](dataTypes#number);
    - images: [number](dataTypes#number);
    - threads: [number](dataTypes#number);
    - allThreads: [number](dataTypes#number);
    - contexts: [number](dataTypes#number);
    - timestamp: [number](dataTypes#number);
- }} - 内存快照

读取当前的内存数据. 此方法不会先执行垃圾回收, 因此 Java 堆用量包含尚未回收的垃圾; 需要回收后的数据时使用 [collect](#m-collect).

快照对象的各属性均为数值, 字节数以字节 (B) 为单位:

| 属性 | 含义 |
| --- | --- |
| javaHeapUsed | 已用 Java 堆, 即 `Runtime.totalMemory() - Runtime.freeMemory()` |
| javaHeapTotal | 虚拟机当前已分配的 Java 堆, 即 `Runtime.totalMemory()` |
| javaHeapMax | Java 堆上限, 即 `Runtime.maxMemory()` |
| nativeHeapAllocated | 原生堆已分配的字节数, 即 `Debug.getNativeHeapAllocatedSize()` |
| nativeHeapSize | 原生堆总大小, 即 `Debug.getNativeHeapSize()` |
| pss | AutoJs6 进程的 PSS (按比例计算的实际占用内存) |
| engines | 存活的脚本引擎数, 包括当前脚本 |
| retainedRuntimes | 已结束但尚未被回收的脚本运行时数, 与 [retained](#m-retained) 返回的数组长度相同 |
| images | 既未调用 `recycle` 也未被垃圾回收的脚本图像数 |
| threads | 为脚本创建的线程数, 按线程名判断 (脚本线程, `threads.start` 与 `threads.pool` 的线程, 定时器线程, 截屏线程与 Looper 线程) |
| allThreads | AutoJs6 进程内全部存活的线程数 |
| contexts | 已进入且尚未释放的 Rhino Context 数, 无法统计时为 `-1` |
| timestamp | 读数时刻, 自 1970-01-01 00:00:00 UTC 起的毫秒数 |

读取 PSS 需要遍历进程的内存映射, 一次调用通常需要几毫秒.

```js
let s = memory.snapshot();
console.log('Java 堆: ' + fmt.bytes(s.javaHeapUsed) + ' / ' + fmt.bytes(s.javaHeapMax));
console.log('原生堆: ' + fmt.bytes(s.nativeHeapAllocated));
console.log('存活引擎: ' + s.engines + ', 未回收的运行时: ' + s.retainedRuntimes);
```

## [m] collect

### collect(timeout?)

**`6.8.0`**

- **[ timeout = `10000` ]** { [number](dataTypes#number) } - 等待已结束的脚本运行时被回收的最长时间 (毫秒), 取值范围为 `0` 到 `60000`
- <ins>**returns**</ins> { [Object](dataTypes#object) } - 回收后的内存快照, 属性与 [snapshot](#m-snapshot) 的返回值相同

强制执行垃圾回收, 并等待已结束的脚本运行时被回收, 然后返回新的内存快照.

此方法先执行 3 次垃圾回收与 finalization, 之后每 100 毫秒检查一次, 每 500 毫秒再执行一次垃圾回收. 已结束的脚本运行时全部被回收时立即返回, 否则在 `timeout` 毫秒后返回. 返回的快照中 `retainedRuntimes` 不为 `0` 时, 说明仍有对象持有已结束脚本的运行时, 可用 [retained](#m-retained) 查看是哪些脚本.

此方法会阻塞当前线程, 最长约 `timeout` 毫秒, 不要在 UI 线程中调用. 脚本被停止时, 等待立即中止.

`timeout` 不是 `0` 到 `60000` 之间的数值时抛出异常.

```js
let before = memory.snapshot();
let after = memory.collect();
console.log('回收前 Java 堆: ' + fmt.bytes(before.javaHeapUsed));
console.log('回收后 Java 堆: ' + fmt.bytes(after.javaHeapUsed));
console.log('仍未回收的运行时: ' + after.retainedRuntimes);
```

## [m] retained

### retained()

**`6.8.0`**

- <ins>**returns**</ins> { [Object](dataTypes#object)[] } - 已结束但尚未被回收的脚本运行时, 先结束的在前

列出已结束但仍未被回收的脚本运行时.

数组元素的属性如下:

| 属性 | 类型 | 含义 |
| --- | --- | --- |
| id | [number](dataTypes#number) | 脚本引擎的 ID |
| name | [string](dataTypes#string) | 脚本名称 |
| source | [string](dataTypes#string) | 脚本开始运行时控制台显示的脚本路径 |
| endedAt | [number](dataTypes#number) | 脚本结束的时刻, 自 1970-01-01 00:00:00 UTC 起的毫秒数 |
| elapsedMs | [number](dataTypes#number) | 脚本结束后经过的毫秒数 |
| exitKind | [string](dataTypes#string) | 结束方式: `normal` 为自行结束, `error` 为因未捕获的错误结束, `stopped` 为被停止 (在任务列表中停止, `engines.stopAll` 等强制停止, 或脚本自身调用 `exit`) |

脚本刚结束时, 其运行时在下一次垃圾回收之前也会出现在此列表中. 先调用 [collect](#m-collect), 之后仍在列表中的脚本运行时才是被其他对象持有的, 通常意味着内存泄漏.

此列表只包含 JavaScript (Rhino) 脚本.

```js
memory.collect();
memory.retained().forEach((r) => {
    console.log(r.name + ' (' + r.exitKind + '), 已结束 ' + Math.round(r.elapsedMs / 1000) + ' 秒');
});
```

## [m] dumpHeap

### dumpHeap(path?)

**`6.8.0`**

- **[ path ]** { [string](dataTypes#string) } - hprof 文件的路径或其所在目录
- <ins>**returns**</ins> { [string](dataTypes#string) } - 写出的 hprof 文件的绝对路径

把 AutoJs6 进程的 Java 堆转储为 hprof 文件. 该文件可用 Android Studio 打开, 查看对象数量, 大小与引用关系.

`path` 的相对路径与 [files.path](files#m-path) 一样相对于脚本的工作目录解析. `path` 为已存在的目录或以 `/` 结尾时, 文件写入该目录; 省略 `path` 时写入工作目录. 这两种情况下的文件名为 `AutoJs6-heap-<yyyyMMdd-HHmmss>.hprof`. 同名文件会被覆盖. `path` 为空字符串或不是字符串时抛出异常.

转储期间 AutoJs6 进程会暂停, 堆越大耗时越长 (例如 50 MB 的堆约需 2 秒).

hprof 文件包含堆中的全部数据, 包括所有正在运行的脚本的变量, 以及其中的账户, 令牌等敏感内容, 请谨慎保存与分享. 因此:

- AutoJs6 的 debug 版本中, 此方法始终可用.
- AutoJs6 的正式 (release) 版本中, 需要先在 "开发者选项 > 内存诊断" 中开启 "允许脚本导出 hprof", 否则抛出异常. 此开关默认关闭.

```js
let file = memory.dumpHeap('./heap/before-cleanup.hprof');
console.log('已写出: ' + file);
```

## [m] watch

### watch(obj, name?)

**`6.8.0`**

- **obj** { [Object](dataTypes#object) } - 预期很快会被回收的对象
- **[ name ]** { [string](dataTypes#string) } - 说明文字, 显示在 LeakCanary 的报告中
- <ins>**returns**</ins> { [boolean](dataTypes#boolean) } - LeakCanary 是否开始监视该对象

把对象交给 [LeakCanary](https://square.github.io/leakcanary/) 监视. 若该对象在一段时间后 (LeakCanary 默认为 5 秒) 仍未被回收, LeakCanary 会将其计为残留对象并通过通知提示, 分析堆转储后可在其 "Leaks" 应用中查看引用链.

此方法仅在 AutoJs6 的 debug 版本中, 且 "开发者选项 > 内存诊断 > LeakCanary" 开启时有效, 此时返回 `true`. 其他情况下此方法不做任何事, 并返回 `false`.

`obj` 为 Java 对象时监视该 Java 对象本身, 而不是其在脚本中的包装. `obj` 为 `null`, `undefined`, 字符串, 数字或布尔值等原始值时抛出异常. 省略 `name` 时, 说明文字为 `memory.watch(<类名>) in <脚本名称>`.

```js
let cache = { items: [] };
memory.watch(cache, '临时缓存');
cache = null;
```
