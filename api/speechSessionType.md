# 听写会话 (SpeechSession)

SpeechSession 是 [speech.listen](speech#m-listen) 返回的听写会话对象, 也是 [SpeechWakeSession](speechWakeSessionType) 的 `wake` 事件中 `listen` 字段携带的听写轮次. 它是 [EventEmitter](eventEmitterType) 的实例, 连接脚本与 3-Macaw Speech 插件中正在运行的识别: 识别结果与状态变化以事件形式在打开会话的脚本线程到达, 脚本可以提前结束会话, 或为外部 PCM 会话输入音频.

会话打开期间脚本保持运行, 不需要也不应使用 `sleep` 循环等待; `end` 是唯一的终结事件, 之后不再有事件, 脚本没有其它工作时随即结束. 脚本退出时未结束的会话被静默取消 (不触发事件), 插件随即释放麦克风. 插件同一时刻只有一个麦克风会话, 见 [麦克风会话与后台收音](speech#麦克风会话与后台收音).

`on` / `once` 返回自身以便链式调用.

```js
let session = speech.listen({ maxDurationMillis: 20000 });
session.on('ready', e => console.log('模型:', e.modelId))
    .on('partial', e => console.log('...', e.text))
    .on('final', e => console.log(e.startMillis, e.endMillis, e.text))
    .on('end', e => console.log('结束:', e.reason, e.text));
```

---

<p style="font: bold 2em sans-serif; color: #FF7043">SpeechSession</p>

---

## [@] SpeechSession

**`6.8.0`**

会话对象. 不能用 `new` 构造, 只由 [speech.listen](speech#m-listen) 产生, 或作为唤醒会话 `wake` 事件的 `listen` 字段出现.

## [p#] id

**`6.8.0`**

- { [string](dataTypes#string) | [null](dataTypes#null) }

插件分配的会话 ID, 来自 `ready` 事件, 之前为 `null`.

## [p#] modelId

**`6.8.0`**

- { [string](dataTypes#string) | [null](dataTypes#null) }

服务本会话的识别模型 ID, 来自 `ready` 事件, 之前为 `null`.

## [p#] source

**`6.8.0`**

- { `'mic'` | `'external'` }

音频来源. 唤醒会话的听写轮次总是 `'mic'`.

## [p#] round

**`6.8.0`**

- { [number](dataTypes#number) }

唤醒会话中的听写轮次序号 (从 `0` 开始); 由 [speech.listen](speech#m-listen) 打开的会话为 `-1`.

## [p#] startedAt

**`6.8.0`**

- { [number](dataTypes#number) }

请求打开会话时的时间戳 (自 1970-01-01 UTC 起的毫秒数).

## [p#] state

**`6.8.0`**

- { `'pending'` | `'active'` | `'ended'` }

`ready` 事件之前为 `'pending'` (模型加载中), 之后为 `'active'`, 结束后为 `'ended'`.

## [p#] reason

**`6.8.0`**

- { [string](dataTypes#string) | [null](dataTypes#null) }

结束后为 `end` 事件的原因, 之前为 `null`; 脚本退出时的静默取消为 `'script-exit'`.

## [p#] isEnded

**`6.8.0`**

- { [boolean](dataTypes#boolean) }

会话是否已结束 (含脚本退出时的静默取消).

## [m#] isActive

### session.isActive()

**`6.8.0`**

- <ins>**returns**</ins> { [boolean](dataTypes#boolean) } - 会话是否尚未结束 (`'pending'` 或 `'active'`)

## [m#] result

### session.result()

**`6.8.0`**

- <ins>**returns**</ins> { [string](dataTypes#string) } - 目前已确定的文字

结束前返回已收到的 `final` 文字的拼接, 结束后返回插件给出的整段文字. 拼接时各段直接相连, 只在两个字母或数字会相连处插入一个空格, 因此中文不含多余空格, 英文单词保持分隔.

## [m#] stop

### session.stop()

**`6.8.0`**

- <ins>**returns**</ins> { [SpeechSession](#speechsession) } - 自身

冲刷识别器并结束会话: 尚有未确定的文字时先触发最后一个 `final`, 然后以原因 `'stopped'` 触发 `end`. 重复调用或结束后调用无副作用. 对唤醒会话的听写轮次调用时只结束该轮, 唤醒会话随后恢复关键词检测.

## [m#] cancel

### session.cancel()

**`6.8.0`**

- <ins>**returns**</ins> { [SpeechSession](#speechsession) } - 自身

不产生最终结果地立即结束会话, `end` 的原因为 `'cancelled'`. 重复调用或结束后调用无副作用. 对唤醒会话的听写轮次调用时只结束该轮并丢弃其文字.

## [m#] feed

### session.feed(pcm)

**`6.8.0`**

- **pcm** { [ArrayBuffer](dataTypes#arraybuffer) | [TypedArray](dataTypes#typedarray) | [ByteArray](dataTypes#bytearray) } - 16 kHz 单声道 PCM16 LE 音频, 每次最多 1 MiB
- <ins>**returns**</ins> { [SpeechSession](#speechsession) } - 自身

向以 `source: 'external'` 打开的会话输入音频. 大端字节序的 `Int16Array` 会被转换为小端; 不接受普通 JavaScript 数字数组.

- 麦克风会话 (包括唤醒会话的听写轮次) 调用时抛出 `INVALID_OPTIONS`.
- 会话已结束时抛出 `SESSION_CLOSED`.
- 超过 1 MiB 时抛出 `LIMIT_EXCEEDED`; 字节数为奇数时抛出 `INVALID_OPTIONS`; 空数据被忽略.

外部会话的 `timeoutMillis` 与 `maxDurationMillis` 按已输入的音频时长计算, 暂停输入不会导致超时.

```js
let session = speech.listen({ source: 'external' });
session.on('ready', () => {
    session.feed(files.readBytes('./clip.pcm'));
    session.stop();
});
session.on('end', () => console.log(session.result()));
```

## [m#] on

### session.on(event, listener)

**`6.8.0`**

- **event** { [string](dataTypes#string) } - 事件名
- **listener** { [Function](dataTypes#function) } - 监听器
- <ins>**returns**</ins> { [SpeechSession](#speechsession) } - 自身

注册监听器. `once` 形态相同, 只触发一次. 其余成员见 [EventEmitter](eventEmitterType).

## [m#] toString

### session.toString()

**`6.8.0`**

- <ins>**returns**</ins> { [string](dataTypes#string) } - 形如 `SpeechSession { source: mic, state: active, modelId: asr.sense-voice.zh-en-ja-ko-yue.2024-07-17.int8 }`

## 事件 (Events)

| 事件 | 回调参数 | 说明 |
| --- | --- | --- |
| `ready` | `({ sessionId, modelId })` | 模型已加载, 开始收音或接受输入 |
| `partial` | `({ text, segmentIndex })` | 当前一句话到目前为止的文字; 流式模型逐字更新, 离线模型按语音段更新 |
| `final` | `({ text, segmentIndex, startMillis, endMillis, language?, tags? })` | 一句话的最终文字; 时间相对会话的音频开头, 单位为毫秒 |
| `endpoint` | `()` | 一句话结束, 紧随其 `final` 之后; 流式与离线模型都会触发 |
| `vad` | `({ active })` | 语音活动状态变化, `active` 为 `true` 表示正在说话 |
| `warning` | `({ code, message })` | 非致命问题, 会话继续 |
| `error` | `(err: SpeechError)` | 会话失败, 随后以原因 `'error'` 触发 `end` |
| `end` | `({ reason, text })` | 会话已结束, 不再有事件; `text` 为整段文字 |

- `final` 的 `language` 为识别出的语种 (SenseVoice), `tags` 可能含 `emotion` 与 `event` (SenseVoice 的情感与声音事件标签).
- `warning` 的常见代码: `PERMISSION_DENIED` (收到的是静音, 或麦克风前台服务被拒绝, 见 [麦克风会话与后台收音](speech#麦克风会话与后台收音)), `INVALID_OPTIONS` (模型不支持热词, 或不认识语言提示).
- `end` 的原因: `'stopped'` (调用 `stop`), `'timeout'` (达到 `timeoutMillis`), `'maxDuration'` (达到 `maxDurationMillis`), `'cancelled'` (调用 `cancel`), `'error'` (插件侧失败或插件进程退出, 此时先触发 `code` 为 `SESSION_CLOSED` 等的 `error`), `'overflow'` (脚本积压的事件超过 512 条或 8 MiB, 先触发 `code` 为 `LIMIT_EXCEEDED` 的 `error`, 积压的事件被丢弃, 插件会话随之取消).
- 唤醒会话结束时, 仍在进行的听写轮次先以相同原因触发自己的 `end`.

事件对象都是普通 JavaScript 对象, `error` 事件的参数为 [SpeechError](speech#c-speecherror). 建议始终注册 `end` 与 `error` 监听器.
