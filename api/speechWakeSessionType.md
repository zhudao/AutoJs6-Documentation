# 唤醒会话 (SpeechWakeSession)

SpeechWakeSession 是 [speech.wake](speech#m-wake) 返回的唤醒词会话对象, 是 [EventEmitter](eventEmitterType) 的实例: 3-Macaw Speech 插件从麦克风持续检测关键词, 每次命中以 `wake` 事件在打开会话的脚本线程到达. 设置 `thenListen` 后, 每次命中自动转入一轮听写, 该轮的 [SpeechSession](speechSessionType) 由 `wake` 事件的 `listen` 字段交给脚本, 该轮结束后恢复关键词检测.

会话打开期间脚本保持运行, 直至 `end` 事件; `end` 是唯一的终结事件, 各听写轮次的 `end` 只是普通事件. 脚本退出时未结束的会话被静默取消, 插件随即释放麦克风. 唤醒会话占用插件唯一的麦克风; 插件自己的听写入口 (数字助理, 系统语音输入, 系统识别服务, 试听试说) 收音期间, 唤醒会话保持打开但静默, 见 [唤醒会话的临时接管](speech#唤醒会话的临时接管).

`on` / `once` 返回自身以便链式调用.

```js
let wake = speech.wake(['小六小六', 'hey macaw'], { thenListen: true });
wake.on('wake', e => {
    console.log('唤醒:', e.keyword);
    e.listen.on('final', r => console.log('指令:', r.text));
}).on('end', e => console.log('唤醒会话结束:', e.reason));
```

---

<p style="font: bold 2em sans-serif; color: #FF7043">SpeechWakeSession</p>

---

## [@] SpeechWakeSession

**`6.8.0`**

会话对象. 不能用 `new` 构造, 只由 [speech.wake](speech#m-wake) 产生.

## [p#] keywords

**`6.8.0`**

- { [string](dataTypes#string)[[]](dataTypes#array) }

脚本所写的关键词原文 (对象形式取其 `keyword`), 顺序与调用时一致.

## [p#] id

**`6.8.0`**

- { [string](dataTypes#string) | [null](dataTypes#null) }

插件分配的会话 ID, 来自 `ready` 事件, 之前为 `null`.

## [p#] modelId

**`6.8.0`**

- { [string](dataTypes#string) | [null](dataTypes#null) }

唤醒词模型 ID, 来自 `ready` 事件, 之前为 `null`.

## [p#] startedAt

**`6.8.0`**

- { [number](dataTypes#number) }

请求打开会话时的时间戳 (自 1970-01-01 UTC 起的毫秒数).

## [p#] state

**`6.8.0`**

- { `'pending'` | `'active'` | `'ended'` }

`ready` 事件之前为 `'pending'`, 之后为 `'active'`, 结束后为 `'ended'`. 被临时接管期间仍为 `'active'`.

## [p#] reason

**`6.8.0`**

- { [string](dataTypes#string) | [null](dataTypes#null) }

结束后为 `end` 事件的原因, 之前为 `null`; 脚本退出时的静默取消为 `'script-exit'`.

## [p#] isEnded

**`6.8.0`**

- { [boolean](dataTypes#boolean) }

会话是否已结束 (含脚本退出时的静默取消).

## [m#] isActive

### wakeSession.isActive()

**`6.8.0`**

- <ins>**returns**</ins> { [boolean](dataTypes#boolean) } - 会话是否尚未结束

## [m#] stop

### wakeSession.stop()

**`6.8.0`**

- <ins>**returns**</ins> { [SpeechWakeSession](#speechwakesession) } - 自身

结束唤醒会话及正在进行的听写轮次, `end` 的原因为 `'stopped'`. 重复调用或结束后调用无副作用. 只想结束当前听写轮次而继续检测关键词时, 对该轮的 [SpeechSession](speechSessionType) 调用 `stop` 或 `cancel`.

唤醒会话没有听写会话的 `cancel`, `feed` 与 `result`.

## [m#] on

### wakeSession.on(event, listener)

**`6.8.0`**

- **event** { [string](dataTypes#string) } - 事件名
- **listener** { [Function](dataTypes#function) } - 监听器
- <ins>**returns**</ins> { [SpeechWakeSession](#speechwakesession) } - 自身

注册监听器. `once` 形态相同, 只触发一次. 其余成员见 [EventEmitter](eventEmitterType).

## [m#] toString

### wakeSession.toString()

**`6.8.0`**

- <ins>**returns**</ins> { [string](dataTypes#string) } - 形如 `SpeechWakeSession { keywords: [小六小六], state: active }`

## 事件 (Events)

| 事件 | 回调参数 | 说明 |
| --- | --- | --- |
| `ready` | `({ sessionId, modelId })` | 模型已加载, 开始检测 |
| `wake` | `({ keyword, score, timestampMillis, listen? })` | 检测到关键词 |
| `warning` | `({ code, message })` | 非致命问题, 会话继续 |
| `error` | `(err: SpeechError)` | 会话失败, 随后以原因 `'error'` 触发 `end` |
| `end` | `({ reason })` | 会话已结束, 不再有事件 |

`wake` 事件的字段:

- **keyword** { [string](dataTypes#string) } - 命中的关键词, 与脚本所写的原文一致
- **score** { [number](dataTypes#number) } - 3-Macaw Speech 使用的 sherpa-onnx 1.13.8 不报告命中置信度, 此值为该关键词的触发阈值 (其 `threshold`, 未指定时为模型默认值)
- **timestampMillis** { [number](dataTypes#number) } - 插件报告命中时的时间戳 (自 1970-01-01 UTC 起的毫秒数)
- **[ listen ]** { [SpeechSession](speechSessionType) } - 设置 `thenListen` 时为本次命中后的听写轮次, 其 `round` 从 `0` 递增

使用 `thenListen` 时, 应在 `wake` 监听器中为 `listen` 注册事件: 该轮的事件在 `wake` 监听器返回后才开始投递, 因此不会错过 `ready` 等早期事件. 听写轮次进行期间不检测关键词, 也不会被插件自己的听写入口临时接管.

- `warning` 的常见代码为 `PERMISSION_DENIED`, 含义同 [SpeechSession](speechSessionType#事件-events).
- `end` 的原因: `'stopped'`, `'error'` (插件侧失败或插件进程退出), `'overflow'` (脚本积压的事件超过 512 条或 8 MiB, 先触发 `code` 为 `LIMIT_EXCEEDED` 的 `error`).
- 被临时接管期间不触发任何事件; 接管结束后自动恢复检测, 不会补发接管期间的命中.

事件对象都是普通 JavaScript 对象, `error` 事件的参数为 [SpeechError](speech#c-speecherror).
